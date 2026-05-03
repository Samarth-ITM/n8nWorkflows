Find Quality YouTube Videos with Automated Filtering & Relevance Scoring to Google Sheets

https://n8nworkflows.xyz/workflows/find-quality-youtube-videos-with-automated-filtering---relevance-scoring-to-google-sheets-5797


# Find Quality YouTube Videos with Automated Filtering & Relevance Scoring to Google Sheets

### 1. Workflow Overview

This workflow automates the discovery of high-quality, educational YouTube videos related to AI and prompt engineering topics, filtering and scoring them by relevance before exporting the results to a Google Sheets document for easy review and tracking.

The workflow is logically divided into the following main blocks:

- **1.1 Input Setup:** Define search queries that specify the video topics to look for.
- **1.2 YouTube Search:** Query the YouTube Data API v3 for videos matching the search queries, restricted to the Education category and sorted by relevance.
- **1.3 Initial Filtering:** Filter video search results by title keywords to exclude promotional or scammy content.
- **1.4 Metadata Enrichment & Quality Filtering:** Retrieve detailed metadata (views, likes, publish date) for each video, then apply quality filters based on engagement and recency.
- **1.5 Deduplication and Scoring:** Remove duplicate entries and calculate a relevance score based on keywords and quality metrics.
- **1.6 Sorting, Limiting & Export:** Sort videos by relevance score, limit the number of entries, and append the final curated list to a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup

- **Overview:**  
  Sets the list of search queries to use for YouTube video searching. This block facilitates easy modification of search terms.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Set Query  
  - Split Query  
  - Sticky Note1

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually  
    - Configuration: Default, no parameters  
    - Input: None  
    - Output: Connected to Set Query  
    - Edge cases: None typical, manual start only

  - **Set Query**  
    - Type: Set  
    - Role: Assigns an array of search queries under a variable named `query`  
    - Configuration: The `query` is an array of strings such as "AI agents tutorial step by step", "AI tools tutorial for beginners", etc.  
    - Key variables: `query` array  
    - Input: Manual Trigger node output  
    - Output: Connected to Split Query  
    - Edge cases: Empty or malformed array may lead to no search results

  - **Split Query**  
    - Type: Split Out  
    - Role: Splits the `query` array into individual items to be processed separately  
    - Configuration: Splits by the `query` field  
    - Input: Set Query output  
    - Output: Connected to Search YouTube node  
    - Edge cases: Large query arrays may cause delays or API rate limits

  - **Sticky Note1**  
    - Content: Explains that this node defines search queries for YouTube and suggests modifying the queries as needed.  
    - Positioned near Set Query node.

#### 2.2 YouTube Search

- **Overview:**  
  Queries the YouTube Data API for videos matching each search term, restricted to the Education category (category ID 27), returning up to 50 videos sorted by relevance.

- **Nodes Involved:**  
  - Search YouTube  
  - Split Results  
  - Sticky Note

- **Node Details:**

  - **Search YouTube**  
    - Type: HTTP Request  
    - Role: Calls YouTube Data API’s search endpoint to retrieve video results for each query  
    - Configuration:  
      - URL: https://www.googleapis.com/youtube/v3/search  
      - Query Parameters: `part=id,snippet`, `q` is dynamic from current query, `type=video`, `maxResults=50`, `order=relevance`, `videoCategoryId=27` (Education)  
      - Authentication: Uses YouTube OAuth2 credentials configured in n8n workspace  
    - Input: Split Query output (single query string)  
    - Output: Connected to Split Results  
    - Edge cases: API quota limits, invalid credentials, network timeouts  
    - Sticky Note (attached to this block): Instructions on enabling YouTube Data API v3, adding credentials, and modifying node credentials.

  - **Split Results**  
    - Type: Split Out  
    - Role: Splits the array of video search results (`items`) into individual items for processing  
    - Configuration: Splits on the `items` field  
    - Input: Search YouTube node output  
    - Output: Connected to Filter for Relevance node  
    - Edge cases: Empty `items` array leads to no further processing

  - **Sticky Note**  
    - Content: Explains the YouTube search API call and setup instructions for credentials and API enablement.

#### 2.3 Initial Filtering by Title Relevance

- **Overview:**  
  Filters videos based on title keywords to include educational/tutorial content and exclude promotional or scam-related terms.

- **Nodes Involved:**  
  - Filter for Relevance  
  - Sticky Note3

- **Node Details:**

  - **Filter for Relevance**  
    - Type: Filter  
    - Role: Applies multiple regex-based string conditions on the video title (lowercased) to filter in educational content and filter out promotional/scammy content  
    - Configuration:  
      - Includes keywords like "tutorial", "course", "guide", "training", "learn", "education", "explained", "masterclass", "step by step", "beginners guide", "how to build", "fundamentals"  
      - Excludes keywords like "money", "rich", "profit", "earn", "cash", "income", "$", "sell", "agency", "business", "client", "freelance", "side hustle", "make money"  
      - Additional exclusions for "autopilot", "passive income", "subs", "faceless videos", "automation money", "100% automated money"  
      - Also excludes sensational terms like "insane", "crazy", "ultimate secret", "hack", "trick", "must do", "steal these", "best way to make", "ways to make", "you need to", and numbered "ways" or "insane"  
    - Input: Split Results output (single video item)  
    - Output: Connected to Loop over Items  
    - Edge cases: Overly strict filters may exclude good content; regex failures if title is missing  

  - **Sticky Note3**  
    - Content: Describes this filter as the first quality and relevance gate excluding non-educational or scammy videos, encourages modifying the criteria.

#### 2.4 Metadata Enrichment & Quality Filtering Loop

- **Overview:**  
  For each filtered video, fetch detailed metadata such as view and like counts. Filter further based on minimum views, likes, and publish date, then remove unnecessary fields before further processing.

- **Nodes Involved:**  
  - Loop over Items  
  - Get Video Metadata  
  - Filter for Quality  
  - Remove Unnecessary Fields  
  - Sticky Note2

- **Node Details:**

  - **Loop over Items**  
    - Type: Split In Batches  
    - Role: Processes videos in batches of 10 for efficiency and rate limiting  
    - Configuration: Batch size 10, executes once per batch  
    - Input: Filter for Relevance output  
    - Output: Two outputs  
      - Main (index 0): To Remove Duplicate Videos (deduplication branch)  
      - Main (index 1): To Get Video Metadata (metadata enrichment branch)  
    - Edge cases: Large datasets may cause delays or API throttling

  - **Get Video Metadata**  
    - Type: HTTP Request  
    - Role: Calls YouTube Data API v3 videos endpoint to get detailed snippet and statistics for each video ID  
    - Configuration:  
      - URL: https://www.googleapis.com/youtube/v3/videos  
      - Query Params: part=snippet,statistics; id=videoId from current JSON  
      - Authentication: YouTube OAuth2 credentials  
    - Input: Loop over Items output (video item)  
    - Output: Connected to Filter for Quality  
    - Edge cases: API quota, missing video ID, network errors

  - **Filter for Quality**  
    - Type: Filter  
    - Role: Filters videos based on:  
      - Views > 10,000  
      - Likes > 100  
      - Published after 1 Dec 2023  
    - Configuration: Numeric and datetime comparisons on metadata fields  
    - Input: Get Video Metadata output  
    - Output: Connected to Remove Unnecessary Fields  
    - Edge cases: Videos without stats or publish date may be excluded

  - **Remove Unnecessary Fields**  
    - Type: Set  
    - Role: Simplifies the JSON output to include only essential fields: Title, Channel, Published At, Views, Likes, Description, URL  
    - Configuration: Extracts these fields from the nested JSON structure, builds URL from video ID  
    - Input: Filter for Quality output  
    - Output: Connected to Loop over Items (back to continue processing)  
    - Edge cases: Missing fields in JSON may result in empty or incomplete data

  - **Sticky Note2**  
    - Content: Explains the metadata enrichment loop, filtering criteria, and encourages modifying filters for quality or relevance. Also reminds to update YouTube credentials in the Get Video Metadata node.

#### 2.5 Deduplication and Scoring

- **Overview:**  
  Removes duplicate videos by URL, calculates a relevance score based on keyword presence in the title and quality indicators, applying penalties for monetization-related terms.

- **Nodes Involved:**  
  - Remove Duplicate Videos  
  - Generate Relevance Score  
  - Sticky Note5

- **Node Details:**

  - **Remove Duplicate Videos**  
    - Type: Remove Duplicates  
    - Role: Removes duplicate entries comparing the `URL` field to ensure unique videos  
    - Configuration: Compare by URL string  
    - Input: Loop over Items output (deduplication branch)  
    - Output: Connected to Generate Relevance Score  
    - Edge cases: Different URLs for same content are not deduplicated; missing URLs cause issues

  - **Generate Relevance Score**  
    - Type: Set  
    - Role: Calculates a numeric `RelevanceScore` based on presence of educational and AI-specific keywords in the title, quality metrics (views, likes), and negative weighting for monetization keywords  
    - Configuration:  
      - Adds points for keywords like "tutorial", "course", "guide", "explained", "beginner", "step by step", "complete"  
      - AI terms include "prompt engineering", "ai agent", "ai tools", "prompting" with higher weights  
      - Adds points for views and likes thresholds (>100k views, >50k views, >1000 likes)  
      - Deducts points for presence of "money", "rich", "earn" in title  
      - Also copies over all relevant fields for downstream use  
    - Input: Remove Duplicate Videos output  
    - Output: Connected to Sort by Relevance  
    - Edge cases: String methods may fail if any field missing; parseInt failures if views/likes are undefined

  - **Sticky Note5**  
    - Content: Describes this final step of deduplication, scoring, and sending to Google Sheets. Instructions to setup Google Sheet columns and credentials, and suggests modifying the scoring formula.

#### 2.6 Sorting, Limiting & Export to Google Sheets

- **Overview:**  
  Sorts videos by the computed relevance score in descending order, limits to top 50 entries, waits briefly, and appends results to a Google Sheets document.

- **Nodes Involved:**  
  - Sort by Relevance  
  - Force Limit  
  - Wait  
  - Send to Google Sheets

- **Node Details:**

  - **Sort by Relevance**  
    - Type: Sort  
    - Role: Sorts the dataset descending by the `RelevanceScore` field  
    - Configuration: Sort by field `RelevanceScore` descending  
    - Input: Generate Relevance Score output  
    - Output: Connected to Force Limit  
    - Edge cases: Missing relevance scores cause sorting issues

  - **Force Limit**  
    - Type: Limit  
    - Role: Limits the number of items to maximum 50  
    - Configuration: maxItems = 50  
    - Input: Sort by Relevance output  
    - Output: Connected to Wait  
    - Edge cases: If fewer than 50 items, passes all through

  - **Wait**  
    - Type: Wait  
    - Role: Waits 3 seconds to ensure data readiness or API limits  
    - Configuration: Amount = 3 seconds  
    - Input: Force Limit output  
    - Output: Connected to Send to Google Sheets  
    - Edge cases: Minimal, just a delay

  - **Send to Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends the final curated video data to a Google Sheet with columns Title, Channel, Published At, Views, Likes, Description, URL  
    - Configuration:  
      - Mapping explicitly defines field mappings for each column  
      - Operation: append  
      - Document ID and Sheet Name configured to a specific Google Sheet  
      - Uses Google Sheets OAuth2 credentials  
    - Input: Wait output  
    - Output: None (terminal node)  
    - Edge cases: Credential issues, API quota, sheet not existing or misconfigured columns

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                                 | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                                                             |
|----------------------------|------------------------|------------------------------------------------|--------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Starts the workflow                             | None                           | Set Query                    |                                                                                                                                         |
| Set Query                  | Set                    | Defines array of search queries                  | When clicking ‘Execute workflow’ | Split Query                  | Explains query setup and encourages modification                                                                                         |
| Split Query                | Split Out              | Splits query array into individual queries      | Set Query                     | Search YouTube               |                                                                                                                                         |
| Search YouTube             | HTTP Request           | Calls YouTube API to search videos               | Split Query                   | Split Results                | Explains API setup, enabling YouTube Data API, credentials setup, and node configuration instructions                                    |
| Split Results              | Split Out              | Splits the array of video results into items    | Search YouTube                | Filter for Relevance         |                                                                                                                                         |
| Filter for Relevance       | Filter                 | Filters videos by title keywords for relevance  | Split Results                 | Loop over Items              | Describes exclusion of promotional/scammy content and encourages filter modification                                                    |
| Loop over Items            | Split In Batches       | Processes videos in batches for metadata enrichment and deduplication | Filter for Relevance          | Remove Duplicate Videos, Get Video Metadata |                                                                                                                                         |
| Get Video Metadata         | HTTP Request           | Fetches detailed metadata for each video        | Loop over Items               | Filter for Quality           | Instructs updating YouTube credentials and modifying filtering criteria                                                                  |
| Filter for Quality         | Filter                 | Filters videos by views, likes, and publish date | Get Video Metadata            | Remove Unnecessary Fields    |                                                                                                                                         |
| Remove Unnecessary Fields  | Set                    | Keeps only essential video data fields           | Filter for Quality            | Loop over Items              |                                                                                                                                         |
| Remove Duplicate Videos    | Remove Duplicates      | Removes duplicate video entries by URL           | Loop over Items               | Generate Relevance Score     | Explains deduplication and scoring before Google Sheets export                                                                           |
| Generate Relevance Score   | Set                    | Calculates numeric relevance score and copies fields | Remove Duplicate Videos       | Sort by Relevance           | Details scoring formula and suggests modification                                                                                        |
| Sort by Relevance          | Sort                   | Sorts videos descending by relevance score       | Generate Relevance Score       | Force Limit                 |                                                                                                                                         |
| Force Limit                | Limit                  | Limits to top 50 videos                           | Sort by Relevance             | Wait                        |                                                                                                                                         |
| Wait                      | Wait                   | Delays for 3 seconds                             | Force Limit                  | Send to Google Sheets       |                                                                                                                                         |
| Send to Google Sheets      | Google Sheets          | Appends curated video data to Google Sheet       | Wait                         | None                        | Instructs Google Sheet creation, credential setup, and mapping configuration                                                            |
| Sticky Note                | Sticky Note            | Multiple notes explaining API setup, filtering, scoring, and export steps | Various                      | None                        | Covers API enablement, query setup, filtering criteria, scoring formula, and Google Sheets setup with links to documentation              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node named "When clicking ‘Execute workflow’".**  
   - No parameters needed. This node will start the workflow manually.

2. **Add a Set node named "Set Query".**  
   - Create a field `query` of type array.  
   - Assign it an array of search strings, for example:  
     - "AI agents tutorial step by step"  
     - "AI tools tutorial for beginners"  
     - "AI agent building tutorial"  
     - "AI prompt engineering techniques"  
     - "best AI tools training"  
     - "prompt engineering best practices"  
   - Connect Manual Trigger output to this node.

3. **Add a Split Out node named "Split Query".**  
   - Set `fieldToSplitOut` to `query`.  
   - Connect "Set Query" output to this node.

4. **Add an HTTP Request node named "Search YouTube".**  
   - Method: GET  
   - URL: https://www.googleapis.com/youtube/v3/search  
   - Query Parameters:  
     - part: id,snippet  
     - q: Expression `{{$json.query}}` (current query item)  
     - type: video  
     - maxResults: 50  
     - order: relevance  
     - videoCategoryId: 27 (Education)  
   - Authentication: Use YouTube OAuth2 credentials configured in n8n.  
   - Connect "Split Query" output to this node.

5. **Add a Split Out node named "Split Results".**  
   - Set `fieldToSplitOut` to `items`.  
   - Connect "Search YouTube" output to this node.

6. **Add a Filter node named "Filter for Relevance".**  
   - Add multiple string regex conditions on `{{$json.snippet.title.toLowerCase()}}` for including educational keywords and excluding promotional/scammy keywords.  
   - Use "and" combinator.  
   - Connect "Split Results" output to this node.

7. **Add a Split In Batches node named "Loop over Items".**  
   - Batch size: 10  
   - Connect "Filter for Relevance" output to this node.

8. **Add an HTTP Request node named "Get Video Metadata".**  
   - Method: GET  
   - URL: https://www.googleapis.com/youtube/v3/videos  
   - Query Parameters:  
     - part: snippet,statistics  
     - id: Expression `{{$json.id.videoId}}` from item  
   - Authentication: Use YouTube OAuth2 credentials.  
   - Connect "Loop over Items" output index 1 to this node.

9. **Add a Filter node named "Filter for Quality".**  
   - Conditions:  
     - `{{$json.items[0].statistics.viewCount}}` > 10,000  
     - `{{$json.items[0].statistics.likeCount}}` > 100  
     - `{{$json.items[0].snippet.publishedAt}}` after 2023-12-01T00:00:00  
   - Connect "Get Video Metadata" output to this node.

10. **Add a Set node named "Remove Unnecessary Fields".**  
    - Assign fields for output:  
      - Title: `{{$json.items[0].snippet.title}}`  
      - Channel: `{{$json.items[0].snippet.channelTitle}}`  
      - Published At: `{{$json.items[0].snippet.publishedAt}}`  
      - Views: `{{$json.items[0].statistics.viewCount}}`  
      - Likes: `{{$json.items[0].statistics.likeCount}}`  
      - Description: `{{$json.items[0].snippet.description}}`  
      - URL: `https://www.youtube.com/watch?v={{$json.items[0].id}}`  
    - Connect "Filter for Quality" output to this node.

11. **Connect "Remove Unnecessary Fields" output back to "Loop over Items" input index 0.**  
    - This creates a loop for processing enriched items separately.

12. **Connect "Loop over Items" output index 0 to a Remove Duplicates node named "Remove Duplicate Videos".**  
    - Compare by `{{$json.URL}}`.  

13. **Add a Set node named "Generate Relevance Score".**  
    - Calculate `RelevanceScore` using expressions:  
      - Add points for presence of keywords (“tutorial”, “course”, etc.) in Title  
      - Add points for views and likes thresholds  
      - Subtract points if Title contains monetization keywords ("money", "rich", "earn")  
    - Copy all other relevant fields to output.  
    - Connect "Remove Duplicate Videos" output to this node.

14. **Add a Sort node named "Sort by Relevance".**  
    - Sort by field: `RelevanceScore` descending.  
    - Connect "Generate Relevance Score" output to this node.

15. **Add a Limit node named "Force Limit".**  
    - Max items: 50  
    - Connect "Sort by Relevance" output to this node.

16. **Add a Wait node named "Wait".**  
    - Delay: 3 seconds  
    - Connect "Force Limit" output to this node.

17. **Add a Google Sheets node named "Send to Google Sheets".**  
    - Operation: Append  
    - Document ID: Your target Google Sheet ID  
    - Sheet Name: e.g., "Sheet1" or gid=0  
    - Map columns: Title, Channel, Published At, Views, Likes, Description, URL to corresponding fields from JSON  
    - Use Google Sheets OAuth2 credentials.  
    - Connect "Wait" output to this node.

18. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Enable the YouTube Data API v3 in Google Cloud Console before running the workflow.                                                                                                                                               | https://console.cloud.google.com/apis/library/youtube.googleapis.com                                                          |
| Register and configure your YouTube OAuth2 credentials to use with the workflow nodes.                                                                                                                                           | https://developers.google.com/youtube/registering_an_application                                                             |
| Create a Google Sheet with columns: Title, Channel, Published At, Views, Likes, Description, URL for this workflow to append data properly.                                                                                      | Google Sheets setup                                                                                                           |
| Modify filter criteria and relevance scoring formula to better fit your content criteria or quality standards.                                                                                                                    | In nodes "Filter for Relevance", "Filter for Quality", and "Generate Relevance Score"                                         |
| This workflow processes batch requests and includes delay to avoid API rate limits and ensure data integrity.                                                                                                                    | Node: Loop over Items and Wait node                                                                                          |
| The workflow is designed to run manually, allowing you to update search queries or credentials before execution.                                                                                                                  | Manual Trigger node                                                                                                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.