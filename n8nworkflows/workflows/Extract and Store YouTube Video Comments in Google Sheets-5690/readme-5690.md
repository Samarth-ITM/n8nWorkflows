Extract and Store YouTube Video Comments in Google Sheets

https://n8nworkflows.xyz/workflows/extract-and-store-youtube-video-comments-in-google-sheets-5690


# Extract and Store YouTube Video Comments in Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of comments from YouTube videos and stores them in a Google Sheets document for easy review and analysis. It is designed for users who want to collect YouTube video engagement data such as comments, likes, replies, and author information, then centralize this data in a structured spreadsheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Retrieve YouTube video URLs marked as "Ready" from a Google Sheets tab.
- **1.2 URL Validation:** Check each video URL for validity before processing.
- **1.3 YouTube API Comment Fetching:** Loop through each video URL and request comment data from the YouTube Data API.
- **1.4 Response Validation and Data Splitting:** Verify the API response status, then split the received batch of comments into individual comment entries.
- **1.5 Comment Processing and Storage:** For each comment, check if it exists, then insert or update it in the Google Sheets "Results" tab.
- **1.6 Status Update:** Update the status of each processed video URL in Google Sheets to "Finished" or "Error" depending on success or failure.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block extracts the list of YouTube video URLs to process from a Google Sheets tab named "Video URLs". Only rows with the status "Ready" are fetched to ensure selective processing.

- **Nodes Involved:**  
  - `Test Workflow` (Manual trigger)  
  - `Google Sheets - Get Video URLs`  
  - `If - Check Video Url is Not Empty`

- **Node Details:**

  - **Test Workflow**  
    - Type: Manual trigger  
    - Role: Starts the workflow manually on demand.  
    - Inputs: None  
    - Outputs: Triggers the next node to fetch video URLs.  
    - Edge cases: User must manually trigger; no automatic scheduling.

  - **Google Sheets - Get Video URLs**  
    - Type: Google Sheets  
    - Role: Queries the "Video URLs" tab filtering rows where `status` equals "ready".  
    - Configuration:  
      - Document ID and Sheet ID point to the connected Google Sheet template with the video URLs.  
      - Filters applied on the `status` column to fetch only relevant rows.  
    - Inputs: Trigger from Manual node  
    - Outputs: List of video URL entries, each including a row number and status.  
    - Credentials: Requires Google Sheets OAuth2 credentials with read access.  
    - Failure types: Authentication errors, network timeouts, or empty result sets.

  - **If - Check Video Url is Not Empty**  
    - Type: Conditional (If) node  
    - Role: Checks if the `video_url` field from the fetched rows is non-empty before processing.  
    - Key expression: `{{$json.video_url}}` not empty.  
    - Inputs: Output of Google Sheets node  
    - Outputs: Passes valid URLs to next block, discards empty entries.  
    - Edge cases: Blank or malformed URLs bypassed here.

---

#### 2.2 URL Validation and Loop Setup

- **Overview:**  
  This block prepares the valid URLs for iterative processing by batching them and initiating the loop for comment fetching.

- **Nodes Involved:**  
  - `Loop Over Items`

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes video URLs one at a time (or in batches) to avoid overloading API or rate limits.  
    - Configuration: Default batch size (implied 1), no special options set.  
    - Inputs: Valid URLs from previous If node  
    - Outputs: Individual video URL items sent for API requests.  
    - Edge cases: Large input lists may require batch size tuning to avoid timeouts.

---

#### 2.3 YouTube API Comment Fetching

- **Overview:**  
  For each video URL, the workflow calls the YouTube Data API to fetch comment threads using the video ID extracted via regex.

- **Nodes Involved:**  
  - `HTTP Request - Get Comments`  
  - `If - Check Success Response`

- **Node Details:**

  - **HTTP Request - Get Comments**  
    - Type: HTTP Request  
    - Role: Sends a GET request to YouTube API endpoint `/commentThreads` for a specific video ID.  
    - Configuration:  
      - URL: `https://www.googleapis.com/youtube/v3/commentThreads`  
      - Query parameters:  
        - `part=snippet` to get comment details  
        - `videoId` extracted using regex from the video URL: `{{$json.video_url.match(/(?:v=|\/)([0-9A-Za-z_-]{11})/)[1] || ''}}`  
        - `limit=100` (maximum comments per page)  
      - Authentication: Uses predefined YouTube OAuth2 credentials.  
      - Pagination: Automatically uses `pageToken` for paginated results until all comments are fetched.  
    - Inputs: Single video URL from Loop node  
    - Outputs: Full API response including status code and body.  
    - Edge cases / Failures:  
      - Invalid or expired API credentials  
      - Quota exceeded errors  
      - Invalid video ID causing 404 or empty results  
      - Network timeouts

  - **If - Check Success Response**  
    - Type: Conditional (If) node  
    - Role: Checks if HTTP response status code equals 200 (success).  
    - Condition expression: `{{$json.statusCode}} === 200`  
    - Inputs: API response node  
    - Outputs:  
      - True branch: Continue processing comments  
      - False branch: Update status with error  
    - Edge cases: Handles API errors gracefully by branching accordingly.

---

#### 2.4 Response Validation and Data Splitting

- **Overview:**  
  After confirming a successful API response, this block splits the fetched comment threads array into individual comment items for processing.

- **Nodes Involved:**  
  - `Split Out`  
  - `If - Check Comment Exists`

- **Node Details:**

  - **Split Out**  
    - Type: SplitOut  
    - Role: Splits the array found in `body.items` of the API response into separate workflow items.  
    - Configuration: Splits on the field `body.items`.  
    - Inputs: API response node (successful)  
    - Outputs: Individual comment thread JSON objects per item.  
    - Edge cases: Empty or missing `items` array results in no output items.

  - **If - Check Comment Exists**  
    - Type: Conditional (If) node  
    - Role: Checks if each comment thread actually contains a comment snippet by verifying existence of `snippet.videoId`.  
    - Condition expression: checks `{{$json.snippet.videoId}}` exists.  
    - Inputs: Individual comment items  
    - Outputs:  
      - True branch: Valid comment to process  
      - False branch: Skips empty or invalid comments  
    - Edge cases: API may return empty or deleted comments; these are filtered out here.

---

#### 2.5 Comment Processing and Storage

- **Overview:**  
  Each valid comment is appended or updated in the "Results" tab of the Google Sheets document, ensuring deduplication by `comment_id`.

- **Nodes Involved:**  
  - `Google Sheets - Insert/Update Comment`  
  - `Google Sheets - Update Status`  
  - `Google Sheets - Update Status - Error`

- **Node Details:**

  - **Google Sheets - Insert/Update Comment**  
    - Type: Google Sheets  
    - Role: Inserts new or updates existing comment records in the "Results" tab.  
    - Configuration:  
      - Operation: `appendOrUpdate`  
      - Matching column: `comment_id` to avoid duplicates  
      - Columns mapped:  
        - `likes`: number of likes on comment  
        - `reply`: total reply count for comment  
        - `comment`: original text of comment  
        - `video_url`: constructed as `https://www.youtube.com/watch?v={{snippet.videoId}}`  
        - `comment_id`  
        - `author_name`  
        - `published_at`: datetime formatted as `YYYY-MM-DD HH:mm:ss`  
      - Document & Sheet IDs point to the connected Google Sheet under "Results" tab  
    - Inputs: Valid comment items from previous If node  
    - Outputs: On success, triggers status update  
    - Credentials: Google Sheets OAuth2 credentials with edit access  
    - Edge cases: Rate limits on Google API, malformed comment data.

  - **Google Sheets - Update Status**  
    - Type: Google Sheets  
    - Role: Updates the processed video's row in "Video URLs" tab to status "finish" with current timestamp.  
    - Configuration:  
      - Operation: `update`  
      - Matching column: `row_number` to locate the correct row  
      - Columns updated:  
        - `status` = "finish"  
        - `last_fetched_time` = current ISO timestamp trimmed to seconds  
      - Document & Sheet IDs point to the connected Google Sheet under "Video URLs" tab  
    - Inputs: Triggered after comment insert/update node  
    - Outputs: Triggers next loop iteration  
    - Edge cases: Failure in updating status may cause reprocessing of same video.

  - **Google Sheets - Update Status - Error**  
    - Type: Google Sheets  
    - Role: Updates the video's row in case of API or processing errors, marking status as "error".  
    - Configuration: Similar to Update Status node but sets `status` to "error".  
    - Inputs: Triggered on failure branches from API call or success check nodes  
    - Outputs: Triggers loop for next video  
    - Edge cases: Ensures failed URLs are marked, avoiding infinite retry loops.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                            | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                                                       |
|-------------------------------|-------------------------|--------------------------------------------|--------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Test Workflow                  | Manual Trigger          | Starts the workflow manually                | -                              | Google Sheets - Get Video URLs    |                                                                                                                                  |
| Google Sheets - Get Video URLs | Google Sheets           | Fetches list of video URLs with "ready" status | Test Workflow                  | If - Check Video Url is Not Empty | Sticky Note1: Explains fetching video URLs from Google Sheet "Video URLs" tab filtered by status "Ready".                        |
| If - Check Video Url is Not Empty | If (Conditional)       | Filters out empty or invalid video URLs     | Google Sheets - Get Video URLs  | Loop Over Items                   | Sticky Note1                                                                                                                     |
| Loop Over Items               | SplitInBatches          | Loops through each video URL individually   | If - Check Video Url is Not Empty | HTTP Request - Get Comments      | Sticky Note2: Describes sending API requests to YouTube for each video URL.                                                     |
| HTTP Request - Get Comments    | HTTP Request            | Calls YouTube API to fetch comments         | Loop Over Items                | If - Check Success Response       | Sticky Note2                                                                                                                     |
| If - Check Success Response    | If (Conditional)        | Validates API response status                | HTTP Request - Get Comments    | Split Out (success), Google Sheets - Update Status - Error (failure) | Sticky Note3: Validates response and processes or errors accordingly.                                                           |
| Split Out                     | SplitOut                | Splits comment threads array into single comment items | If - Check Success Response    | If - Check Comment Exists          | Sticky Note3                                                                                                                     |
| If - Check Comment Exists      | If (Conditional)        | Checks that comment data exists              | Split Out                     | Google Sheets - Insert/Update Comment, Google Sheets - Update Status - Error | Sticky Note3                                                                                                                     |
| Google Sheets - Insert/Update Comment | Google Sheets           | Inserts or updates each comment in "Results" tab | If - Check Comment Exists       | Google Sheets - Update Status      | Sticky Note4: Comments are stored, and video status is updated after processing.                                               |
| Google Sheets - Update Status  | Google Sheets           | Marks video URL row as "finish" after processing | Google Sheets - Insert/Update Comment | Loop Over Items                   | Sticky Note4                                                                                                                     |
| Google Sheets - Update Status - Error | Google Sheets           | Marks video URL row as "error" on failures  | If - Check Success Response, If - Check Comment Exists | Loop Over Items                   | Sticky Note4                                                                                                                     |
| Sticky Note                   | Sticky Note             | Workflow description and usage instructions | -                              | -                                | Main sticky note with detailed workflow overview and instructions                                                             |
| Sticky Note1                  | Sticky Note             | Notes on video URLs reading step             | -                              | -                                | See above                                                                                                                       |
| Sticky Note2                  | Sticky Note             | Notes on YouTube API comment fetching        | -                              | -                                | See above                                                                                                                       |
| Sticky Note3                  | Sticky Note             | Notes on response validation and splitting   | -                              | -                                | See above                                                                                                                       |
| Sticky Note4                  | Sticky Note             | Notes on saving comments and updating status | -                              | -                                | See above                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Add a **Manual Trigger** node named `Test Workflow` to start the workflow manually.

2. **Add Google Sheets Node to Get Video URLs**
   - Create a **Google Sheets** node named `Google Sheets - Get Video URLs`.
   - Configure credentials with Google Sheets OAuth2 (with access to your target spreadsheet).
   - Set operation to "Read Rows".
   - Set Document ID and Sheet ID to point to your Google Sheet containing video URLs.
   - Apply a filter where `status` column equals `"ready"`.
   - Connect `Test Workflow` output to this node.

3. **Add Conditional Node to Check Video URL**
   - Add an **If** node named `If - Check Video Url is Not Empty`.
   - Condition: Check if `video_url` field from input is not empty.
   - Connect output of `Google Sheets - Get Video URLs` to this node.

4. **Add SplitInBatches Node to Loop Over URLs**
   - Add a **SplitInBatches** node named `Loop Over Items`.
   - Leave batch size as default (1).
   - Connect the True output of `If - Check Video Url is Not Empty` to this node.

5. **Add HTTP Request Node to Fetch Comments**
   - Add an **HTTP Request** node named `HTTP Request - Get Comments`.
   - Set method to GET.
   - URL: `https://www.googleapis.com/youtube/v3/commentThreads`.
   - Query Parameters:
     - `part = snippet`
     - `videoId = {{$json.video_url.match(/(?:v=|\/)([0-9A-Za-z_-]{11})/)[1] || ''}}`
     - `limit = 100`
   - Enable pagination with pageToken until no nextPageToken.
   - Set authentication to YouTube OAuth2 credentials.
   - Connect output of `Loop Over Items` to this node.

6. **Add If Node to Check Success Response**
   - Add an **If** node named `If - Check Success Response`.
   - Condition: `{{$json.statusCode}} === 200`
   - Connect output of `HTTP Request - Get Comments` to this node.
   - True output goes to the next step; False output to error handling.

7. **Add SplitOut Node to Split Comments**
   - Add a **SplitOut** node named `Split Out`.
   - Field to split: `body.items`.
   - Connect True output of `If - Check Success Response` to this node.

8. **Add If Node to Check Comment Exists**
   - Add an **If** node named `If - Check Comment Exists`.
   - Condition: Check if `snippet.videoId` exists in each split comment.
   - Connect output of `Split Out` to this node.

9. **Add Google Sheets Node to Insert or Update Comments**
   - Add a **Google Sheets** node named `Google Sheets - Insert/Update Comment`.
   - Credentials: Google Sheets OAuth2 with write access.
   - Operation: `appendOrUpdate`.
   - Document ID and Sheet ID point to the "Results" tab in the Google Sheet.
   - Mapping columns (match on `comment_id`):
     - `video_url` = `https://www.youtube.com/watch?v={{ $json.snippet.videoId }}`
     - `comment_id` = `{{$json.snippet.topLevelComment.id}}`
     - `comment` = `{{$json.snippet.topLevelComment.snippet.textOriginal}}`
     - `author_name` = `{{$json.snippet.topLevelComment.snippet.authorDisplayName}}`
     - `likes` = `{{$json.snippet.topLevelComment.snippet.likeCount}}`
     - `reply` = `{{$json.snippet.totalReplyCount}}`
     - `published_at` = `{{$json.snippet.topLevelComment.snippet.publishedAt.toString().slice(0,19).replace('T',' ')}}`
   - Connect True output of `If - Check Comment Exists` to this node.

10. **Add Google Sheets Node to Update Status on Success**
    - Add a **Google Sheets** node named `Google Sheets - Update Status`.
    - Operation: `update`.
    - Document ID and Sheet ID point to the "Video URLs" tab.
    - Matching column: `row_number` from the original video URL row.
    - Columns to update:
      - `status` = `finish`
      - `last_fetched_time` = current timestamp formatted as `YYYY-MM-DD HH:mm:ss`
    - Connect output of `Google Sheets - Insert/Update Comment` to this node.
    - Connect output of this node back to `Loop Over Items` to process the next video.

11. **Add Google Sheets Node to Update Status on Error**
    - Add a **Google Sheets** node named `Google Sheets - Update Status - Error`.
    - Operation: `update`.
    - Same sheet and column setup as above.
    - Columns to update:
      - `status` = `error`
      - `last_fetched_time` = current timestamp.
    - Connect False outputs from `If - Check Success Response` and `If - Check Comment Exists` to this node.
    - Connect its output back to `Loop Over Items`.

12. **Final Workflow Notes**
    - Ensure all Google Sheets nodes have correct OAuth2 credentials with sufficient permissions.
    - Ensure the YouTube OAuth2 credential is configured with YouTube Data API enabled.
    - Prepare the Google Sheet with two tabs:
      - "Video URLs" with columns: status, video_url, row_number (and others)
      - "Results" tab for storing comments.
    - Adjust batch size or pagination if needed for large datasets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow template is created by Agent Circle to demonstrate how to crawl YouTube video comments and store them in a Google Sheet for analysis or reporting. It is useful for creators, marketers, analysts, and growth teams.                                                                                                                                                                                                 | https://www.agentcircle.ai/                                                                        |
| The connected Google Sheet template ("YouTube - Get Video Comments") can be duplicated here for use: [Google Sheets Template](https://docs.google.com/spreadsheets/d/1F5yEhjBWu3fnwgHGLsPLD9_tWRqUnxu4p0DEvUTae1Y/edit?gid=426418282#gid=426418282)                                                                                                                                                                                | Link to Google Sheets template                                                                     |
| Setup in Google Cloud Console is required with enabled APIs for YouTube Data API and Google Sheets API. OAuth2 credentials must be created and linked in n8n nodes.                                                                                                                                                                                                                                                              | Google Cloud Console Documentation                                                                 |
| To automate this workflow beyond manual trigger, a Google Sheets trigger node can be added to watch the "Video URLs" tab for new "Ready" entries and start the workflow automatically.                                                                                                                                                                                                                                            | n8n Google Sheets Trigger Documentation                                                           |
| Community and support channels for Agent Circle: Discord, Facebook, X, YouTube, LinkedIn — useful for troubleshooting and workflow inspiration.                                                                                                                                                                                                                                                                                 | Discord: https://discord.gg/d8SkCzKwnP <br> Facebook: https://www.facebook.com/agentcircle/ <br> X: https://x.com/agent_circle <br> YouTube: https://www.youtube.com/@agentcircle <br> LinkedIn: https://www.linkedin.com/company/agentcircle |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects existing content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.