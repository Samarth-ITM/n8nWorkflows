Generate Automated Content Performance Reports from YouTube, TikTok and Skool with AI

https://n8nworkflows.xyz/workflows/generate-automated-content-performance-reports-from-youtube--tiktok-and-skool-with-ai-11483


# Generate Automated Content Performance Reports from YouTube, TikTok and Skool with AI

### 1. Workflow Overview

This workflow automates the collection, analysis, and reporting of weekly content performance metrics from YouTube, TikTok, and Skool communities. It targets content creators, agencies, and community managers who want to automate their weekly metrics reporting without manual data entry, enabling data-driven content strategy improvements.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Data Collection**: Periodically triggers data scraping for all platforms.
- **1.2 Social Media Data Scraping and Extraction**: Retrieves raw data from YouTube, TikTok, and Skool.
- **1.3 Data Aggregation and Mapping**: Combines and maps collected metrics into unified structures.
- **1.4 Weekly Content Filtering and Metrics Calculation**: Filters recent content (last 7 days), extracts detailed stats, and calculates engagement and growth metrics.
- **1.5 Historical Data Lookup and Growth Computation**: Searches Airtable for last week’s data to compute growth.
- **1.6 AI-Powered Analysis Report Generation**: Uses an LLM to generate a strategic, HTML-formatted performance report.
- **1.7 Reporting and Data Archival**: Sends the HTML report via email and records weekly KPIs in Airtable.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Collection

**Overview:**  
This block triggers the workflow on schedules to periodically update current content metrics and perform weekly comprehensive analysis.

**Nodes Involved:**  
- Schedule Trigger  
- Schedule Trigger1  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow every 5 minutes for near real-time updates of metrics.  
  - Configuration: Interval set to every 5 minutes (minutes field).  
  - Inputs: None (trigger node).  
  - Outputs: Connects to initial scraping nodes for current metrics.  
  - Edge Cases: Missed triggers if n8n instance is down or paused.  

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Initiates the weekly comprehensive metrics gathering and report generation every Sunday at 21:00 (9 PM).  
  - Configuration: Cron expression set to "0 21 * * 0" (weekly Sunday 9 PM).  
  - Inputs: None (trigger node).  
  - Outputs: Connects to scraping and aggregation nodes for weekly report.  
  - Edge Cases: Missed runs could delay weekly reports; time zone considerations if server timezone differs.  

---

#### 1.2 Social Media Data Scraping and Extraction

**Overview:**  
Retrieves raw metrics and follower counts from YouTube, TikTok, and Skool using API calls and web scraping techniques.

**Nodes Involved:**  
- Get skool members  
- Get skool members2  
- Markdown  
- Markdown1  
- Get channel stats  
- Get channel stats1  
- Get tiktok followers  
- extract current tiktok followers  
- Get tiktok data  
- Format Channel stats  
- Get last weeks vids  
- Get Video Data  
- Get Transcripts  
- Format youtube sub count  
- Format youtube video stats  
- Extract Skool member count  
- Extract member count1  
- Extract follower count for tiktok  

**Node Details:**  

- **Get skool members / Get skool members2**  
  - Type: HTTP Request  
  - Role: Scrapes Skool community web page for member data (HTML response).  
  - Configuration: GET request to Skool URL with headers mimicking a browser user-agent to avoid blocking.  
  - Inputs: Trigger nodes  
  - Outputs: HTML content passed to Markdown nodes.  
  - Edge Cases: Possible HTTP errors, changes in Skool HTML structure causing extraction failures.  
  - OnError: Continue error output to avoid workflow halt on errors.  

- **Markdown / Markdown1**  
  - Type: Markdown  
  - Role: Converts scraped HTML data into markdown or text format for easier extraction.  
  - Inputs: From Get skool members nodes.  
  - Outputs: Feeds into extraction code nodes.  

- **Extract Skool member count / Extract member count1**  
  - Type: Set  
  - Role: Extracts numeric Skool member count from markdown or HTML text using regex patterns.  
  - Configuration: Regex patterns checking for “X members” or “[XMembers]” in text.  
  - Inputs: Markdown nodes.  
  - Outputs: Member count as string.  
  - Edge Cases: Regex may fail if Skool page layout changes, resulting in missing or incorrect member counts.  

- **Get channel stats / Get channel stats1**  
  - Type: YouTube node (YouTube OAuth2)  
  - Role: Fetches current channel statistics, especially subscriber count.  
  - Configuration: Operation “get” on channel ID with “statistics” part requested.  
  - Inputs: Scheduled trigger nodes.  
  - Outputs: Raw channel statistics JSON.  
  - Credentials: YouTube OAuth2 credentials required.  
  - Edge Cases: OAuth token expiry, API quota limits, channel ID misconfiguration.  

- **Format youtube sub count**  
  - Type: Set  
  - Role: Extracts subscriber count from YouTube stats JSON and formats it as string under “YT subs”.  
  - Inputs: Get channel stats node.  

- **Get tiktok followers**  
  - Type: HTTP Request  
  - Role: Scrapes TikTok user page HTML to get raw data.  
  - Configuration: GET request to TikTok profile URL with headers mimicking browser requests.  
  - Inputs: Scheduled trigger node.  
  - Outputs: HTML response passed to extraction code node.  
  - Edge Cases: TikTok webpage structure changes, rate limiting, blocking by TikTok.  

- **extract current tiktok followers**  
  - Type: Code  
  - Role: Parses TikTok HTML to extract follower count and other profile stats from embedded JSON script tag.  
  - Inputs: Get tiktok followers node.  
  - Outputs: JSON with followerCount and additional profile stats.  
  - Edge Cases: Parsing fails if HTML structure changes or JSON script is missing. Returns error JSON on failure.  

- **Get tiktok data**  
  - Type: HTTP Request (POST) to Apify TikTok Scraper actor  
  - Role: Retrieves TikTok video data for the profile, including stats of recent videos.  
  - Configuration: JSON body with profiles array and options to not download media, only metadata.  
  - Inputs: Scheduled trigger1 node.  
  - Outputs: TikTok video data items.  
  - Edge Cases: API token invalid/expired, request timeouts (timeout set to 31 seconds), rate limiting.  

- **Format Channel stats**  
  - Type: Set  
  - Role: Maps TikTok video data fields to normalized field names used downstream (videoLikes, videoSaves, etc.).  
  - Inputs: Get tiktok data node.  

- **Get last weeks vids**  
  - Type: YouTube node (YouTube OAuth2)  
  - Role: Retrieves list of videos posted in the last 7 days on the YouTube channel.  
  - Configuration: Filter by channelId and publishedAfter set dynamically to 7 days ago.  
  - Inputs: Scheduled trigger1 node.  
  - Outputs: List of videos.  
  - Edge Cases: API quota limits, empty video list if no recent uploads.  

- **Get Video Data**  
  - Type: HTTP Request  
  - Role: For each video, fetches detailed video data including statistics, snippet, and content details from YouTube Data API v3.  
  - Inputs: Get last weeks vids node.  
  - Outputs: Detailed video info for each video.  
  - Edge Cases: API request failures, missing data fields, quota limits.  

- **Get Transcripts**  
  - Type: HTTP Request (POST) to Apify YouTube Transcripts actor  
  - Role: Retrieves transcripts for the videos, enhancing content analysis.  
  - Configuration: JSON body specifies URLs and options; uses Apify proxy with retries.  
  - Inputs: Get Video Data node.  
  - Outputs: Transcript text JSON.  
  - Edge Cases: API token issues, transcript unavailability, rate limits, network errors.  

- **Format youtube video stats**  
  - Type: Set  
  - Role: Extracts and maps YouTube video stats and transcript to normalized fields.  
  - Inputs: Get Transcripts node.  
  - Outputs: Normalized video stats objects.  

---

#### 1.3 Data Aggregation and Mapping

**Overview:**  
Combines data from YouTube, TikTok, and Skool scrapes, mapping key counts into a consistent format for downstream processing.

**Nodes Involved:**  
- Merge  
- Merge1  
- Merge2  
- combine the data  
- Map out all 3 counts for each social platform  
- combine items  
- Extract follower count for tiktok  
- Aggregate1  

**Node Details:**  

- **Merge**  
  - Type: Merge (3 inputs)  
  - Role: Merges outputs from Skool members, YouTube channel stats, and TikTok follower extraction.  
  - Inputs: Format youtube sub count, Extract Skool member count, extract current tiktok followers.  
  - Outputs: Combined data for mapping.  

- **combine the data**  
  - Type: Aggregate  
  - Role: Aggregates all incoming items into an array for easier processing.  
  - Inputs: From the Merge node.  
  - Outputs: Aggregated array of platform counts.  

- **Map out all 3 counts for each social platform**  
  - Type: Set  
  - Role: Maps aggregated counts into named fields: "Skool members", "Sub count YT", "TT followers".  
  - Inputs: combine the data node.  

- **Search records**  
  - Type: Airtable Search  
  - Role: Looks up existing record in Airtable "Current metrics" table using "Calculation" as match key.  
  - Inputs: Map out all 3 counts node.  
  - Outputs: Existing record for update.  

- **Update record**  
  - Type: Airtable Update  
  - Role: Updates the Airtable record with current counts for YT subs, Skool members, and TikTok followers.  
  - Inputs: Search records node.  

- **Merge1**  
  - Type: Merge (5 inputs)  
  - Role: Merges data from recent YouTube videos, their stats, TikTok video stats, Skool member counts, and YouTube channel stats.  
  - Inputs: Format youtube video stats, Get channel stats1, Extract member count1, Format Channel stats, last weeks lable1.  
  - Outputs: Combined detailed weekly data for analysis.  

- **Merge2**  
  - Type: Merge  
  - Role: Merges TikTok data with other platforms’ data after filtering last 7 days.  
  - Inputs: last 7 days? node, Format Channel stats, Get skool members2.  

- **combine items**  
  - Type: Code  
  - Role: Converts multiple items into a single combined JSON array for TikTok video data.  
  - Inputs: Get tiktok data.  

- **Extract follower count for tiktok**  
  - Type: Set  
  - Role: Extracts follower count from combined TikTok items.  

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates all item data for analysis and feeding into AI model.  
  - Inputs: Merge1 node outputs.  

---

#### 1.4 Weekly Content Filtering and Metrics Calculation

**Overview:**  
Filters content posted in the last 7 days, calculates weekly engagement metrics and growth in followers/subscribers/members compared to last week.

**Nodes Involved:**  
- last 7 days?  
- add up metrics  
- last weeks lable  
- last weeks lable1  
- Search records1  
- Search records2  
- current metrics  
- Edit Fields4  
- calculate metrics  
- Get week lable  

**Node Details:**  

- **last 7 days?**  
  - Type: If  
  - Role: Filters TikTok posts to only those published within the last 7 days using timestamp comparison.  
  - Inputs: Format Channel stats node.  
  - Outputs: Passes filtered items downstream.  

- **add up metrics**  
  - Type: Code  
  - Role: Totals YouTube and TikTok engagement metrics (views, likes, comments, shares, saves, favorites) across filtered content.  
  - Inputs: Aggregate1 node output (all combined data).  
  - Outputs: JSON with summed metrics.  
  - Edge Cases: Empty input array results in zeros for all metrics.  

- **last weeks lable / last weeks lable1**  
  - Type: Code  
  - Role: Computes the ISO week label string for the previous week in the format "YYYY-W##".  
  - Inputs: current metrics node (last weeks lable), Aggregate1 (last weeks lable1).  
  - Outputs: JSON with prevWeekLabel.  

- **Search records1 / Search records2**  
  - Type: Airtable Search  
  - Role: Queries Airtable "Weekly Content KPIs" table for the record matching the previous week's label.  
  - Inputs: last weeks lable nodes.  
  - Outputs: Last week’s stored metrics for growth calculation.  
  - Edge Cases: Missing record means baseline data is not found; manual creation required on first run.  

- **current metrics**  
  - Type: Airtable Search  
  - Role: Fetches the latest current metrics record from Airtable "Current metrics" table for comparison.  
  - Inputs: add up metrics node output.  

- **Edit Fields4**  
  - Type: Set  
  - Role: Prepares dataset with last week and current metrics side-by-side for YouTube subs, TikTok followers, and Skool members as strings.  
  - Inputs: Search records2 node output (last week’s data).  

- **calculate metrics**  
  - Type: Code  
  - Role: Calculates net growth for subscribers/followers/members by subtracting last week’s totals from current, producing absolute numbers.  
  - Inputs: Edit Fields4 node.  
  - Outputs: JSON with “YouTube subs added”, “TikTok followers added”, “Skool members added”.  

- **Get week lable**  
  - Type: Code  
  - Role: Generates the current week’s ISO week label for storing this week’s data.  

---

#### 1.5 Historical Data Lookup and Growth Computation

**Overview:**  
Looks up last week’s data in Airtable and calculates weekly growth of followers, subscribers, and community members.

**Nodes Involved:**  
- Search records1  
- Search records2  
- Edit Fields4  
- calculate metrics  

**Node Details:**  

- **Search records1 / Search records2**  
  - Type: Airtable Search  
  - Role: Finds last week’s record in the “Weekly Content KPIs” Airtable base.  
  - Inputs: last weeks lable or last weeks lable1 nodes.  
  - Outputs: Last week’s metrics.  

- **Edit Fields4**  
  - Type: Set  
  - Role: Aligns last week and current week metrics for easy comparison.  

- **calculate metrics**  
  - Type: Code  
  - Role: Computes the numeric growth (absolute difference) between current and last week’s metrics.  

---

#### 1.6 AI-Powered Analysis Report Generation

**Overview:**  
Generates a detailed HTML email report analyzing weekly content performance, using a large language model (LLM) to interpret data and provide strategic recommendations.

**Nodes Involved:**  
- Generate analysis report  
- OpenRouter Chat Model  

**Node Details:**  

- **Generate analysis report**  
  - Type: Chain LLM (LangChain)  
  - Role: Sends combined weekly content metrics and last week’s totals to the OpenRouter AI model for analysis.  
  - Configuration:  
    - Uses Google Gemini 2.5 Flash model via OpenRouter API credentials.  
    - Provides a detailed prompt defining an analysis framework including performance overview, top performers by engagement, content analysis, key insights, and recommendations.  
    - The model returns a full HTML email with strict formatting rules to be sent directly.  
  - Inputs: Merged weekly data, last week’s stats, calculated growth metrics.  
  - Edge Cases: API rate limits, model errors, prompt misconfiguration.  

- **OpenRouter Chat Model**  
  - Type: AI Language Model node  
  - Role: Interfaces with OpenRouter API to generate textual output from prompt.  
  - Inputs: Generate analysis report node (as sub-node).  

---

#### 1.7 Reporting and Data Archival

**Overview:**  
Sends the generated HTML performance report via Gmail and archives the weekly KPI data into Airtable.

**Nodes Involved:**  
- Send html style report to your email  
- Create a record  

**Node Details:**  

- **Send html style report to your email**  
  - Type: Gmail node  
  - Role: Sends an email containing the LLM-generated HTML report to the configured email address.  
  - Configuration:  
    - Recipient email set in node parameter.  
    - Message content is the HTML from Generate analysis report.  
    - Gmail OAuth2 credentials required.  
  - Inputs: Generate analysis report node.  
  - Edge Cases: Authentication errors, quota limits, email delivery failures.  

- **Create a record**  
  - Type: Airtable Create  
  - Role: Stores the current week’s calculated KPIs into the "Weekly Content KPIs" table for historical reference.  
  - Inputs: calculate metrics node and Get week lable node.  
  - Edge Cases: Airtable API errors, schema mismatches.  

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                                     | Input Node(s)                                             | Output Node(s)                                           | Sticky Note                                                                                                               |
|--------------------------------|----------------------------------|----------------------------------------------------|-----------------------------------------------------------|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                 | Triggers frequent (5 min) scraping of current stats| None                                                      | Get skool members, Get channel stats, Get tiktok followers| ## Every 5 min content stats updater ... See Sticky Note11 content for details                                           |
| Get skool members             | HTTP Request                    | Scrapes Skool community HTML for members           | Schedule Trigger                                          | Markdown                                                 | ## Scrape your skool community ... See Sticky Note1 content                                                               |
| Markdown                      | Markdown                       | Converts Skool HTML to markdown                      | Get skool members                                        | Extract Skool member count                                |                                                                                                                           |
| Extract Skool member count    | Set                            | Extracts Skool member count number from text        | Markdown                                                 | Merge                                                    |                                                                                                                           |
| Get channel stats             | YouTube                        | Fetches current YouTube channel statistics          | Schedule Trigger                                          | Format youtube sub count                                 | ## Get current YT sub count ... See Sticky Note4 content                                                                   |
| Format youtube sub count      | Set                            | Extracts subscriber count from YouTube stats        | Get channel stats                                        | Merge                                                    |                                                                                                                           |
| Get tiktok followers          | HTTP Request                   | Scrapes TikTok profile HTML for follower data       | Schedule Trigger                                          | extract current tiktok followers                         | ## Scrape your tiktok ... See Sticky Note5 content                                                                          |
| extract current tiktok followers | Code                          | Parses TikTok HTML to extract follower count         | Get tiktok followers                                     | Merge                                                    |                                                                                                                           |
| Merge                        | Merge                          | Combines Skool, YouTube, and TikTok current counts  | Extract Skool member count, Format youtube sub count, extract current tiktok followers | combine the data                                        |                                                                                                                           |
| combine the data             | Aggregate                      | Aggregates the combined counts into array            | Merge                                                    | Map out all 3 counts for each social platform            |                                                                                                                           |
| Map out all 3 counts for each social platform | Set              | Maps aggregated counts into named fields             | combine the data                                        | Search records                                           |                                                                                                                           |
| Search records               | Airtable Search                | Finds existing record in Airtable "Current metrics" | Map out all 3 counts for each social platform            | Update record                                           |                                                                                                                           |
| Update record               | Airtable Update                | Updates Airtable record with latest counts           | Search records                                          | None                                                     |                                                                                                                           |
| Schedule Trigger1            | Schedule Trigger               | Triggers weekly comprehensive scraping and analysis | None                                                      | Get last weeks vids, Get skool members2, Get tiktok data, Get channel stats1, Get week lable |                                                                                                                           |
| Get last weeks vids          | YouTube                       | Gets videos posted in last 7 days                     | Schedule Trigger1                                       | Get Video Data                                          | ## Scrape your youtube channel ... See Sticky Note2 content                                                                 |
| Get Video Data               | HTTP Request                  | Gets detailed video data for each video               | Get last weeks vids                                    | Get Transcripts                                        |                                                                                                                           |
| Get Transcripts              | HTTP Request                  | Retrieves YouTube video transcripts                   | Get Video Data                                         | Format youtube video stats                             |                                                                                                                           |
| Format youtube video stats   | Set                          | Maps YouTube video stats to normalized fields         | Get Transcripts                                       | Merge1                                                 |                                                                                                                           |
| Get skool members2           | HTTP Request                 | Scrapes Skool community HTML (weekly run)             | Schedule Trigger1                                     | Markdown1                                               | ## Scrape your skool community ... See Sticky Note1 content (same as earlier)                                              |
| Markdown1                   | Markdown                    | Converts Skool HTML to markdown (weekly run)            | Get skool members2                                   | Extract member count1                                  |                                                                                                                           |
| Extract member count1        | Set                         | Extracts Skool member count from markdown text         | Markdown1                                            | Merge1                                                 |                                                                                                                           |
| Get tiktok data              | HTTP Request                | Retrieves TikTok video data via Apify actor            | Schedule Trigger1                                   | Format Channel stats, combine items                    | ## Scrape your tiktok ... See Sticky Note5 content (same)                                                                  |
| Format Channel stats         | Set                        | Maps TikTok video stats to normalized fields           | Get tiktok data                                      | last 7 days?                                          |                                                                                                                           |
| last 7 days?                | If                         | Filters TikTok videos published within last 7 days     | Format Channel stats                                | Merge2                                                |                                                                                                                           |
| Merge1                      | Merge                      | Merges YouTube video stats, channel stats, Skool members, TikTok stats, and last week label | Format youtube video stats, Get channel stats1, Extract member count1, Format Channel stats, last weeks lable1 | Aggregate1                                          |                                                                                                                           |
| Merge2                      | Merge                      | Merges TikTok filtered data with other platforms' data | last 7 days?, Format Channel stats, Get skool members2 | Merge1                                                |                                                                                                                           |
| combine items               | Code                      | Combines TikTok items into array                         | Get tiktok data                                    | Extract follower count for tiktok                   |                                                                                                                           |
| Extract follower count for tiktok | Set                  | Extracts follower count from TikTok combined items       | combine items                                      | Merge2                                                |                                                                                                                           |
| Aggregate1                  | Aggregate                | Aggregates all merged items into one array for analysis  | Merge1                                             | add up metrics, last weeks lable1                    |                                                                                                                           |
| add up metrics              | Code                    | Sums total engagement metrics across YouTube and TikTok | Aggregate1                                         | current metrics                                      | ## Calculate weekly stats ... See Sticky Note9 content                                                                      |
| current metrics             | Airtable Search          | Retrieves current metrics from Airtable for comparison   | add up metrics                                     | last weeks lable                                     |                                                                                                                           |
| last weeks lable            | Code                    | Generates previous week’s ISO week label                   | current metrics                                    | Search records2                                    |                                                                                                                           |
| Search records2             | Airtable Search          | Finds last week’s KPI record in Airtable                    | last weeks lable                                  | Edit Fields4                                       |                                                                                                                           |
| Edit Fields4                | Set                     | Prepares last week and current week metrics for growth calc | Search records2                                   | calculate metrics                                  |                                                                                                                           |
| calculate metrics           | Code                    | Calculates absolute growth of subs/followers/members         | Edit Fields4                                      | Create a record                                   |                                                                                                                           |
| Create a record             | Airtable Create          | Stores this week’s KPI data in Airtable                      | calculate metrics                                  | None                                               | ## Calculate weekly stats ... See Sticky Note9 content (same as above)                                                     |
| last weeks lable1           | Code                    | Computes previous week label for use in weekly analysis      | Aggregate1                                        | Search records1                                   |                                                                                                                           |
| Search records1             | Airtable Search          | Searches for last week’s weekly KPI record                    | last weeks lable1                                 | Generate analysis report                          |                                                                                                                           |
| Generate analysis report    | Chain LLM                | Uses AI to generate detailed HTML email analysis and recommendations | Search records1                                   | Send html style report to your email             | ## Analyze weekly content performance and send HTML style analysis email ... See Sticky Note8 content                       |
| OpenRouter Chat Model       | AI Language Model         | Interfaces with OpenRouter API to process LLM prompt          | Generate analysis report                          | Generate analysis report (response)                |                                                                                                                           |
| Send html style report to your email | Gmail                | Sends the generated HTML report as an email                   | Generate analysis report                          | None                                               |                                                                                                                           |
| Get week lable              | Code                    | Generates current week ISO label for KPI storage                | Schedule Trigger1                                 | Merge1                                             | ## Creates the "week lable" for storing stats in airtable ... See Sticky Note6 content                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create two Schedule Trigger nodes:**  
   - Name one “Schedule Trigger” with interval every 5 minutes.  
   - Name the other “Schedule Trigger1” with cron expression `0 21 * * 0` for weekly Sunday 9 PM trigger.

2. **Add Skool scraping nodes:**  
   - Create two HTTP Request nodes named “Get skool members” and “Get skool members2”.  
   - Configure URL to your Skool community page. Add appropriate headers mimicking browsers.  
   - Connect “Schedule Trigger” to “Get skool members”, and “Schedule Trigger1” to “Get skool members2”.

3. **Add Markdown nodes:**  
   - Add two Markdown nodes named “Markdown” and “Markdown1”.  
   - Connect from respective Skool HTTP nodes.

4. **Add Set nodes to extract Skool members counts:**  
   - Add “Extract Skool member count” and “Extract member count1” Set nodes.  
   - Use regex on markdown content to extract numbers matching `(\d+)\s*members` or `\[(\d+)Members\]`.  
   - Connect Markdown nodes to these Set nodes.

5. **Add YouTube channel stats nodes:**  
   - Create two YouTube nodes named “Get channel stats” and “Get channel stats1”.  
   - Configure OAuth2 credentials and set operation “get” with part “statistics” and your YouTube channel ID.  
   - Connect “Schedule Trigger” to “Get channel stats”, “Schedule Trigger1” to “Get channel stats1”.

6. **Add “Format youtube sub count” Set node:**  
   - Extract subscriber count from YouTube stats JSON.  
   - Connect from “Get channel stats”.

7. **Add TikTok follower scraping:**  
   - Create HTTP Request node “Get tiktok followers” configured to GET your TikTok profile page URL with browser-like headers.  
   - Connect “Schedule Trigger” to this node.

8. **Add Code node “extract current tiktok followers”:**  
   - Parse HTML response from TikTok page for JSON data in script tag and extract follower count and other stats.  
   - Connect “Get tiktok followers” to this node.

9. **Add TikTok data retrieval:**  
   - Create HTTP Request node “Get tiktok data” to POST to Apify TikTok Scraper actor API with your token and profile.  
   - Connect “Schedule Trigger1” to this node.

10. **Add “Format Channel stats” Set node:**  
    - Map TikTok video data fields to normalized keys (videoLikes, videoSaves, etc.).  
    - Connect “Get tiktok data” to this node.

11. **Add “combine items” Code node:**  
    - Aggregate TikTok video items into array.  
    - Connect “Get tiktok data” to this node.

12. **Add “Extract follower count for tiktok” Set node:**  
    - Extract follower count from combined TikTok items.  
    - Connect “combine items” to it.

13. **Add Merge nodes to combine platform data:**  
    - “Merge” node to merge Skool members, YouTube subs, and TikTok followers (3 inputs).  
    - “Merge1” node to merge YouTube video stats, YouTube channel stats, Skool member count, TikTok stats, and last weeks label (5 inputs).  
    - “Merge2” node to merge TikTok filtered data with other platforms data.

14. **Add “Map out all 3 counts for each social platform” Set node:**  
    - Map merged data to named fields like “Skool members”, “Sub count YT”, “TT followers”.

15. **Add Airtable nodes:**  
    - “Search records” to find existing current metrics record by “Calculation” field.  
    - “Update record” to update current metrics.  
    - “Search records1” and “Search records2” to find last week’s KPI records by week label.  
    - “Create a record” to store new weekly KPIs.

16. **Add YouTube last week videos nodes:**  
    - “Get last weeks vids” YouTube node filtered by publishedAfter 7 days ago.  
    - “Get Video Data” HTTP Request node for detailed video data.  
    - “Get Transcripts” HTTP Request node to Apify YouTube Transcripts actor.  
    - “Format youtube video stats” Set node to normalize fields.

17. **Add “last 7 days?” If node:**  
    - Filters TikTok posts based on postedDateandTime within last 7 days.

18. **Add “add up metrics” Code node:**  
    - Sum engagement metrics (views, likes, comments, etc.) for YouTube and TikTok from aggregated data.

19. **Add “current metrics” Airtable search node:**  
    - Fetch current metrics for comparison.

20. **Add “last weeks lable” and “last weeks lable1” Code nodes:**  
    - Generate previous week ISO week labels for querying last week’s data.

21. **Add “Edit Fields4” Set node:**  
    - Prepare last week and current week fields side-by-side.

22. **Add “calculate metrics” Code node:**  
    - Compute weekly growth by subtracting last week’s counts from current.

23. **Add “Get week lable” Code node:**  
    - Generate current week label for storing KPIs.

24. **Add “Aggregate1” node:**  
    - Aggregate all combined weekly data for AI analysis input.

25. **Add AI nodes:**  
    - “OpenRouter Chat Model” with OpenRouter API credentials configured for model “google/gemini-2.5-flash”.  
    - “Generate analysis report” Chain LLM node configured with extensive prompt for performance analysis and HTML email generation.  
    - Connect “Search records1” (last week’s data) to “Generate analysis report” and “OpenRouter Chat Model”.

26. **Add “Send html style report to your email” Gmail node:**  
    - Configure with Gmail OAuth2 credentials, recipient email, subject line, and message content from AI report.

27. **Connect final nodes:**  
    - Connect “calculate metrics” to “Create a record” for weekly KPI archival.  
    - Connect “Generate analysis report” to “Send html style report to your email”.

28. **Credentials setup:**  
    - Configure and test credentials for YouTube OAuth2, Airtable API, Gmail OAuth2, OpenRouter API, and Apify token.  
    - Replace placeholder variables (e.g., YOUR_CHANNEL_ID, YOUR_SKOOL_URL_HERE, YOUR_API_KEY) with real values.

29. **Manual setup before first run:**  
    - Create baseline record in Airtable “Weekly Content KPIs” for previous week with current totals to allow growth calculation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Created by Jordan Austin for content creators, agencies, and community managers to automate weekly reporting and analysis. First run requires manual baseline record creation in Airtable.                                                                                                                                                                                                                                                                                                                                     | See Sticky Note13 content, accessible in workflow.                                                                             |
| APIs used: Apify (for YouTube transcripts and TikTok scraping), YouTube Data API, Airtable API, Gmail API, OpenRouter for AI model.                                                                                                                                                                                                                                                                                                                                                                                           | Apify: https://www.apify.com, YouTube API docs: https://developers.google.com/youtube/v3, Airtable docs: https://airtable.com   |
| Video on setting up Google APIs including YouTube and Gmail: https://youtube.com/shorts/FBRQNmCN0F8?feature=share                                                                                                                                                                                                                                                                                                                                                                                                              | n8n integration docs for Google credentials: https://docs.n8n.io/integrations/builtin/credentials/google/                       |
| Airtable base template to duplicate: https://airtable.com/appkessfths2QABG9/shr1ceynxbhDcTSxB                                                                                                                                                                                                                                                                                                                                                                                                                                  | Used for storing current metrics and weekly KPIs.                                                                               |
| Prompt instructions emphasize strict HTML formatting, correct calculation of engagement rates, and clear differentiation between quality (engagement) and scale (views). The report is designed to be emailed as a fully-formed HTML message with actionable insights.                                                                                                                                                                                                                                                          | Prompt and AI instructions embedded in “Generate analysis report” node.                                                        |
| Workflow handles error resilience gracefully by continuing on HTTP request errors (e.g., Skool or TikTok scraping) to ensure overall workflow stability.                                                                                                                                                                                                                                                                                                                                                                     | HTTP Request nodes with onError set to continueErrorOutput.                                                                    |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.