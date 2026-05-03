Youtube Outlier Detector (Find trending content based on your competitors)

https://n8nworkflows.xyz/workflows/youtube-outlier-detector--find-trending-content-based-on-your-competitors--2903


# Youtube Outlier Detector (Find trending content based on your competitors)

### 1. Workflow Overview

This workflow, titled **"Youtube Outlier Detector (Find trending content based on your competitors)"**, automates the process of identifying trending YouTube videos within a niche by detecting outliers—videos that significantly outperform the average views of a competitor channel. It is designed for content creators and marketers to streamline competitor analysis and content research without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**  
  Receives competitor channel IDs and initializes the processing loop.

- **1.2 Historical Data Retrieval and Video Fetching**  
  Queries the PostgreSQL database for the latest stored video publish date per channel and fetches new videos from YouTube API accordingly.

- **1.3 Video Data Enrichment and Filtering**  
  Retrieves detailed video statistics, filters out short videos (e.g., YouTube Shorts), and prepares data for storage.

- **1.4 Data Storage and Management**  
  Structures and inserts new video data into the PostgreSQL database, ensuring efficient incremental updates.

- **1.5 Outlier Detection and Trending Video Identification**  
  Analyzes stored video data to compute channel averages, excludes extreme outliers, and identifies videos published in the last two weeks with views at least twice the channel average.

- **1.6 Output and Reporting**  
  Outputs the best-performing videos with engagement metrics for further analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block receives the list of competitor YouTube channel IDs to process and initiates the workflow loop over these channels.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger, disabled)  
  - `Execute Workflow Trigger` (Execute Workflow Trigger)  
  - `Loop Over Items` (Split In Batches)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual triggering for testing; disabled in production.  
    - Input/Output: No input; outputs predefined competitor channels.  
    - Edge Cases: Disabled node will not trigger unless enabled.

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for automated or scheduled execution.  
    - Input/Output: No input; triggers downstream nodes.  
    - Edge Cases: None specific.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Iterates over each competitor channel item individually for sequential processing.  
    - Configuration: Default batch size (1) to process channels one by one.  
    - Input: List of competitor channels.  
    - Output: Single channel item per iteration.  
    - Edge Cases: Empty input list leads to no iterations.

---

#### 2.2 Historical Data Retrieval and Video Fetching

- **Overview:**  
  For each competitor channel, this block queries the database for the latest stored video publish time and fetches new videos from YouTube published after that date (or last 3 months on first run).

- **Nodes Involved:**  
  - `fetch_last_registered` (PostgreSQL)  
  - `get_videos` (YouTube)  
  - `if_is_empty` (If)

- **Node Details:**

  - **fetch_last_registered**  
    - Type: PostgreSQL  
    - Role: Retrieves the most recent video publish time for the current channel from `video_statistics` table.  
    - Query: `SELECT MAX(publish_time) AS latest_publish_time FROM video_statistics WHERE channel_id = '{{ $json.id }}';`  
    - Input: Current channel ID from loop.  
    - Output: Latest publish time or null if no data.  
    - Edge Cases: Database connection errors, empty results.

  - **get_videos**  
    - Type: YouTube  
    - Role: Fetches up to 50 videos from the channel published after the latest publish time or last 3 months if none.  
    - Configuration:  
      - Channel ID from loop item.  
      - `publishedAfter` dynamically set:  
        - If `latest_publish_time` exists, add 1 hour to avoid duplicates.  
        - Else, 3 months ago from current date.  
      - Region: US, order by relevance, moderate safe search.  
    - Credentials: YouTube OAuth2.  
    - Edge Cases: API quota limits, invalid channel ID, network errors.

  - **if_is_empty**  
    - Type: If  
    - Role: Checks if the fetched videos list is empty or not.  
    - Condition: Checks if input JSON is not empty.  
    - Output:  
      - True branch: Proceed to fetch detailed video data.  
      - False branch: Mark channel as already populated.  
    - Edge Cases: Expression evaluation errors.

---

#### 2.3 Video Data Enrichment and Filtering

- **Overview:**  
  This block enriches each video with detailed statistics, filters out short videos (less than 210 seconds), and prepares data for insertion.

- **Nodes Involved:**  
  - `find_video_data1` (HTTP Request)  
  - `remove_shorts` (Code)  
  - `if_empty` (If)

- **Node Details:**

  - **find_video_data1**  
    - Type: HTTP Request  
    - Role: Calls YouTube Data API to get `contentDetails` and `statistics` for each video ID.  
    - Configuration:  
      - URL: `https://www.googleapis.com/youtube/v3/videos`  
      - Query parameters: API key from environment, video ID from input, parts requested: contentDetails, statistics.  
    - Edge Cases: API key invalid, quota exceeded, missing video ID.

  - **remove_shorts**  
    - Type: Code (JavaScript)  
    - Role: Filters out videos with duration ≤ 210 seconds (shorts).  
    - Logic: Parses ISO8601 duration, converts to seconds, filters accordingly.  
    - Edge Cases: Missing or malformed duration field, logs warnings.

  - **if_empty**  
    - Type: If  
    - Role: Checks if filtered video list is empty after removing shorts.  
    - Condition: Input JSON not empty.  
    - Output:  
      - True: Proceed to map data for insertion.  
      - False: Loop back to next channel item.  
    - Edge Cases: Expression errors.

---

#### 2.4 Data Storage and Management

- **Overview:**  
  Structures video data into database-compatible format and inserts new records into PostgreSQL, enabling incremental data storage.

- **Nodes Involved:**  
  - `map_data` (Set)  
  - `structure_data` (Code)  
  - `create_query` (Code)  
  - `insert_items` (PostgreSQL)  
  - `Loop Over Items` (Split In Batches, loop continuation)

- **Node Details:**

  - **map_data**  
    - Type: Set  
    - Role: Extracts and renames relevant video fields (id, viewCount, likeCount, commentCount, publishTime, channelId) for further processing.  
    - Expressions: Uses data from `get_videos` and `if_is_empty` nodes.  
    - Edge Cases: Missing fields in input JSON.

  - **structure_data**  
    - Type: Code (JavaScript)  
    - Role: Cleans data by replacing null likeCount and commentCount with "0" strings, filters out videos with null viewCount.  
    - Edge Cases: Null or missing fields.

  - **create_query**  
    - Type: Code (JavaScript)  
    - Role: Dynamically builds parameterized SQL INSERT query for batch insertion into `video_statistics` table.  
    - Logic:  
      - Maps video data to columns: id, view_count, like_count, comment_count, publish_time, channel_id.  
      - Creates placeholders for prepared statement parameters.  
    - Output: Query string and parameters array.  
    - Edge Cases: Empty input array, SQL injection mitigated by parameterization.

  - **insert_items**  
    - Type: PostgreSQL  
    - Role: Executes the INSERT query with parameters to store video data.  
    - Credentials: PostgreSQL account.  
    - Edge Cases: Duplicate primary key errors, connection issues.

  - **Loop Over Items** (second usage)  
    - Role: Continues looping over channels after insertion or skips if already populated.  
    - Edge Cases: Infinite loops if not properly configured.

---

#### 2.5 Outlier Detection and Trending Video Identification

- **Overview:**  
  Queries the database to compute average views per channel excluding top and bottom 2 videos (unless ≤10 videos), then identifies videos published within last 2 weeks with views ≥ 2× average.

- **Nodes Involved:**  
  - `Postgres` (PostgreSQL)  
  - `sanitize_data` (Code)

- **Node Details:**

  - **Postgres**  
    - Type: PostgreSQL  
    - Role: Runs a complex SQL query that:  
      - Ranks videos by view count per channel.  
      - Excludes top 2 and bottom 2 videos to avoid skewing averages (unless channel has ≤10 videos).  
      - Calculates average views per channel.  
      - Aggregates video data per channel with average views.  
    - Output: JSON with channel averages and video lists.  
    - Edge Cases: Large dataset performance, query errors.

  - **sanitize_data**  
    - Type: Code (JavaScript)  
    - Role: Filters videos published within last 14 days and with views at least twice the channel average.  
    - Calculates a "score" as (like_count / view_count) * 100 for engagement metric.  
    - Outputs array of best-performing videos with relevant metadata and URLs.  
    - Edge Cases: Date parsing errors, division by zero if view_count is zero.

---

#### 2.6 Output and Reporting

- **Overview:**  
  Final output of the workflow is the list of trending videos with engagement metrics, ready for analysis or further processing.

- **Nodes Involved:**  
  - Output from `sanitize_data` node (final node in flow)

- **Node Details:**

  - Outputs an array of objects containing:  
    - Video ID and URL  
    - View count  
    - Like count  
    - Comment count  
    - Engagement score  
    - Channel URL  
  - Edge Cases: Empty output if no trending videos detected.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                                | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                      |
|----------------------------|-------------------------|-----------------------------------------------|-------------------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Manual start for testing                       | -                             | Loop Over Items               |                                                                                                 |
| Execute Workflow Trigger    | Execute Workflow Trigger | Automated workflow start                       | -                             | Postgres                     |                                                                                                 |
| Loop Over Items            | Split In Batches         | Iterates over competitor channels              | When clicking ‘Test workflow’ / insert_items / already_populated | fetch_last_registered / fetch_last_registered |                                                                                                 |
| fetch_last_registered       | PostgreSQL              | Fetch latest stored video publish time per channel | Loop Over Items               | get_videos                   |                                                                                                 |
| get_videos                 | YouTube                 | Fetch new videos published after last stored date | fetch_last_registered         | if_is_empty                  |                                                                                                 |
| if_is_empty                | If                      | Checks if new videos were fetched              | get_videos                    | find_video_data1 / already_populated |                                                                                                 |
| find_video_data1           | HTTP Request            | Fetch detailed video statistics                 | if_is_empty                   | remove_shorts                |                                                                                                 |
| remove_shorts              | Code                    | Filter out short videos (<210 seconds)          | find_video_data1              | if_empty                    |                                                                                                 |
| if_empty                   | If                      | Check if filtered videos list is empty          | remove_shorts                 | map_data / Loop Over Items   |                                                                                                 |
| map_data                   | Set                     | Map and rename video fields for DB insertion    | if_empty                     | structure_data               |                                                                                                 |
| structure_data             | Code                    | Clean and normalize video data                   | map_data                     | create_query                 |                                                                                                 |
| create_query               | Code                    | Build parameterized SQL INSERT query             | structure_data               | insert_items                 |                                                                                                 |
| insert_items               | PostgreSQL              | Insert new video data into database               | create_query                 | Loop Over Items              |                                                                                                 |
| already_populated          | Set                     | Mark channel as already populated if no new videos | if_is_empty (false branch)   | Loop Over Items              |                                                                                                 |
| Postgres                  | PostgreSQL              | Query average views and aggregate video data     | Execute Workflow Trigger      | sanitize_data               |                                                                                                 |
| sanitize_data              | Code                    | Identify trending videos based on outlier criteria | Postgres                    | (Workflow Output)            |                                                                                                 |
| create_table               | PostgreSQL              | Create `video_statistics` table (setup)           | -                           | -                            |                                                                                                 |
| drop table                | PostgreSQL              | Drop `video_statistics` table (cleanup)           | -                           | -                            |                                                                                                 |
| see table                 | PostgreSQL              | View all records in `video_statistics` table      | -                           | -                            |                                                                                                 |
| Sticky Note               | Sticky Note             | "Save Videos To Database" (covers storage block)  | -                           | -                            |                                                                                                 |
| Sticky Note1              | Sticky Note             | "Fetch best performing videos from last 2 weeks" | -                           | -                            |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create PostgreSQL Table (One-time Setup):**  
   - Add a PostgreSQL node named `create_table`.  
   - Configure with your PostgreSQL credentials.  
   - Use the query:  
     ```sql
     CREATE TABLE video_statistics (
       id VARCHAR(255) PRIMARY KEY,
       view_count INT NOT NULL,
       like_count INT NOT NULL,
       comment_count INT NOT NULL,
       publish_time TIMESTAMP NOT NULL,
       channel_id VARCHAR(255) NOT NULL
     );
     ```  
   - Execute once to create the table.

2. **Set Up Workflow Trigger:**  
   - Add an `Execute Workflow Trigger` node as the entry point for scheduled or automated runs.

3. **Input Competitor Channels:**  
   - Add a `Manual Trigger` node named `When clicking ‘Test workflow’` (optional for manual testing).  
   - Set its output data to include competitor channel IDs and URLs as JSON objects.

4. **Loop Over Competitor Channels:**  
   - Add a `Split In Batches` node named `Loop Over Items`.  
   - Connect it to the trigger node(s).  
   - This node will process each channel individually.

5. **Fetch Latest Stored Video Publish Time:**  
   - Add a PostgreSQL node named `fetch_last_registered`.  
   - Query:  
     ```sql
     SELECT MAX(publish_time) AS latest_publish_time
     FROM video_statistics
     WHERE channel_id = '{{ $json.id }}';
     ```  
   - Connect from `Loop Over Items`.

6. **Fetch New Videos from YouTube:**  
   - Add a YouTube node named `get_videos`.  
   - Configure with YouTube OAuth2 credentials.  
   - Set resource to `video`.  
   - Set filters:  
     - `channelId` from current item (`={{ $('Loop Over Items').item.json.id }}`)  
     - `publishedAfter` dynamically:  
       ```js
       {{$json.latest_publish_time ? new Date(new Date($json.latest_publish_time).getTime() + 3600000).toISOString() : new Date(Date.now() - 3 * 30 * 24 * 60 * 60 * 1000).toISOString()}}
       ```  
   - Limit to 50 videos, regionCode US, order relevance, safeSearch moderate.

7. **Check If Videos Were Fetched:**  
   - Add an `If` node named `if_is_empty`.  
   - Condition: Check if input JSON is not empty (not empty object).  
   - True branch proceeds to fetch detailed video data; False branch marks channel as already populated.

8. **Fetch Detailed Video Data:**  
   - Add an HTTP Request node named `find_video_data1`.  
   - URL: `https://www.googleapis.com/youtube/v3/videos`  
   - Query parameters:  
     - `key`: `={{ $env["GOOGLE_API_KEY"] }}`  
     - `id`: `={{ $json.id.videoId }}`  
     - `part`: `contentDetails,statistics`  
   - Connect from `if_is_empty` true branch.

9. **Filter Out Shorts:**  
   - Add a Code node named `remove_shorts`.  
   - Paste the provided JavaScript code that parses ISO8601 durations and filters videos longer than 210 seconds.  
   - Connect from `find_video_data1`.

10. **Check If Filtered Videos Exist:**  
    - Add an `If` node named `if_empty`.  
    - Condition: Input JSON not empty.  
    - True branch proceeds to map data; False branch loops back to next channel.

11. **Map Video Data for DB:**  
    - Add a Set node named `map_data`.  
    - Map fields:  
      - `id` = `={{ $json.items[0].id }}`  
      - `viewCount` = `={{ $json.items[0].statistics.viewCount }}`  
      - `likeCount` = `={{ $json.items[0].statistics.likeCount }}`  
      - `commentCount` = `={{ $json.items[0].statistics.commentCount }}`  
      - `publishTime` = `={{ $('get_videos').item.json.snippet.publishedAt }}`  
      - `channelId` = `={{ $('if_is_empty').item.json.snippet.channelId }}`

12. **Clean and Normalize Data:**  
    - Add a Code node named `structure_data`.  
    - Use provided JavaScript to replace null likeCount/commentCount with "0" and filter out null viewCount.

13. **Create Parameterized Insert Query:**  
    - Add a Code node named `create_query`.  
    - Use provided JavaScript to build a parameterized SQL INSERT statement for batch insertion.

14. **Insert Data into PostgreSQL:**  
    - Add a PostgreSQL node named `insert_items`.  
    - Use credentials and execute the query from `create_query`.

15. **Loop Back to Next Channel:**  
    - Connect `insert_items` back to `Loop Over Items` to continue processing.

16. **Handle Already Populated Channels:**  
    - Add a Set node named `already_populated` to log/report channels with no new videos.  
    - Connect from `if_is_empty` false branch back to `Loop Over Items`.

17. **Outlier Detection Query:**  
    - Add a PostgreSQL node named `Postgres`.  
    - Use the provided complex SQL query to compute average views excluding top/bottom 2 videos and aggregate data.

18. **Identify Trending Videos:**  
    - Add a Code node named `sanitize_data`.  
    - Use provided JavaScript to filter videos published within last 14 days and with views ≥ 2× average, calculate engagement score.

19. **Final Output:**  
    - The output of `sanitize_data` is the list of trending videos with metrics.

20. **Optional Maintenance Nodes:**  
    - Add PostgreSQL nodes `drop table` and `see table` for database management and inspection.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Video explanation of the workflow is available at: [YouTube Video](https://www.youtube.com/watch?v=pIfT9e-zPO0) | Provides detailed walkthrough and use cases for this workflow.                                  |
| Workflow requires valid YouTube Data API key and PostgreSQL database access.                        | Ensure credentials are configured in n8n for YouTube OAuth2 and PostgreSQL nodes.                |
| The workflow filters out YouTube Shorts by excluding videos shorter than 210 seconds.               | This threshold can be adjusted in the `remove_shorts` code node if needed.                       |
| Outlier detection excludes top 2 and bottom 2 videos per channel to avoid skewing averages.         | This logic is implemented in the SQL query within the `Postgres` node.                           |
| The workflow is designed for educational and research purposes to aid content creators.             | Not intended for commercial use without modification or compliance with YouTube API terms.      |

---

This structured documentation provides a comprehensive understanding of the workflow’s logic, node configurations, and data flow, enabling replication, modification, and troubleshooting by advanced users or AI agents.