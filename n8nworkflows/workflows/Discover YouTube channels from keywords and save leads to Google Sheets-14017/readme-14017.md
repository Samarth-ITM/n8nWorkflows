Discover YouTube channels from keywords and save leads to Google Sheets

https://n8nworkflows.xyz/workflows/discover-youtube-channels-from-keywords-and-save-leads-to-google-sheets-14017


# Discover YouTube channels from keywords and save leads to Google Sheets

# 1. Workflow Overview

This workflow discovers YouTube channels from a queue of target keywords, evaluates channel quality using YouTube statistics, and stores qualified channels in Google Sheets. It is designed for lead generation, influencer discovery, and niche creator research.

The automation runs on a schedule, processes one pending keyword per execution, searches YouTube videos for that keyword, extracts unique channel IDs from the video results, fetches detailed channel metadata, applies qualification thresholds, and then appends or updates the resulting leads in a spreadsheet.

There is also an alternative Apify-based branch present in the workflow, but it is disabled and not part of the active execution path.

## 1.1 Scheduled Start

The workflow begins automatically on a cron schedule. Each run is intended to pick up the next available keyword.

## 1.2 Keyword Retrieval and Status Update

The workflow reads keywords from a Google Sheet, filters for rows with `Status = Pending`, selects only the first pending keyword, and then updates that keyword row to mark it as processed while also storing the last execution timestamp.

## 1.3 YouTube Discovery

Using the selected keyword, the workflow calls the YouTube Search API to retrieve up to 50 matching videos.

## 1.4 Channel Extraction and Qualification

The workflow extracts unique `channelId` values from the video search response, requests channel details from the YouTube Channels API, and filters channels according to minimum thresholds for subscribers, videos, and views. It also attempts to extract email and social links from channel metadata.

## 1.5 Lead Persistence

Qualified channels are saved into a second Google Sheet using an append-or-update strategy keyed by Channel ID, preventing duplicates while keeping records refreshed.

## 1.6 Disabled Alternative Branch

A separate disabled branch uses an Apify actor to scrape YouTube channel leads and then normalize/store them into another sheet. This branch is currently inactive but still part of the workflow definition and should be documented for maintenance.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduling and Workflow Entry

### Overview
This block provides the workflow’s only active entry point. It triggers execution automatically based on a cron expression.

### Nodes Involved
- Schedule Trigger

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry node that launches the workflow on a recurring schedule.
- **Configuration choices:** Uses a cron expression of `* * * * *`, meaning the workflow runs every minute.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - **Input:** None, as this is a trigger node.
  - **Output:** `List of Target Keywords`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Overlapping executions may occur if runs take longer than one minute.
  - Frequent scheduling can consume API quota quickly, especially with YouTube API usage.
  - If the workflow is inactive, this node does nothing.
- **Sub-workflow reference:** None.

---

## Block 2 — Keyword Queue Retrieval and State Management

### Overview
This block loads pending keywords from Google Sheets, selects a single keyword for this run, and updates its status so it is not processed again in future runs. It effectively acts as a simple queue processor.

### Nodes Involved
- List of Target Keywords
- Get the first keyword
- Update Keyword Status -> Processing

### Node Details

#### List of Target Keywords
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from the keyword tracking sheet.
- **Configuration choices:**
  - Uses Google Sheets OAuth2 credentials.
  - Reads from spreadsheet `Test Sheet`.
  - Targets sheet `Sheet1` (`gid=0`).
  - Applies a filter where `Status = Pending`.
- **Key expressions or variables used:** None directly in parameters beyond the filter value.
- **Input and output connections:**
  - **Input:** `Schedule Trigger`
  - **Output:** `Get the first keyword`
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - OAuth token expiration or revoked Google access.
  - Missing `Status` column or renamed sheet.
  - Empty result set when no pending keywords exist.
  - Spreadsheet structure drift can break mappings downstream.
- **Sub-workflow reference:** None.

#### Get the first keyword
- **Type and technical role:** `n8n-nodes-base.code`; reduces all pending keyword rows to only the first item.
- **Configuration choices:**
  - JavaScript code uses `$input.all()`.
  - Returns `[]` if no pending items exist.
  - Returns `[items[0]]` to process exactly one keyword per run.
- **Key expressions or variables used:**
  - `$input.all()`
- **Input and output connections:**
  - **Input:** `List of Target Keywords`
  - **Output:** `Update Keyword Status -> Processing`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If the upstream node returns no items, the workflow stops cleanly here.
  - Assumes the first item is the intended priority; there is no explicit sorting.
  - If row data does not contain `Keyword`, the next node’s update logic will fail or write incomplete data.
- **Sub-workflow reference:** None.

#### Update Keyword Status -> Processing
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates or inserts the keyword row in the queue sheet.
- **Configuration choices:**
  - Uses append-or-update mode.
  - Matches rows by `Keyword`.
  - Writes:
    - `Keyword = {{$json.Keyword}}`
    - `Status = Processed`
    - `Last Run = {{$now}}`
  - Uses the same spreadsheet and `Sheet1`.
- **Key expressions or variables used:**
  - `={{ $json.Keyword }}`
  - `={{ $now }}`
- **Input and output connections:**
  - **Input:** `Get the first keyword`
  - **Output:** `Search Videos based on Keywords`
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - The node name says “Processing” but the actual value written is `Processed`. This is an important naming/logic mismatch.
  - Updating the row before downstream success means failed runs can still mark a keyword as done.
  - If multiple runs execute concurrently, the same keyword might still be selected by different runs before the sheet is updated.
  - Matching by `Keyword` assumes uniqueness; duplicate keyword rows may cause unexpected updates.
- **Sub-workflow reference:** None.

---

## Block 3 — YouTube Video Search

### Overview
This block queries the YouTube Search API using the selected keyword. The goal is not to keep the videos themselves, but to discover channels behind those videos.

### Nodes Involved
- Search Videos based on Keywords

### Node Details

#### Search Videos based on Keywords
- **Type and technical role:** `n8n-nodes-base.httpRequest`; performs a REST API request against the YouTube Search endpoint.
- **Configuration choices:**
  - URL: `https://www.googleapis.com/youtube/v3/search`
  - Authentication: predefined credential type using YouTube OAuth2
  - Query parameters:
    - `part=snippet`
    - `q={{ $json.Keyword }}`
    - `type=video`
    - `maxResults=50`
- **Key expressions or variables used:**
  - `={{ $json.Keyword }}`
- **Input and output connections:**
  - **Input:** `Update Keyword Status -> Processing`
  - **Output:** `Extract Channel Ids from the Video Results`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Invalid or expired YouTube OAuth2 credentials.
  - YouTube quota exhaustion.
  - No results for a keyword.
  - API response shape changes or localization differences.
  - Network errors, rate limits, or HTTP 4xx/5xx responses.
- **Sub-workflow reference:** None.

---

## Block 4 — Channel ID Extraction and YouTube Channel Data Retrieval

### Overview
This block converts the video search results into a deduplicated list of channel IDs, then uses the YouTube Channels API to fetch detailed channel statistics and metadata.

### Nodes Involved
- Extract Channel Ids from the Video Results
- Get Channel Stats

### Node Details

#### Extract Channel Ids from the Video Results
- **Type and technical role:** `n8n-nodes-base.code`; parses the search response and builds a single comma-separated string of unique channel IDs.
- **Configuration choices:**
  - Reads `$json.items || []`
  - Extracts `item.snippet?.channelId`
  - Removes undefined values
  - Deduplicates IDs using `Set`
  - Outputs one item with `channelIdsString`
- **Key expressions or variables used:**
  - `$json.items`
  - Optional chaining on `item.snippet?.channelId`
- **Input and output connections:**
  - **Input:** `Search Videos based on Keywords`
  - **Output:** `Get Channel Stats`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If the YouTube search response has no `items`, this node returns an empty `channelIdsString`.
  - If all IDs are missing, the next API call may be malformed or return no data.
  - Assumes channel ID is in `snippet.channelId`, which is valid for standard search results but still depends on API consistency.
- **Sub-workflow reference:** None.

#### Get Channel Stats
- **Type and technical role:** `n8n-nodes-base.httpRequest`; fetches channel statistics and profile metadata from the YouTube Channels API.
- **Configuration choices:**
  - URL: `https://www.googleapis.com/youtube/v3/channels`
  - Authentication: predefined YouTube OAuth2 credential
  - Query parameters:
    - `part=statistics, snippet, brandingSettings`
    - `id = =id={{ $json.channelIdsString }}`
- **Key expressions or variables used:**
  - `{{ $json.channelIdsString }}`
- **Input and output connections:**
  - **Input:** `Extract Channel Ids from the Video Results`
  - **Output:** `Filter based on Criteria`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - There appears to be a configuration anomaly in the `id` parameter: `=id={{ $json.channelIdsString }}`. Normally this should likely be just `={{ $json.channelIdsString }}`. As configured, it may send an invalid value.
  - If `channelIdsString` is empty, the request may fail or return an empty response.
  - YouTube quota and auth issues apply here as well.
  - `brandingSettings` may not contain all expected fields.
- **Sub-workflow reference:** None.

---

## Block 5 — Qualification and Lead Normalization

### Overview
This block filters channels by quality thresholds and transforms the YouTube channel payload into a lead record format suitable for storage. It also attempts to extract contact and social information.

### Nodes Involved
- Filter based on Criteria

### Node Details

#### Filter based on Criteria
- **Type and technical role:** `n8n-nodes-base.code`; applies business rules and maps API fields into a normalized lead object.
- **Configuration choices:**
  - Reads `const channels = $json.items || []`
  - Thresholds:
    - `subscriberThreshold = 10000`
    - `videoThreshold = 100`
    - `viewThreshold = 500000`
  - Extracts emails from `brandingSettings.channel.description` using regex.
  - Attempts to merge with `channel.emails || []`
  - Attempts to read social links from `channel.brandingSettings?.channel?.externalLinks || {}`
  - Returns one item per qualified channel.
  - If no channels pass, returns one item `{ retry: true }`.
- **Key expressions or variables used:**
  - `channel.statistics?.subscriberCount`
  - `channel.statistics?.videoCount`
  - `channel.statistics?.viewCount`
  - `channel.brandingSettings?.channel?.description`
  - email regex
- **Input and output connections:**
  - **Input:** `Get Channel Stats`
  - **Output:** `Append or update row in sheet`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - The `retry: true` fallback does not actually control workflow retry behavior; it simply creates an item that will continue downstream.
  - If no channels qualify, the downstream Google Sheets node may insert or update a mostly empty row because it receives an item lacking channel fields.
  - `externalLinks` is not guaranteed in the YouTube Channels API response as used here; socials may remain blank.
  - `channel.emails` is not a standard YouTube field, so that merge is mostly ineffective unless another upstream system adds it.
  - Parsing counts with `parseInt` is safe for integer strings, but missing or malformed values default to zero.
- **Sub-workflow reference:** None.

---

## Block 6 — Final Storage in Google Sheets

### Overview
This block persists qualified channels to a destination sheet. It prevents duplicate entries by matching on Channel ID.

### Nodes Involved
- Append or update row in sheet

### Node Details

#### Append or update row in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; writes final lead rows to Google Sheets.
- **Configuration choices:**
  - Spreadsheet: `Test Sheet`
  - Sheet: `Sheet2`
  - Operation: `appendOrUpdate`
  - Matching column: `Channel ID`
  - Writes mapped fields:
    - Channel ID
    - Channel Name
    - Subscribers
    - Total Videos
    - Total Views
    - Country
    - Email
    - Instagram
    - Twitter
    - Tiktok
    - Discord
    - Facebook
- **Key expressions or variables used:**
  - `={{ $json.channelId }}`
  - `={{ $json.title }}`
  - `={{ $json.subscribers }}`
  - `={{ $json.videos }}`
  - `={{ $json.views }}`
  - `={{ $json.country }}`
  - `={{ $json.email }}`
  - social fields from `$json`
- **Input and output connections:**
  - **Input:** `Filter based on Criteria`
  - **Output:** None
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - If upstream sends `{ retry: true }` without channel data, this node may create an incomplete row depending on sheet behavior.
  - If the `Channel ID` column is missing or renamed, deduplication fails.
  - OAuth or permission issues can prevent writes.
  - Data typing is left as-is; no forced string conversion is enabled.
- **Sub-workflow reference:** None.

---

## Block 7 — Disabled Apify-Based Alternative Flow

### Overview
This block is disabled and not connected to the active path. It appears to be an alternative lead discovery method using an Apify actor to scrape YouTube channel and contact data directly from keywords.

### Nodes Involved
- Run an Actor and get dataset
- Code in JavaScript
- Append or update row in sheet1

### Node Details

#### Run an Actor and get dataset
- **Type and technical role:** `@apify/n8n-nodes-apify.apify`; executes an Apify actor and returns its dataset.
- **Configuration choices:**
  - Operation: Run actor and get dataset
  - Actor: `Youtube Channel Email Scraper By Keyword`
  - Custom input body includes:
    - `country: "United States"`
    - `keywords: ["{{ $json.Keyword }}"]`
    - `maxLeadsPerKeyword: 100`
    - `scrapeLeadsWithEmail: true`
  - Authentication: Apify OAuth2
- **Key expressions or variables used:**
  - `{{ $json.Keyword }}`
- **Input and output connections:**
  - **Input:** None connected in active workflow
  - **Output:** `Code in JavaScript`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Disabled node, so it will not run unless manually enabled and wired.
  - Requires Apify credentials and available actor access.
  - Actor runtime failures, scraping blocks, or dataset schema changes can break downstream parsing.
- **Sub-workflow reference:** None.

#### Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`; cleans Apify results, extracts contact/social fields, and filters by thresholds.
- **Configuration choices:**
  - Uses email regex against `emails`
  - Extracts `social_links`
  - Thresholds:
    - subscribers >= 10000
    - videos >= 100
  - Emits normalized item format for Google Sheets
- **Key expressions or variables used:**
  - `$input.all()`
  - `item.json.emails`
  - `item.json.social_links`
  - `parseInt(channel.subscribers || "0")`
  - `parseInt(channel.videos || "0")`
- **Input and output connections:**
  - **Input:** `Run an Actor and get dataset`
  - **Output:** `Append or update row in sheet1`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Disabled node.
  - Schema assumptions may fail if actor output changes.
  - `parseInt` on formatted strings may give incorrect results if values include commas or labels.
- **Sub-workflow reference:** None.

#### Append or update row in sheet1
- **Type and technical role:** `n8n-nodes-base.googleSheets`; writes Apify-derived leads into a dedicated sheet.
- **Configuration choices:**
  - Spreadsheet: `Test Sheet`
  - Sheet: `Using Apify`
  - Operation: `appendOrUpdate`
  - Matching column: `Channel URL`
  - Maps fields such as channel name, URL, keyword, email, socials, subscribers, and video count
  - Uses expressions like:
    - `subscriber_count.match(/\d+/)[0]`
    - `video_count.match(/\d+/)[0]`
- **Key expressions or variables used:**
  - Regex extraction from `subscriber_count`
  - Regex extraction from `video_count`
- **Input and output connections:**
  - **Input:** `Code in JavaScript`
  - **Output:** None
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Disabled node.
  - Regex `.match(/\d+/)[0]` will throw if no digit sequence exists.
  - Matching only the first digit sequence can truncate values like `12,345` into `12`.
  - OAuth and sheet schema issues apply here as well.
- **Sub-workflow reference:** None.

---

## Block 8 — Documentation Sticky Notes

### Overview
These nodes are non-executable annotations embedded in the canvas. They describe workflow intent, setup requirements, and each major step.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas annotation.
- **Configuration choices:** Provides a workflow-level overview describing discovery, filtering, and spreadsheet storage.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for scheduling step.
- **Configuration choices:** Explains periodic scheduling and one-keyword-per-run behavior.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for YouTube search step.
- **Configuration choices:** Explains using video search to discover channels.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for keyword retrieval and single-item processing.
- **Configuration choices:** Explains pending keyword retrieval and immediate processed marking.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for final spreadsheet save.
- **Configuration choices:** Explains append-or-update persistence of qualified channels.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for extraction, stats lookup, and filtering.
- **Configuration choices:** Describes deduplication and threshold filtering.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for setup prerequisites.
- **Configuration choices:** Lists credentials, storage, and threshold tuning requirements.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow on a cron schedule |  | List of Target Keywords | ## Step 1. Schedule workflow trigger<br><br>This workflow starts automatically at a scheduled interval.<br><br>The scheduler allows you to run the automation periodically, for example every 5 minutes, every hour, or once per day depending on your discovery needs.<br><br>Each run processes the next available keyword from the keyword list. |
| List of Target Keywords | n8n-nodes-base.googleSheets | Reads pending keywords from Google Sheets | Schedule Trigger | Get the first keyword | ## Step 2. Retrieve Target Keywords and Process One Keyword at a Time.<br><br>This step retrieves the list of target keywords from the data source. Each keyword represents a topic or niche that will be used to search for related YouTube videos and discover relevant channels.<br><br>Example keywords might include:<br>• dashcam<br>• productivity tools<br>• AI automation<br>• gaming highlights<br><br>To avoid duplicate processing, the workflow selects the first unprocessed keyword from the list.<br><br>Once selected, the keyword is immediately marked as processed to ensure it is not used again in future runs.<br><br>This ensures that the workflow processes each keyword only once. |
| Get the first keyword | n8n-nodes-base.code | Selects only the first pending keyword | List of Target Keywords | Update Keyword Status -> Processing | ## Step 2. Retrieve Target Keywords and Process One Keyword at a Time.<br><br>This step retrieves the list of target keywords from the data source. Each keyword represents a topic or niche that will be used to search for related YouTube videos and discover relevant channels.<br><br>Example keywords might include:<br>• dashcam<br>• productivity tools<br>• AI automation<br>• gaming highlights<br><br>To avoid duplicate processing, the workflow selects the first unprocessed keyword from the list.<br><br>Once selected, the keyword is immediately marked as processed to ensure it is not used again in future runs.<br><br>This ensures that the workflow processes each keyword only once. |
| Update Keyword Status -> Processing | n8n-nodes-base.googleSheets | Marks the selected keyword row as processed and stores last run time | Get the first keyword | Search Videos based on Keywords | ## Step 2. Retrieve Target Keywords and Process One Keyword at a Time.<br><br>This step retrieves the list of target keywords from the data source. Each keyword represents a topic or niche that will be used to search for related YouTube videos and discover relevant channels.<br><br>Example keywords might include:<br>• dashcam<br>• productivity tools<br>• AI automation<br>• gaming highlights<br><br>To avoid duplicate processing, the workflow selects the first unprocessed keyword from the list.<br><br>Once selected, the keyword is immediately marked as processed to ensure it is not used again in future runs.<br><br>This ensures that the workflow processes each keyword only once. |
| Search Videos based on Keywords | n8n-nodes-base.httpRequest | Calls the YouTube Search API to find videos for the keyword | Update Keyword Status -> Processing | Extract Channel Ids from the Video Results | ## Step 3. Search YouTube Videos<br><br>Using the selected keyword, the workflow searches YouTube videos through the YouTube API.<br><br>Videos are used as the discovery source because every video is linked to a channel.<br><br>By analyzing these results, the workflow can identify relevant creators within the niche. |
| Extract Channel Ids from the Video Results | n8n-nodes-base.code | Extracts and deduplicates channel IDs from YouTube search results | Search Videos based on Keywords | Get Channel Stats | ## Step 4. Extract Channel IDs, Retrieve Channel Stats, and Filter Qualified Channels<br>From the video results, the workflow extracts the unique channel IDs of the creators who published those videos.<br><br>Duplicate channel IDs are removed to avoid requesting the same channel statistics multiple times.<br><br>For each discovered channel, the workflow retrieves detailed statistics such as:<br>• Subscriber count<br>• Total number of videos<br>• Total channel views<br><br>These metrics are used to evaluate whether the channel meets the qualification criteria.<br><br>Channels are filtered based on predefined thresholds to identify high quality creators.<br><br>Example criteria:<br>• Minimum subscribers<br>• Minimum number of videos<br>• Minimum total views<br><br>Only channels that meet all requirements proceed to the next step. |
| Get Channel Stats | n8n-nodes-base.httpRequest | Calls the YouTube Channels API to retrieve channel metadata and statistics | Extract Channel Ids from the Video Results | Filter based on Criteria | ## Step 4. Extract Channel IDs, Retrieve Channel Stats, and Filter Qualified Channels<br>From the video results, the workflow extracts the unique channel IDs of the creators who published those videos.<br><br>Duplicate channel IDs are removed to avoid requesting the same channel statistics multiple times.<br><br>For each discovered channel, the workflow retrieves detailed statistics such as:<br>• Subscriber count<br>• Total number of videos<br>• Total channel views<br><br>These metrics are used to evaluate whether the channel meets the qualification criteria.<br><br>Channels are filtered based on predefined thresholds to identify high quality creators.<br><br>Example criteria:<br>• Minimum subscribers<br>• Minimum number of videos<br>• Minimum total views<br><br>Only channels that meet all requirements proceed to the next step. |
| Filter based on Criteria | n8n-nodes-base.code | Applies thresholds and normalizes channel lead data | Get Channel Stats | Append or update row in sheet | ## Step 4. Extract Channel IDs, Retrieve Channel Stats, and Filter Qualified Channels<br>From the video results, the workflow extracts the unique channel IDs of the creators who published those videos.<br><br>Duplicate channel IDs are removed to avoid requesting the same channel statistics multiple times.<br><br>For each discovered channel, the workflow retrieves detailed statistics such as:<br>• Subscriber count<br>• Total number of videos<br>• Total channel views<br><br>These metrics are used to evaluate whether the channel meets the qualification criteria.<br><br>Channels are filtered based on predefined thresholds to identify high quality creators.<br><br>Example criteria:<br>• Minimum subscribers<br>• Minimum number of videos<br>• Minimum total views<br><br>Only channels that meet all requirements proceed to the next step. |
| Append or update row in sheet | n8n-nodes-base.googleSheets | Saves or updates qualified channels in the destination sheet | Filter based on Criteria |  | ## Step 5. Save Results to Spreadsheet<br><br>Qualified channels are saved to the spreadsheet.<br><br>If the channel already exists, its information is updated. If it is new, a new row is added.<br><br>This creates a continuously growing database of qualified YouTube channels. |
| Run an Actor and get dataset | @apify/n8n-nodes-apify.apify | Disabled alternative method for scraping YouTube leads via Apify |  | Code in JavaScript |  |
| Code in JavaScript | n8n-nodes-base.code | Disabled Apify result cleaner and filter | Run an Actor and get dataset | Append or update row in sheet1 |  |
| Append or update row in sheet1 | n8n-nodes-base.googleSheets | Disabled Apify output writer | Code in JavaScript |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for workflow overview |  |  | ## Workflow overview<br><br>This workflow automatically discovers high quality YouTube channels based on a list of target keywords.<br><br>It runs on a schedule, searches for videos related to each keyword, extracts the channels behind those videos, retrieves channel statistics, and filters channels based on predefined quality criteria such as subscriber count, total views, and video count.<br><br>The final result is stored in a spreadsheet where channels are either added as new entries or updated if they already exist. This workflow is useful for influencer discovery, lead generation, or creator research. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for scheduled trigger block |  |  | ## Step 1. Schedule workflow trigger<br><br>This workflow starts automatically at a scheduled interval.<br><br>The scheduler allows you to run the automation periodically, for example every 5 minutes, every hour, or once per day depending on your discovery needs.<br><br>Each run processes the next available keyword from the keyword list. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for YouTube search block |  |  | ## Step 3. Search YouTube Videos<br><br>Using the selected keyword, the workflow searches YouTube videos through the YouTube API.<br><br>Videos are used as the discovery source because every video is linked to a channel.<br><br>By analyzing these results, the workflow can identify relevant creators within the niche. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for keyword processing block |  |  | ## Step 2. Retrieve Target Keywords and Process One Keyword at a Time.<br><br>This step retrieves the list of target keywords from the data source. Each keyword represents a topic or niche that will be used to search for related YouTube videos and discover relevant channels.<br><br>Example keywords might include:<br>• dashcam<br>• productivity tools<br>• AI automation<br>• gaming highlights<br><br>To avoid duplicate processing, the workflow selects the first unprocessed keyword from the list.<br><br>Once selected, the keyword is immediately marked as processed to ensure it is not used again in future runs.<br><br>This ensures that the workflow processes each keyword only once. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note for persistence block |  |  | ## Step 5. Save Results to Spreadsheet<br><br>Qualified channels are saved to the spreadsheet.<br><br>If the channel already exists, its information is updated. If it is new, a new row is added.<br><br>This creates a continuously growing database of qualified YouTube channels. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas note for extraction/filtering block |  |  | ## Step 4. Extract Channel IDs, Retrieve Channel Stats, and Filter Qualified Channels<br>From the video results, the workflow extracts the unique channel IDs of the creators who published those videos.<br><br>Duplicate channel IDs are removed to avoid requesting the same channel statistics multiple times.<br><br>For each discovered channel, the workflow retrieves detailed statistics such as:<br>• Subscriber count<br>• Total number of videos<br>• Total channel views<br><br>These metrics are used to evaluate whether the channel meets the qualification criteria.<br><br>Channels are filtered based on predefined thresholds to identify high quality creators.<br><br>Example criteria:<br>• Minimum subscribers<br>• Minimum number of videos<br>• Minimum total views<br><br>Only channels that meet all requirements proceed to the next step. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas note for setup prerequisites |  |  | ## Setup instructions<br><br>Before running this workflow, configure the following:<br><br>Add your YouTube API credentials.<br>Connect your spreadsheet or database where keywords and channel data are stored.<br>Adjust the filtering thresholds to match your target creator profile. |

---

# 4. Reproducing the Workflow from Scratch

## Prerequisites
1. Create or prepare a Google Spreadsheet with:
   - **Sheet1** for keywords, with columns at minimum:
     - `Keyword`
     - `Status`
     - `Last Run`
   - **Sheet2** for discovered channels, with columns:
     - `Channel ID`
     - `Channel Name`
     - `Subscribers`
     - `Total Videos`
     - `Total Views`
     - `Country`
     - `Email`
     - `Instagram`
     - `Twitter`
     - `Tiktok`
     - `Discord`
     - `Facebook`
2. Add a few keyword rows in `Sheet1` with `Status = Pending`.
3. Configure credentials:
   - **Google Sheets OAuth2**
   - **YouTube OAuth2**
   - Optional: **Apify OAuth2** if you want the disabled alternative branch.

---

## Active workflow build steps

### 1. Create the Schedule Trigger node
1. Add a **Schedule Trigger** node.
2. Set the schedule rule to **Cron**.
3. Use the expression:
   - `* * * * *`
4. This runs the workflow every minute.
5. Rename it to **Schedule Trigger**.

### 2. Create the keyword reader node
1. Add a **Google Sheets** node.
2. Rename it to **List of Target Keywords**.
3. Select your **Google Sheets OAuth2** credential.
4. Set the target spreadsheet to your keyword file.
5. Select `Sheet1`.
6. Configure it to read rows.
7. Add a filter:
   - Column: `Status`
   - Value: `Pending`
8. Connect **Schedule Trigger -> List of Target Keywords**.

### 3. Create the single-keyword selector
1. Add a **Code** node.
2. Rename it to **Get the first keyword**.
3. Paste this logic:
   ```javascript
   const items = $input.all();

   if (!items || items.length === 0) {
     return [];
   }

   return [items[0]];
   ```
4. Connect **List of Target Keywords -> Get the first keyword**.

### 4. Create the keyword status updater
1. Add another **Google Sheets** node.
2. Rename it to **Update Keyword Status -> Processing**.
3. Select the same Google Sheets credential.
4. Target the same spreadsheet and `Sheet1`.
5. Set operation to **Append or Update Row**.
6. Set the matching column to:
   - `Keyword`
7. Map fields:
   - `Keyword` = `{{ $json.Keyword }}`
   - `Status` = `Processed`
   - `Last Run` = `{{ $now }}`
8. Connect **Get the first keyword -> Update Keyword Status -> Processing**.

> Important: although the node name says “Processing”, it actually writes `Processed`. You may want to change either the node name or the written status for clarity.

### 5. Create the YouTube search node
1. Add an **HTTP Request** node.
2. Rename it to **Search Videos based on Keywords**.
3. Set:
   - Method: `GET`
   - URL: `https://www.googleapis.com/youtube/v3/search`
4. Enable query parameters:
   - `part` = `snippet`
   - `q` = `{{ $json.Keyword }}`
   - `type` = `video`
   - `maxResults` = `50`
5. Set authentication to **Predefined Credential Type**.
6. Choose **YouTube OAuth2** credential type and your account.
7. Connect **Update Keyword Status -> Processing -> Search Videos based on Keywords**.

### 6. Create the channel ID extractor
1. Add a **Code** node.
2. Rename it to **Extract Channel Ids from the Video Results**.
3. Paste:
   ```javascript
   const items = $json.items || [];

   const channelIds = items
     .map(item => item.snippet?.channelId)
     .filter(id => id !== undefined);

   const uniqueChannelIds = [...new Set(channelIds)];
   const channelIdsString = uniqueChannelIds.join(",");

   return [
     {
       json: {
         channelIdsString
       }
     }
   ];
   ```
4. Connect **Search Videos based on Keywords -> Extract Channel Ids from the Video Results**.

### 7. Create the YouTube channel stats node
1. Add another **HTTP Request** node.
2. Rename it to **Get Channel Stats**.
3. Set:
   - Method: `GET`
   - URL: `https://www.googleapis.com/youtube/v3/channels`
4. Add query parameters:
   - `part` = `statistics, snippet, brandingSettings`
   - `id` = `{{ $json.channelIdsString }}`
5. Use **Predefined Credential Type** with your **YouTube OAuth2** credential.
6. Connect **Extract Channel Ids from the Video Results -> Get Channel Stats**.

> Recommended correction: use `{{ $json.channelIdsString }}` directly. The exported workflow appears to contain an incorrect `=id=...` pattern in this field.

### 8. Create the qualification filter node
1. Add a **Code** node.
2. Rename it to **Filter based on Criteria**.
3. Paste:
   ```javascript
   const channels = $json.items || [];

   const subscriberThreshold = 10000;
   const videoThreshold = 100;
   const viewThreshold = 500000;

   const emailRegex = /[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/;

   const passedChannels = channels
     .map(channel => {
       const subs = parseInt(channel.statistics?.subscriberCount || "0");
       const vids = parseInt(channel.statistics?.videoCount || "0");
       const views = parseInt(channel.statistics?.viewCount || "0");

       if (subs < subscriberThreshold || vids < videoThreshold || views < viewThreshold) {
         return null;
       }

       const description = channel.brandingSettings?.channel?.description || "";
       const descEmails = (description.match(emailRegex) || []);

       const existingEmails = channel.emails || [];
       const allEmails = [...existingEmails, ...descEmails].filter(Boolean);
       const cleanEmail = allEmails.length ? allEmails[0] : "";

       const socials = channel.brandingSettings?.channel?.externalLinks || {};

       return {
         json: {
           retry: false,
           channelId: channel.id,
           title: channel.snippet?.title,
           country: channel.snippet?.country || "",
           subscribers: subs,
           videos: vids,
           views: views,
           email: cleanEmail,
           instagram: socials.instagram || "",
           twitter: socials.twitter || "",
           tiktok: socials.tiktok || "",
           discord: socials.discord || "",
           facebook: socials.facebook || ""
         }
       };
     })
     .filter(channel => channel !== null);

   if (passedChannels.length === 0) {
     return [{ json: { retry: true } }];
   }

   return passedChannels;
   ```
4. Connect **Get Channel Stats -> Filter based on Criteria**.

> Recommended improvement: instead of returning `{ retry: true }`, return `[]` when no channels qualify. That prevents blank rows downstream.

### 9. Create the destination Google Sheets node
1. Add a **Google Sheets** node.
2. Rename it to **Append or update row in sheet**.
3. Select your **Google Sheets OAuth2** credential.
4. Target the same spreadsheet.
5. Select `Sheet2`.
6. Set operation to **Append or Update Row**.
7. Set matching column:
   - `Channel ID`
8. Map the columns:
   - `Channel ID` = `{{ $json.channelId }}`
   - `Channel Name` = `{{ $json.title }}`
   - `Subscribers` = `{{ $json.subscribers }}`
   - `Total Videos` = `{{ $json.videos }}`
   - `Total Views` = `{{ $json.views }}`
   - `Country` = `{{ $json.country }}`
   - `Email` = `{{ $json.email }}`
   - `Instagram` = `{{ $json.instagram }}`
   - `Twitter` = `{{ $json.twitter }}`
   - `Tiktok` = `{{ $json.tiktok }}`
   - `Discord` = `{{ $json.discord }}`
   - `Facebook` = `{{ $json.facebook }}`
9. Connect **Filter based on Criteria -> Append or update row in sheet**.

---

## Optional disabled Apify branch

If you want to recreate the disabled alternative path as reference:

### 10. Create the Apify actor node
1. Add an **Apify** node.
2. Rename it to **Run an Actor and get dataset**.
3. Choose operation **Run actor and get dataset**.
4. Select your **Apify OAuth2** credential.
5. Choose actor:
   - `Youtube Channel Email Scraper By Keyword`
6. Set custom input body:
   ```json
   {
     "country": "United States",
     "keywords": ["{{ $json.Keyword }}"],
     "maxLeadsPerKeyword": 100,
     "scrapeLeadsWithEmail": true
   }
   ```
7. Leave it disabled unless you intend to use it.

### 11. Create the Apify data cleaner
1. Add a **Code** node.
2. Rename it to **Code in JavaScript**.
3. Paste:
   ```javascript
   const emailRegex = /[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/;

   const subscriberThreshold = 10000;
   const videoThreshold = 100;

   return $input.all()
     .map(item => {
       const emails = item.json.emails || [];
       const socials = item.json.social_links || {};

       let cleanEmail = "";
       if (Array.isArray(emails)) {
         const match = emails
           .map(e => e.match(emailRegex))
           .find(e => e !== null);
         cleanEmail = match ? match[0] : "";
       }

       const cleanedItem = {
         ...item.json,
         email: cleanEmail,
         instagram: socials.instagram || "",
         twitter: socials.twitter || "",
         tiktok: socials.tiktok || "",
         discord: socials.discord || "",
         facebook: socials.facebook || ""
       };

       return cleanedItem;
     })
     .filter(channel => {
       const subs = parseInt(channel.subscribers || "0");
       const vids = parseInt(channel.videos || "0");
       return subs >= subscriberThreshold && vids >= videoThreshold;
     })
     .map(channel => ({ json: channel }));
   ```
4. Connect **Run an Actor and get dataset -> Code in JavaScript**.
5. Disable the node if you are preserving parity with the provided workflow.

### 12. Create the Apify output sheet writer
1. Add a **Google Sheets** node.
2. Rename it to **Append or update row in sheet1**.
3. Use the same spreadsheet.
4. Select sheet `Using Apify`.
5. Set operation to **Append or Update Row**.
6. Set matching column:
   - `Channel URL`
7. Map:
   - `Channel Name` = `{{ $json.channel_name }}`
   - `Channel URL` = `{{ $json.channel_url }}`
   - `Email` = `{{ $json.email }}`
   - `Instagram` = `{{ $json.instagram }}`
   - `Tiktok` = `{{ $json.tiktok }}`
   - `Facebook` = `{{ $json.facebook }}`
   - `Discord` = `{{ $json.discord }}`
   - `Twitter` = `{{ $json.twitter }}`
   - `Subscribers` = `{{ $json.subscriber_count.match(/\d+/)[0] }}`
   - `Videos Count` = `{{ $json.video_count.match(/\d+/)[0] }}`
   - `Keyword` = `{{ $json.keyword }}`
8. Connect **Code in JavaScript -> Append or update row in sheet1**.
9. Disable this node if you want to match the original workflow.

> Recommended improvement: replace the regex extraction with safer normalization, because `.match(/\d+/)[0]` can fail or truncate formatted numbers.

---

## Recommended fixes before production use
1. Change the schedule from every minute unless you truly need that frequency.
2. Update keyword status only after a successful downstream run, or introduce separate statuses like:
   - `Pending`
   - `Processing`
   - `Processed`
   - `Failed`
3. Fix the `Get Channel Stats` `id` parameter so it only sends the comma-separated channel IDs.
4. Change the “no qualified channels” return in **Filter based on Criteria** from `{ retry: true }` to no output.
5. Add error handling or branching for:
   - no pending keywords
   - no search results
   - empty channel ID list
   - YouTube API failures
   - Google Sheets write failures
6. Consider adding pagination if you want to process more than 50 search results per keyword.
7. Consider storing the source keyword with each saved channel in `Sheet2` for traceability.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow overview: automatically discovers high quality YouTube channels from target keywords, retrieves channel stats, filters on quality criteria, and stores results in a spreadsheet for lead generation or creator research. | Canvas documentation |
| Step 1 note: workflow starts automatically at a scheduled interval; each run processes the next available keyword. | Canvas documentation |
| Step 2 note: retrieves target keywords, selects the first unprocessed keyword, and marks it as processed to avoid duplicates. | Canvas documentation |
| Step 3 note: uses YouTube video search as the discovery source because each video is linked to a channel. | Canvas documentation |
| Step 4 note: extracts unique channel IDs, retrieves channel metrics, and filters channels based on minimum quality thresholds. | Canvas documentation |
| Step 5 note: saves qualified channels to a spreadsheet using update-or-insert behavior. | Canvas documentation |
| Setup note: configure YouTube API credentials, connect the spreadsheet or database, and adjust threshold values to fit the target creator profile. | Canvas documentation |