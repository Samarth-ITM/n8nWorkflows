Collect YouTube Channel Stats & Contact Info with Google Sheets and SerpAPI

https://n8nworkflows.xyz/workflows/collect-youtube-channel-stats---contact-info-with-google-sheets-and-serpapi-4585


# Collect YouTube Channel Stats & Contact Info with Google Sheets and SerpAPI

### 1. Workflow Overview

This workflow named **"Youtube Channel Intelligence Collector"** automates the process of collecting key YouTube channel statistics and contact information, then updating a Google Sheet to maintain a live intelligence dashboard. It is designed primarily for lead generation, influencer marketing research, and data enrichment use cases where continuous monitoring and updating of YouTube channel data is required.

The workflow is logically divided into four main functional blocks:

- **1.1 Channel Identification & Stats Fetching:** Detects new YouTube channel entries in a Google Sheet, resolves channel IDs, and fetches basic channel statistics (subscribers, total views, etc.).
- **1.2 Recent Video Intelligence:** Retrieves the 5 most recent videos from the channel, collects their view counts, and sums these views to gauge recent engagement.
- **1.3 Contact Intelligence:** Uses SerpAPI to scrape publicly available contact email addresses from the channel’s About page, circumventing YouTube API limitations.
- **1.4 Data Assembly & Sheet Update:** Organizes all collected data into a structured format and updates the original Google Sheet row corresponding to the channel with fresh insights.

---

### 2. Block-by-Block Analysis

#### 2.1 Channel Identification & Stats Fetching

**Overview:**  
This block triggers the workflow when a new YouTube channel is added to the Google Sheet. It converts input URLs or usernames into valid YouTube channel IDs and fetches core channel statistics essential for understanding the channel’s size and reach.

**Nodes Involved:**  
- New Channel Added (Google Sheets Trigger)  
- Get info about channel (HTTP Request)  
- Get Channel Stats (HTTP Request)

**Node Details:**

- **New Channel Added**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Watches a specified Google Sheet for newly added rows (channels).  
  - *Config:* Polls every minute on the sheet named “Sheet1” in the document “Youtube Channel Intelligence Collector.” Uses OAuth2 credentials for Google Sheets API.  
  - *Input:* Detects new row additions.  
  - *Output:* Passes the new channel data downstream.  
  - *Edge Cases:* Google Sheets API quota limits, network issues, or wrong document/sheet IDs could cause missed triggers.

- **Get info about channel**  
  - *Type:* HTTP Request  
  - *Role:* Calls YouTube Data API v3 ‘channels’ endpoint to fetch channel snippet and statistics by channelId.  
  - *Config:* URL format includes a placeholder channel ID and requires a valid YouTube API key (`YOUR_KEY`). Parameters: `part=statistics,snippet&id=CHANNEL_ID`.  
  - *Input:* Channel ID (assumed resolved or static in example).  
  - *Output:* Channel metadata including subscriber count, channel title, and statistics.  
  - *Edge Cases:* API rate limits, invalid API key, or non-existent channel ID can cause errors or empty responses.

- **Get Channel Stats**  
  - *Type:* HTTP Request  
  - *Role:* Uses YouTube Data API ‘search’ endpoint to fetch the 5 most recent videos uploaded by the channel.  
  - *Config:* Query params include `channelId`, `part=snippet`, `order=date`, `maxResults=5`, and API key.  
  - *Input:* Channel ID from previous node’s response JSON path `items[0].id`.  
  - *Output:* List of recent video snippets.  
  - *Edge Cases:* Empty video lists if channel has no videos, API errors, or quota exhaustion.

---

#### 2.2 Recent Video Intelligence

**Overview:**  
This block analyzes recent video performance by extracting video IDs, querying their individual statistics (specifically views), and summing views to understand recent audience engagement.

**Nodes Involved:**  
- Get Recent Video Views (HTTP Request)  
- Prepare views to sum (Set)  
- Sum Video Views (Code)

**Node Details:**

- **Get Recent Video Views**  
  - *Type:* HTTP Request  
  - *Role:* Fetches statistics for up to 5 recent video IDs simultaneously using YouTube’s ‘videos’ endpoint.  
  - *Config:* URL dynamically constructed using template expressions to insert video IDs from previous node’s items, plus dummy IDs `def456,ghi789` (likely placeholders or fallback). Requires API key.  
  - *Input:* Video IDs array from `Get Channel Stats` node.  
  - *Output:* Video statistics including individual view counts.  
  - *Edge Cases:* Invalid or missing video IDs, API quota exceeded, or malformed URLs.

- **Prepare views to sum**  
  - *Type:* Set  
  - *Role:* Extracts and assigns individual video view counts into separate named fields (`Video 1 views` through `Video 5 views`) for easier processing.  
  - *Config:* Uses expressions to map each video’s `statistics.viewCount` from JSON response.  
  - *Input:* Video statistics JSON.  
  - *Output:* Structured object with view counts as numbers.  
  - *Edge Cases:* Missing statistics or view counts for any video could cause undefined or zero values.

- **Sum Video Views**  
  - *Type:* Code (JavaScript)  
  - *Role:* Aggregates the view counts across the 5 videos calculated in the previous node.  
  - *Config:* Iterates over keys containing 'views' and sums their values into `totalViews`.  
  - *Input:* Structured view counts from Set node.  
  - *Output:* Adds `totalViews` field summarizing recent engagement.  
  - *Edge Cases:* Non-numeric or missing values could cause NaN or incorrect sums.

---

#### 2.3 Contact Intelligence

**Overview:**  
This block attempts to retrieve the public contact email from the YouTube channel’s About page by leveraging SerpAPI's YouTube Channel About scraper, which bypasses CAPTCHA protections that prevent direct YouTube API access to email data.

**Nodes Involved:**  
- Get Channel Email (SerpAPI)

**Node Details:**

- **Get Channel Email (SerpAPI)**  
  - *Type:* HTTP Request  
  - *Role:* Queries SerpAPI’s YouTube Channel About endpoint with the channel URL to extract email and other contact details.  
  - *Config:* URL constructed dynamically to include the channel URL from the Google Sheets trigger node, requires a SerpAPI API key (`YOUR_SERPAPI_KEY`).  
  - *Input:* Channel URL from the original sheet entry.  
  - *Output:* JSON including `channel_about_page.email` and other metadata.  
  - *Edge Cases:* No email present (common), SerpAPI quota limits, invalid API key, or changes in YouTube page structure that break parsing.

---

#### 2.4 Data Assembly & Sheet Update

**Overview:**  
This final block consolidates all gathered data (recent views, subscriber count, and email) into a clean object and updates the corresponding Google Sheet row to reflect the enriched data, making the sheet a live, actionable intelligence dashboard.

**Nodes Involved:**  
- Prepare Sheet Data (Set)  
- Update Channel Insights (Google Sheets)

**Node Details:**

- **Prepare Sheet Data**  
  - *Type:* Set  
  - *Role:* Formats the collected data into fields named `Recent Views`, `Total Subscribers`, and `Email` for sheet update.  
  - *Config:* Uses expressions to pull data from upstream nodes including totalViews from code node, subscriberCount from channel info, and email from SerpAPI response.  
  - *Input:* Data from `Sum Video Views`, `Get info about channel`, and `Get Channel Email (SerpAPI)`.  
  - *Output:* Single JSON object ready for sheet mapping.  
  - *Edge Cases:* Missing or empty fields could lead to blank values in the sheet.

- **Update Channel Insights**  
  - *Type:* Google Sheets  
  - *Role:* Updates existing rows in the Google Sheet with the new data matching on `Channel ID`.  
  - *Config:* Uses the same Google Sheet and document as trigger node, mapping columns for `Email`, `Channel ID`, `Recent Views`, `Total Subscribers`. Uses OAuth2 credentials.  
  - *Input:* Prepared sheet data from previous node.  
  - *Output:* Confirmation of row update.  
  - *Edge Cases:* Incorrect matching keys, sheet permission errors, or concurrency conflicts.

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                           | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                                            |
|---------------------------|----------------------------|-----------------------------------------|--------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| New Channel Added         | Google Sheets Trigger       | Detect new channel row in Google Sheet  | (Trigger)                | Get info about channel       | Section 1: Triggers workflow on new channel URL addition                                                                |
| Get info about channel    | HTTP Request               | Fetch channel snippet & statistics      | New Channel Added        | Get Channel Stats            | Section 1: Resolves channel ID and basic stats                                                                          |
| Get Channel Stats         | HTTP Request               | Fetch 5 recent video IDs by channel     | Get info about channel   | Get Recent Video Views       | Section 2: Fetches recent video uploads                                                                                  |
| Get Recent Video Views    | HTTP Request               | Get stats for recent videos (views)     | Get Channel Stats        | Prepare views to sum         | Section 2: Fetches individual video view counts                                                                          |
| Prepare views to sum      | Set                        | Extract video views into separate fields| Get Recent Video Views   | Sum Video Views              | Section 2: Prepares data for summation                                                                                   |
| Sum Video Views           | Code                       | Sum view counts of recent videos        | Prepare views to sum     | Get Channel Email (SerpAPI)  | Section 2: Aggregates recent views                                                                                       |
| Get Channel Email (SerpAPI)| HTTP Request              | Scrape public email from channel About  | Sum Video Views          | Prepare Sheet Data           | Section 3: Attempts to get contact email via SerpAPI                                                                     |
| Prepare Sheet Data        | Set                        | Format collected data for sheet update  | Get Channel Email (SerpAPI)| Update Channel Insights     | Section 4: Organizes data for sheet update                                                                                |
| Update Channel Insights   | Google Sheets              | Update Google Sheet row with new data   | Prepare Sheet Data       | (End)                       | Section 4: Updates Google Sheet with enriched channel info                                                               |
| Sticky Note               | Sticky Note                | Documentation and section overview      |                          |                             | Covers Section 1 description                                                                                             |
| Sticky Note1              | Sticky Note                | Documentation and section overview      |                          |                             | Covers Section 2 description                                                                                             |
| Sticky Note2              | Sticky Note                | Documentation and section overview      |                          |                             | Covers Section 3 description                                                                                             |
| Sticky Note3              | Sticky Note                | Documentation and section overview      |                          |                             | Covers Section 4 description                                                                                             |
| Sticky Note4              | Sticky Note                | Full grouped workflow overview           |                          |                             | Overview of all sections                                                                                                 |
| Sticky Note9              | Sticky Note                | Workflow assistance and contact info    |                          |                             | Support contact and resource links                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node - "New Channel Added"**  
   - Type: Google Sheets Trigger  
   - Configure to watch the Google Sheet named “Sheet1” in the document “Youtube Channel Intelligence Collector” (Document ID: `1TpaJAx7eM1XogAZWPSHls4H1AhYsrQ_FTcohkoZyedQ`).  
   - Trigger on event: Row added, polling every minute.  
   - Use Google Sheets OAuth2 credentials.

2. **Add HTTP Request Node - "Get info about channel"**  
   - Type: HTTP Request  
   - URL: `https://www.googleapis.com/youtube/v3/channels?part=statistics,snippet&id=CHANNEL_ID&key=YOUR_KEY`  
   - Replace `CHANNEL_ID` dynamically from trigger data or previous processing (depending on input format).  
   - Use YouTube Data API v3 key in query parameters.

3. **Add HTTP Request Node - "Get Channel Stats"**  
   - Type: HTTP Request  
   - URL: `https://www.googleapis.com/youtube/v3/search`  
   - Query Parameters:  
     - `channelId`: Extracted channel ID from previous node.  
     - `part`: `snippet`  
     - `order`: `date`  
     - `maxResults`: `5`  
     - `key`: Your YouTube API key.  
   - This fetches the 5 most recent videos.

4. **Add HTTP Request Node - "Get Recent Video Views"**  
   - Type: HTTP Request  
   - URL constructed dynamically to fetch statistics for 5 video IDs:  
     `https://www.googleapis.com/youtube/v3/videos?part=statistics&id=VIDEO_ID_1,VIDEO_ID_2,...&key=YOUR_KEY`  
   - Extract video IDs from previous node’s items using expressions.  
   - Replace `YOUR_KEY` with your YouTube API key.

5. **Add Set Node - "Prepare views to sum"**  
   - Type: Set  
   - Assign 5 fields: `Video 1 views` through `Video 5 views` using expressions to map each video’s `statistics.viewCount` from the previous node’s JSON.

6. **Add Code Node - "Sum Video Views"**  
   - Use JavaScript to sum all fields containing “views” into one field `totalViews`.  
   - Return a single JSON object merging original data plus `totalViews`.

7. **Add HTTP Request Node - "Get Channel Email (SerpAPI)"**  
   - Type: HTTP Request  
   - URL:  
     `https://serpapi.com/search.json?engine=youtube_channel_about&channel_url=https://www.youtube.com/channel/CHANNEL_ID&api_key=YOUR_SERPAPI_KEY`  
   - Dynamically insert channel URL from the trigger node data.  
   - Use valid SerpAPI key.

8. **Add Set Node - "Prepare Sheet Data"**  
   - Type: Set  
   - Assign fields:  
     - `Recent Views` = totalViews from Sum Video Views node  
     - `Total Subscribers` = subscriberCount from Get info about channel node  
     - `Email` = email from SerpAPI response node

9. **Add Google Sheets Node - "Update Channel Insights"**  
   - Type: Google Sheets  
   - Operation: Update  
   - Document ID and Sheet Name same as trigger node.  
   - Matching Column: `Channel ID`  
   - Columns to update: `Email`, `Channel ID`, `Recent Views`, `Total Subscribers`  
   - Use Google Sheets OAuth2 credentials.

10. **Connect nodes in this order:**  
    New Channel Added → Get info about channel → Get Channel Stats → Get Recent Video Views → Prepare views to sum → Sum Video Views → Get Channel Email (SerpAPI) → Prepare Sheet Data → Update Channel Insights

11. **Credentials Needed:**  
    - Google Sheets OAuth2 (for trigger and update nodes)  
    - YouTube Data API key (for all YouTube API requests)  
    - SerpAPI API key (for email scraping)

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow assistance and contact email: Yaron@nofluff.online                                  | Contact for support                                                                                     |
| YouTube channel with workflow tips and videos: https://www.youtube.com/@YaronBeen/videos      | Additional learning resources                                                                           |
| LinkedIn profile of creator: https://www.linkedin.com/in/yaronbeen/                           | Professional profile and networking                                                                    |
| Workflow handles YouTube API rate limits and uses SerpAPI for CAPTCHA bypass                  | Important for reliable email scraping                                                                  |
| Uses Google Sheets as live dashboard and data source                                         | Enables easy monitoring and manual edits                                                               |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, a workflow automation tool. This process strictly adheres to current content policies and does not contain any illegal, offensive, or protected elements. All data processed is legal and publicly accessible.