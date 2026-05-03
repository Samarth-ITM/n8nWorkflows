🚀 YouTube Comment Sentiment Analyzer with Google Sheets & OpenAI

https://n8nworkflows.xyz/workflows/---youtube-comment-sentiment-analyzer-with-google-sheets---openai-3855


# 🚀 YouTube Comment Sentiment Analyzer with Google Sheets & OpenAI

### 1. Workflow Overview

This workflow automates the process of fetching YouTube video comments, analyzing their sentiment, and storing the results in Google Sheets. It is designed for influencers, marketers, and data teams who want quick, automated insights into audience sentiment without manual data handling.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Reads YouTube video URLs and fetch scheduling metadata from a Google Sheet.
- **1.2 Fetch Scheduling Validation:** Checks if the workflow should proceed based on the next scheduled fetch time.
- **1.3 Comment Retrieval:** Fetches all top-level comments for each video URL from the YouTube Data API, handling pagination.
- **1.4 Sentiment Analysis:** Uses OpenAI to classify each comment’s sentiment as Positive, Neutral, or Negative.
- **1.5 Data Formatting:** Prepares comment data and sentiment results for Google Sheets storage.
- **1.6 Data Storage:** Inserts or updates comment data and sentiment into a Google Sheet, and updates fetch timestamps.
- **1.7 Workflow Trigger and Control:** Manual trigger to start the workflow and no-operation nodes to handle conditional branches gracefully.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Reads the list of YouTube video URLs and associated metadata (last and next fetch times) from a Google Sheet to drive the workflow.

- **Nodes Involved:**  
  - Get Video Urls from Google Sheet

- **Node Details:**  
  - **Get Video Urls from Google Sheet**  
    - Type: Google Sheets node (read operation)  
    - Configuration: Reads from a specific Google Sheet document and sheet tab containing `video_urls`, `last_fetched_time`, and `next_fetch_time` columns. Uses a service account for authentication.  
    - Inputs: Triggered by Manual Trigger node.  
    - Outputs: JSON items with video URLs and fetch metadata.  
    - Edge Cases: Possible failures include authentication errors, sheet access issues, or empty data.  
    - Version: 4.5

#### 1.2 Fetch Scheduling Validation

- **Overview:**  
  Determines whether the workflow should proceed with fetching comments based on the `next_fetch_time` field relative to the current time.

- **Nodes Involved:**  
  - check next fetch time is available or not (IF)  
  - check next fetch time is before the current time (IF)  
  - No Operation, do nothing  
  - No Operation, do nothing1

- **Node Details:**  
  - **check next fetch time is available or not**  
    - Type: IF node  
    - Configuration: Checks if `next_fetch_time` is empty (no scheduled fetch). If empty, proceeds to fetch comments. Otherwise, passes to next IF node.  
    - Inputs: From "Get Video Urls from Google Sheet"  
    - Outputs: True branch triggers comment fetch; False branch goes to next IF node.  
    - Edge Cases: Expression failures if `next_fetch_time` is missing or malformed.  
    - Version: 2.2

  - **check next fetch time is before the current time**  
    - Type: IF node  
    - Configuration: Checks if `next_fetch_time` is before current time. If yes, proceeds to fetch comments; else, does nothing.  
    - Inputs: From previous IF node false branch  
    - Outputs: True branch triggers comment fetch; False branch leads to No Operation node.  
    - Edge Cases: Date parsing errors, timezone issues.  
    - Version: 2.2

  - **No Operation, do nothing** and **No Operation, do nothing1**  
    - Type: NoOp nodes  
    - Role: End branches where no action is needed, preventing workflow errors or hanging executions.  
    - Inputs: From IF nodes when conditions are not met.  
    - Outputs: None  
    - Version: 1

#### 1.3 Comment Retrieval

- **Overview:**  
  Fetches all top-level comments for each video URL from the YouTube Data API, handling pagination (up to 100 comments per page).

- **Nodes Involved:**  
  - Get Comments for video urls  
  - Check Success Response  
  - Split Out

- **Node Details:**  
  - **Get Comments for video urls**  
    - Type: HTTP Request node  
    - Configuration:  
      - URL: YouTube Data API v3 endpoint for commentThreads  
      - Query parameters: `part=snippet`, `videoId` extracted via regex from video URL, `maxResults=100`  
      - Authentication: HTTP Query Auth with API key for YouTube Data API  
      - Pagination: Uses `pageToken` parameter to fetch all pages until no nextPageToken is returned  
    - Inputs: From IF nodes that validate fetch timing  
    - Outputs: Full API response with comments data  
    - Edge Cases: API quota limits, invalid video IDs, network failures, malformed URLs, empty comment threads  
    - Version: 4.2

  - **Check Success Response**  
    - Type: IF node  
    - Configuration: Checks if HTTP response status code equals 200 (success)  
    - Inputs: From HTTP Request node  
    - Outputs: True branch proceeds to Split Out; False branch to No Operation node  
    - Edge Cases: API errors, rate limiting, unexpected status codes  
    - Version: 2.2

  - **Split Out**  
    - Type: SplitOut node  
    - Configuration: Splits the array of comments (`body.items`) into individual items for processing one by one  
    - Inputs: From IF node on success  
    - Outputs: Each comment item individually to sentiment analysis  
    - Edge Cases: Empty comment arrays, malformed data  
    - Version: 1

#### 1.4 Sentiment Analysis

- **Overview:**  
  Analyzes the sentiment of each individual comment using OpenAI’s GPT-4o-mini model, categorizing comments as Positive, Neutral, or Negative.

- **Nodes Involved:**  
  - OpenAI Chat Model (configured but not directly connected in this workflow)  
  - Analyze sentiment of every comment

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node  
    - Configuration: Uses GPT-4o-mini model, no special options  
    - Credentials: OpenAI API key  
    - Role: Available for AI language model calls, linked as AI languageModel input to sentiment analysis node  
    - Edge Cases: API key invalid, rate limits, model unavailability  
    - Version: 1.2

  - **Analyze sentiment of every comment**  
    - Type: Langchain Sentiment Analysis node  
    - Configuration:  
      - Categories: Positive, Neutral, Negative  
      - System prompt instructs the model to output JSON with sentiment category only  
      - Input text: Extracted from each comment’s textOriginal field  
      - Uses OpenAI Chat Model node as language model source  
    - Inputs: Each comment item from Split Out  
    - Outputs: JSON with sentiment category added to each comment item  
    - Edge Cases: Model response errors, unexpected output format, API failures  
    - Version: 1

#### 1.5 Data Formatting

- **Overview:**  
  Prepares and formats the comment data along with sentiment results into a structured format suitable for Google Sheets insertion or update.

- **Nodes Involved:**  
  - Format fields as required to save in google sheet

- **Node Details:**  
  - **Format fields as required to save in google sheet**  
    - Type: Set node  
    - Configuration: Assigns multiple fields including commentId, video_url, comment text, authorName, likes, reply count, sentiment category, and published date.  
    - Inputs: From sentiment analysis node  
    - Outputs: Structured JSON ready for Google Sheets  
    - Edge Cases: Missing fields in input JSON, data type mismatches  
    - Version: 3.4

#### 1.6 Data Storage

- **Overview:**  
  Inserts or updates the formatted comment data into the results Google Sheet and updates the fetch timestamps in the video URLs sheet.

- **Nodes Involved:**  
  - Insert and update comment in google sheet  
  - Update last fetched time and next_fetch_time

- **Node Details:**  
  - **Insert and update comment in google sheet**  
    - Type: Google Sheets node (appendOrUpdate operation)  
    - Configuration:  
      - Target sheet: Results sheet (Sheet1) with columns matching formatted fields  
      - Matching column: `commentId` to avoid duplicates  
      - Authentication: Service account  
    - Inputs: From formatting node  
    - Outputs: Triggers timestamp update node  
    - Edge Cases: Sheet access errors, duplicate comment IDs, API rate limits  
    - Version: 4.5

  - **Update last fetched time and next_fetch_time**  
    - Type: Google Sheets node (appendOrUpdate operation)  
    - Configuration:  
      - Target sheet: Video URLs sheet (Sheet2)  
      - Updates `last_fetched_time` to current time, `next_fetch_time` to 5 minutes later  
      - Matching column: `video_urls`  
      - Authentication: Service account  
    - Inputs: From comment insertion node  
    - Outputs: None (workflow end)  
    - Edge Cases: Timezone handling, concurrent updates, sheet access issues  
    - Version: 4.5

#### 1.7 Workflow Trigger and Control

- **Overview:**  
  Provides manual initiation and manages workflow branches with no-operation nodes.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - No Operation, do nothing  
  - No Operation, do nothing1  
  - Sticky Note1 and Sticky Note2 (documentation and instructions)

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger node  
    - Role: Starts the workflow on demand  
    - Outputs: Triggers reading video URLs  
    - Version: 1

  - **Sticky Notes**  
    - Type: Sticky Note nodes  
    - Content: Provide detailed usage instructions, setup tips, and workflow overview  
    - Role: Documentation within the workflow editor  
    - Version: 1

---

### 3. Summary Table

| Node Name                            | Node Type                         | Functional Role                              | Input Node(s)                         | Output Node(s)                          | Sticky Note                                                                                         |
|------------------------------------|----------------------------------|----------------------------------------------|-------------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’       | Manual Trigger                   | Starts the workflow manually                  | -                                   | Get Video Urls from Google Sheet      | See Sticky Note2 for detailed usage instructions                                                  |
| Get Video Urls from Google Sheet    | Google Sheets                   | Reads video URLs and fetch metadata           | When clicking ‘Test workflow’        | check next fetch time is available or not |                                                                                                   |
| check next fetch time is available or not | IF                             | Checks if next_fetch_time is empty             | Get Video Urls from Google Sheet     | Get Comments for video urls, check next fetch time is before the current time |                                                                                                   |
| check next fetch time is before the current time | IF                             | Checks if next_fetch_time is before now        | check next fetch time is available or not | Get Comments for video urls, No Operation, do nothing |                                                                                                   |
| No Operation, do nothing            | NoOp                            | Ends branch when fetch time not reached        | check next fetch time is before the current time | -                                     |                                                                                                   |
| Get Comments for video urls         | HTTP Request                   | Fetches YouTube comments via API               | check next fetch time is available or not, check next fetch time is before the current time | Check Success Response                 |                                                                                                   |
| Check Success Response              | IF                             | Validates HTTP 200 response                     | Get Comments for video urls          | Split Out, No Operation, do nothing1  |                                                                                                   |
| No Operation, do nothing1           | NoOp                            | Ends branch on unsuccessful API response       | Check Success Response               | -                                     |                                                                                                   |
| Split Out                         | SplitOut                       | Splits comments array into individual items    | Check Success Response               | Analyze sentiment of every comment    |                                                                                                   |
| OpenAI Chat Model                   | Langchain OpenAI Chat Model     | Provides GPT-4o-mini model for sentiment analysis | -                                   | Analyze sentiment of every comment (ai_languageModel input) |                                                                                                   |
| Analyze sentiment of every comment | Langchain Sentiment Analysis    | Classifies comment sentiment                    | Split Out, OpenAI Chat Model         | Format fields as required to save in google sheet |                                                                                                   |
| Format fields as required to save in google sheet | Set                             | Formats comment and sentiment data for Sheets | Analyze sentiment of every comment   | Insert and update comment in google sheet |                                                                                                   |
| Insert and update comment in google sheet | Google Sheets                   | Inserts or updates comment data in results sheet | Format fields as required to save in google sheet | Update last fetched time and next_fetch_time |                                                                                                   |
| Update last fetched time and next_fetch_time | Google Sheets                   | Updates fetch timestamps in video URLs sheet   | Insert and update comment in google sheet | -                                     |                                                                                                   |
| Sticky Note1                       | Sticky Note                    | Workflow title and overview                     | -                                   | -                                     | # 🚀 YouTube Comment Sentiment Analyzer with Google Sheets & OpenAI                               |
| Sticky Note2                       | Sticky Note                    | Detailed usage instructions                      | -                                   | -                                     | See section 5 for full content                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on demand.

2. **Add Google Sheets Node to Read Video URLs**  
   - Type: Google Sheets (Read)  
   - Configure:  
     - Document ID: Your Google Sheet ID containing video URLs and timestamps  
     - Sheet Name: Sheet2 (or your URLs sheet)  
     - Authentication: Service Account OAuth2 credential  
   - Output: JSON with columns `video_urls`, `last_fetched_time`, `next_fetch_time`.

3. **Add IF Node to Check if `next_fetch_time` is Empty**  
   - Type: IF  
   - Condition: `next_fetch_time` is empty (string empty)  
   - True: Proceed to fetch comments  
   - False: Proceed to next IF node

4. **Add IF Node to Check if `next_fetch_time` is Before Current Time**  
   - Type: IF  
   - Condition: `next_fetch_time` < current time (`$now.toISO()`)  
   - True: Proceed to fetch comments  
   - False: Connect to No Operation node

5. **Add No Operation Node**  
   - Type: NoOp  
   - Purpose: End workflow branch if fetch time not reached.

6. **Add HTTP Request Node to Fetch YouTube Comments**  
   - Type: HTTP Request  
   - URL: `https://www.googleapis.com/youtube/v3/commentThreads`  
   - Query Parameters:  
     - part: snippet  
     - videoId: Extracted from `video_urls` using regex `(?:v=|\/)([0-9A-Za-z_-]{11})`  
     - maxResults: 100  
   - Authentication: HTTP Query Auth with YouTube Data API key  
   - Pagination: Use `pageToken` parameter from response `nextPageToken` to paginate until no token  
   - Output: Full API response JSON

7. **Add IF Node to Check HTTP Response Status Code**  
   - Type: IF  
   - Condition: `statusCode` equals 200  
   - True: Proceed to Split Out node  
   - False: Connect to No Operation node

8. **Add No Operation Node**  
   - Type: NoOp  
   - Purpose: End branch on API failure.

9. **Add Split Out Node**  
   - Type: SplitOut  
   - Field to split: `body.items` (array of comments)  
   - Purpose: Process each comment individually.

10. **Add Langchain OpenAI Chat Model Node**  
    - Type: Langchain OpenAI Chat Model  
    - Model: GPT-4o-mini  
    - Credentials: OpenAI API key

11. **Add Langchain Sentiment Analysis Node**  
    - Type: Sentiment Analysis  
    - Categories: Positive, Neutral, Negative  
    - System Prompt: Instruct model to output JSON with sentiment category only  
    - Input Text: `snippet.topLevelComment.snippet.textOriginal` from each comment  
    - AI Language Model: Connect to OpenAI Chat Model node

12. **Add Set Node to Format Data for Google Sheets**  
    - Type: Set  
    - Fields to assign:  
      - commentId: `snippet.topLevelComment.id`  
      - video_url: Constructed as `https://www.youtube.com/watch?v={{snippet.videoId}}`  
      - comment: `snippet.topLevelComment.snippet.textOriginal`  
      - authorName: `snippet.topLevelComment.snippet.authorDisplayName`  
      - likes: `snippet.topLevelComment.snippet.likeCount`  
      - reply: `snippet.totalReplyCount`  
      - sentiment: Sentiment category from sentiment analysis  
      - published_at: `snippet.topLevelComment.snippet.publishedAt`

13. **Add Google Sheets Node to Insert or Update Comments**  
    - Type: Google Sheets (appendOrUpdate)  
    - Document ID: Same as input sheet  
    - Sheet Name: Sheet1 (results sheet)  
    - Matching Column: `commentId` (to avoid duplicates)  
    - Columns: Map all fields from Set node  
    - Authentication: Service Account

14. **Add Google Sheets Node to Update Fetch Timestamps**  
    - Type: Google Sheets (appendOrUpdate)  
    - Document ID: Same as input sheet  
    - Sheet Name: Sheet2 (video URLs sheet)  
    - Matching Column: `video_urls`  
    - Columns to update:  
      - `last_fetched_time`: Current time (`$now.toISO()`)  
      - `next_fetch_time`: Current time plus 5 minutes (`$now.plus(5, 'min').toISO()`)  
    - Authentication: Service Account

15. **Connect Nodes in the Following Order:**  
    Manual Trigger → Get Video Urls from Google Sheet → check next fetch time is available or not →  
    (True branch) → Get Comments for video urls → Check Success Response → Split Out → Analyze sentiment of every comment → Format fields → Insert and update comment → Update last fetched time and next_fetch_time  
    (False branch from first IF) → check next fetch time is before the current time →  
    (True branch) → Get Comments for video urls (same as above)  
    (False branch) → No Operation node

16. **Optional:** Add Sticky Notes with usage instructions and workflow overview for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow automates YouTube comment sentiment analysis using Google Sheets and OpenAI, eliminating manual exports and data handling. It supports pagination of YouTube comments and updates Google Sheets with sentiment tags for each comment.                                                                                                                                                                                                                                                                                                                                                                                                                | Workflow purpose and summary                                                                                       |
| Setup requires enabling YouTube Data API v3 in Google Cloud Console, creating an API key, and configuring Google Sheets credentials (preferably service account) in n8n. OpenAI API key is needed for sentiment analysis.                                                                                                                                                                                                                                                                                                                                                                                                                                         | Setup instructions                                                                                                 |
| Replace Manual Trigger with Cron node to automate periodic runs. Customize sentiment analysis prompt to perform other NLP tasks like summarization or translation. Google Sheets can be swapped for Airtable, Notion, or databases.                                                                                                                                                                                                                                                                                                                                                                                                                                   | Customization tips                                                                                                |
| YouTube video ID extraction uses regex `(?:v=|\/)([0-9A-Za-z_-]{11})` to parse video URLs robustly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Technical detail                                                                                                   |
| Google Sheets structure: Sheet1 for results with columns: commentId, video_url, comment, authorName, likes, reply, sentiment, published_at; Sheet2 for video URLs with columns: video_urls, last_fetched_time, next_fetch_time.                                                                                                                                                                                                                                                                                                                                                                                                                                       | Data schema                                                                                                       |
| Workflow includes sticky notes with detailed instructions and usage tips embedded for user reference inside n8n editor.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Documentation inside workflow                                                                                      |
| Potential failure points include API rate limits, authentication errors, malformed URLs, empty or missing data, and model response errors. Implement error handling or retries as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Error and edge case considerations                                                                                 |

---

This completes the comprehensive reference document for the "🚀 YouTube Comment Sentiment Analyzer with Google Sheets & OpenAI" n8n workflow.