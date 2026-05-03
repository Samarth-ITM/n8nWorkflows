Extract YouTube Video Statistics and Save to Google Sheets

https://n8nworkflows.xyz/workflows/extract-youtube-video-statistics-and-save-to-google-sheets-5766


# Extract YouTube Video Statistics and Save to Google Sheets

---

### 1. Workflow Overview

This workflow automates the extraction of detailed YouTube video statistics and stores the results into a Google Sheets spreadsheet for easy tracking and analysis. It is designed for users such as YouTubers, content strategists, marketers, and automation engineers who want to monitor video metrics like views, likes, comments, and metadata efficiently.

**Target Use Cases:**
- Bulk fetching of YouTube video statistics from a list of video URLs.
- Maintaining an up-to-date Google Sheet with video data for reporting or analysis.
- Handling success and error cases to track which videos' data was updated or failed.

**Logical Blocks:**

- **1.1 Input Reception & Filtering:** Triggering the workflow manually and reading the list of YouTube video URLs marked as “Ready” from a Google Sheet.
- **1.2 Video Data Retrieval:** For each video URL, extracting the video ID, then querying the YouTube Data API for key video details.
- **1.3 Result Validation and Storage:** Checking if the API call was successful, then writing the retrieved data back into the Google Sheet or marking the row as error if the call failed.
- **1.4 Loop Control:** Managing the iteration over individual video URLs to process them sequentially.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:**  
  This block initiates the workflow manually and reads video URLs from a Google Sheet tab where the status is set to “ready.” It prepares the items for processing.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Google Sheets - Get Video URLs  
  - Loop Over Items (Split In Batches)  

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point; triggers workflow execution manually.  
    - Configuration: No parameters; triggers on user action.  
    - Inputs: None  
    - Outputs: Connects to "Google Sheets - Get Video URLs"  
    - Edge Cases: None; manual trigger is user-controlled.

  - **Google Sheets - Get Video URLs**  
    - Type: Google Sheets node (Read)  
    - Role: Reads video URLs from a specific Google Sheet tab called “Video Urls.”  
    - Configuration:  
      - Filters rows by status = "ready" to only process videos queued for update.  
      - Document ID and Sheet Name reference a specific Google Sheet template.  
    - Key Expressions: Filters on `status` column with value "ready".  
    - Inputs: From manual trigger.  
    - Outputs: Connects to "Loop Over Items".  
    - Edge Cases:  
      - No rows found with status “ready” means no processing.  
      - Google Sheets API errors or auth failures.  
    - Credentials: Google Sheets OAuth2 API with proper permissions.

  - **Loop Over Items**  
    - Type: Split in Batches  
    - Role: Processes each video URL one by one to avoid API rate limits or concurrency issues.  
    - Configuration: Default batch size (usually 1) to iterate over each item.  
    - Inputs: From Google Sheets - Get Video URLs.  
    - Outputs: Connects to "HTTP - Find Video Data".  
    - Edge Cases:  
      - Empty input array causes loop to skip.  
      - Batch size misconfiguration could cause unexpected concurrency.

---

#### 2.2 Video Data Retrieval

- **Overview:**  
  Extracts the video ID from the URL and sends a request to the YouTube Data API to fetch video statistics and metadata.

- **Nodes Involved:**  
  - HTTP - Find Video Data  
  - If - Check Success Response  

- **Node Details:**

  - **HTTP - Find Video Data**  
    - Type: HTTP Request  
    - Role: Queries YouTube Data API v3 for video details (contentDetails, snippet, statistics).  
    - Configuration:  
      - HTTP method: GET  
      - URL: https://www.googleapis.com/youtube/v3/videos?  
      - Query parameters:  
        - `id`: Extracted video ID from video URL using regex (matches YouTube URL formats).  
        - `part`: "contentDetails,snippet,statistics"  
      - Authentication: Predefined YouTube OAuth2 credentials.  
    - Key Expressions:  
      - Extract video ID using regex: `{{$json.video_url.match(/(?:youtube(?:-nocookie)?\.com\/(?:[^\/\n\s]+\/\S+\/|(?:v|e(?:mbed)?)\/|\S*?[?&]v=)|youtu\.be\/)([a-zA-Z0-9_-]{11})/)[1]}}`  
    - Inputs: From "Loop Over Items".  
    - Outputs: Connects to "If - Check Success Response".  
    - Error Handling: On error, continues regular output but no data.  
    - Edge Cases:  
      - Invalid or malformed video URL causes regex to fail (may throw expression error or undefined).  
      - YouTube API rate limiting or quota exceeded.  
      - OAuth token expiration or invalid credentials.  
      - Video removed or private (no data returned).  

  - **If - Check Success Response**  
    - Type: If node (conditional branching)  
    - Role: Checks if the YouTube API response contains valid video data.  
    - Configuration:  
      - Condition: Checks if the field `pageInfo.totalResults` exists and equals 200 (loosely validated).  
      - This condition is to verify if the response is successful and contains results.  
    - Inputs: From "HTTP - Find Video Data".  
    - Outputs:  
      - True branch: Connects to "Google Sheets - Update Data" (successful fetch).  
      - False branch: Connects to "Google Sheets - Update Data - Error" (failed fetch).  
    - Edge Cases:  
      - API could return zero results or partial data; this condition ensures only full success.  
      - If response schema changes, condition may fail.  

---

#### 2.3 Result Validation and Storage

- **Overview:**  
  Depending on the success of the API call, updates the corresponding Google Sheet row with video statistics or marks the row as error.

- **Nodes Involved:**  
  - Google Sheets - Update Data  
  - Google Sheets - Update Data - Error  

- **Node Details:**

  - **Google Sheets - Update Data**  
    - Type: Google Sheets node (Update)  
    - Role: Writes video statistics and metadata back into the Google Sheet for the matching row.  
    - Configuration:  
      - Sheet Name: "Video Urls" tab.  
      - Document ID: Same Google Sheet as used earlier.  
      - Matching Column: `row_number` to update correct row.  
      - Fields updated include title, description, thumbnails, tags, duration (formatted from ISO8601), view count, like count, comment count, language, published date, channel info, last fetched time, and status set to "finish".  
      - Uses expressions to extract and format data from API response JSON.  
    - Inputs: True branch from "If - Check Success Response".  
    - Outputs: Loops back to "Loop Over Items" for next video.  
    - Credentials: Google Sheets OAuth2 API.  
    - Edge Cases:  
      - Google API update failures or permission issues.  
      - Mismatched `row_number` causing wrong updates.  
      - Data formatting errors (e.g., duration parsing).  

  - **Google Sheets - Update Data - Error**  
    - Type: Google Sheets node (Update)  
    - Role: Updates the row with status "error" and logs last fetched time if API call failed.  
    - Configuration:  
      - Similar to Update Data node but only updates `status` to "error" and `last_fetched_time`.  
      - Matching on `row_number` to update correct row.  
    - Inputs: False branch from "If - Check Success Response".  
    - Outputs: Loops back to "Loop Over Items" for next video.  
    - Credentials: Google Sheets OAuth2 API.  
    - Edge Cases:  
      - Same as update node, with risk of mislabeling rows.  

---

#### 2.4 Loop Control

- **Overview:**  
  Manages the iteration over the list of videos, ensuring each video URL is processed in sequence until all are handled.

- **Nodes Involved:**  
  - Loop Over Items  
  - Connections from Google Sheets - Update Data and Google Sheets - Update Data - Error back to Loop Over Items  

- **Node Details:**

  - **Loop Over Items** (already described in 2.1)  
    - After each update (success or error), the loop proceeds to the next item until all videos processed.  
    - Ensures orderly execution and avoids concurrency issues.  

---

### 3. Summary Table

| Node Name                    | Node Type             | Functional Role                                      | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                      |
|------------------------------|----------------------|-----------------------------------------------------|-----------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger       | Workflow entry point, manual start                   | None                        | Google Sheets - Get Video URLs  |                                                                                                |
| Google Sheets - Get Video URLs| Google Sheets (Read) | Reads video URLs with status “ready” from Sheet     | When clicking ‘Test workflow’| Loop Over Items                 | ## 1. Read Video URLs from Google Sheets - We'll read all video URLs listed in your connected Google Sheet – typically from a tab named **Video URLs**. Only  - The loop iterates over each row that's marked as **Ready** in the status **Column A** for processing. |
| Loop Over Items              | Split In Batches      | Loops over each video URL individually                | Google Sheets - Get Video URLs| HTTP - Find Video Data          |                                                                                                |
| HTTP - Find Video Data       | HTTP Request         | Calls YouTube API to fetch video statistics          | Loop Over Items             | If - Check Success Response     | ## 2. Fetch Video Statistics From YouTube API Once a video URL is read, the workflow sends a request to the YouTube API to pull essential data for the corresponding video. |
| If - Check Success Response  | If                   | Checks if response is valid                           | HTTP - Find Video Data       | Google Sheets - Update Data, Google Sheets - Update Data - Error |                                                                                                |
| Google Sheets - Update Data  | Google Sheets (Update)| Updates Sheet row with video data on success         | If - Check Success Response (true) | Loop Over Items          | ## 3. Write Results Back To Google Sheets - The API response is validated, in the connected Google Sheet: - If successful: the video’s statistics are filled into the corresponding row, and the row's status in **Column A** in the **Video URLs** tab is marked as **Finished**. - If unsuccessful: the row's status in **Column A** in the **Video URLs** tab is updated to **Error** for later review. - Then, the tool moves on to the next available video, if one exists. |
| Google Sheets - Update Data - Error| Google Sheets (Update)| Updates Sheet row status to error on failure           | If - Check Success Response (false)| Loop Over Items          | Same as above                                                                                   |
| Sticky Note                  | Sticky Note          | Provides detailed workflow description and usage info| None                        | None                           | ## [Agent Circle's N8N Workflow] YouTube Video Statistics Crawler - Try It Out!... (full content describing workflow, usage, setup, and support links) |
| Sticky Note1                 | Sticky Note          | Explains the input reading block                      | None                        | None                           | ## 1. Read Video URLs from Google Sheets - We'll read all video URLs listed in your connected Google Sheet – typically from a tab named **Video URLs**. Only  - The loop iterates over each row that's marked as **Ready** in the status **Column A** for processing. |
| Sticky Note2                 | Sticky Note          | Explains fetching video data from YouTube API         | None                        | None                           | ## 2. Fetch Video Statistics From YouTube API Once a video URL is read, the workflow sends a request to the YouTube API to pull essential data for the corresponding video. |
| Sticky Note3                 | Sticky Note          | Explains writing results back to Google Sheets         | None                        | None                           | ## 3. Write Results Back To Google Sheets - The API response is validated, in the connected Google Sheet: - If successful: the video’s statistics are filled into the corresponding row, and the row's status in **Column A** in the **Video URLs** tab is marked as **Finished**. - If unsuccessful: the row's status in **Column A** in the **Video URLs** tab is updated to **Error** for later review. - Then, the tool moves on to the next available video, if one exists. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Node Type: Manual Trigger  
   - No special parameters. This will start the workflow manually.

2. **Add a Google Sheets node to read video URLs**  
   - Node Type: Google Sheets (Read)  
   - Configure credentials for Google Sheets OAuth2 with access to your spreadsheet.  
   - Set Document ID to your Google Sheet ID.  
   - Set Sheet Name to the tab containing video URLs (e.g., "Video Urls").  
   - Add a filter to only select rows where the `status` column equals "ready".  
   - Connect output from Manual Trigger to this node.

3. **Add a Split In Batches node to process videos one at a time**  
   - Node Type: Split In Batches  
   - Connect input from Google Sheets - Get Video URLs node.  
   - Use default batch size = 1 for sequential processing.

4. **Add an HTTP Request node to fetch YouTube video data**  
   - Node Type: HTTP Request  
   - Configure with YouTube Data API v3 endpoint: `https://www.googleapis.com/youtube/v3/videos?`  
   - HTTP Method: GET  
   - Add Query Parameters:  
     - `id`: Extract video ID from the URL using regex (YouTube video ID regex). Example expression: `{{$json.video_url.match(/(?:youtube(?:-nocookie)?\.com\/(?:[^\/\n\s]+\/\S+\/|(?:v|e(?:mbed)?)\/|\S*?[?&]v=)|youtu\.be\/)([a-zA-Z0-9_-]{11})/)[1]}}`  
     - `part`: Set to `"contentDetails,snippet,statistics"`  
   - Authentication: Use preconfigured YouTube OAuth2 credentials with YouTube Data API access.  
   - Connect from Split In Batches node.

5. **Add an If node to check success of API response**  
   - Node Type: If  
   - Condition: Check if `pageInfo.totalResults` exists and equals 200 (or adjust based on expected success criteria).  
   - Connect from HTTP Request node.

6. **Add a Google Sheets node to update data on success**  
   - Node Type: Google Sheets (Update)  
   - Configure credentials with Google Sheets OAuth2.  
   - Set Document ID and Sheet Name to match your spreadsheet and tab.  
   - Matching column: `row_number` (or unique identifier for the row).  
   - Map columns with expressions from API response fields to update:  
     - title, description, thumbnails URL, tags (joined by commas), duration (parse ISO8601 duration), view count, like count, comment count, language, published date, channel id, channel title, last fetched time (current timestamp), status as "finish".  
   - Connect from If node’s "true" output.

7. **Add a Google Sheets node to update row status on error**  
   - Node Type: Google Sheets (Update)  
   - Same configuration as success node, but only update `status` to "error" and `last_fetched_time` with current timestamp.  
   - Connect from If node’s "false" output.

8. **Connect both Google Sheets update nodes back to Split In Batches node**  
   - This enables looping over the next video URL until all are processed.

9. **Optional: Add Sticky Notes for documentation and user instructions**  
   - Add notes explaining each major block and usage instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This N8N template makes it easy to extract key YouTube video data - including title, view count, like count, comment count, and many more - and save it directly into a connected Google Sheet. Use cases include content strategists, marketers, and automation engineers seeking fast, structured access to video-level insights. The workflow is manually triggered but can be automated by adding a Google Sheets trigger. Setup requires Google Cloud Console credentials with appropriate OAuth2 permissions for Google Sheets and YouTube APIs. | Workflow main sticky note content describing purpose, usage, and setup instructions.                                                 |
| Google Sheet template to use: [YouTube - Get Video Statistics](https://docs.google.com/spreadsheets/d/1I9gyb27WiRHz--g-xi-QH1W3WppZJdSfY6L32-DyUw0/edit#gid=426418282) (duplicate this sheet for your own use).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Google Sheets template link.                                                                                                        |
| Make sure Google Cloud Console credentials have enabled scope access for both YouTube Data API and Google Sheets API, and that OAuth2 credentials are configured in n8n nodes accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Credential setup note.                                                                                                              |
| For troubleshooting, watch out for: invalid video URLs causing regex failures, YouTube API quota limits, expired OAuth tokens, Google Sheets write permission errors, and network timeouts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Common error cases and troubleshooting tips.                                                                                        |
| Community and support links: Website (https://www.agentcircle.ai/), Etsy (https://www.etsy.com/shop/AgentCircle), Gumroad (http://agentcircle.gumroad.com/), Discord (https://discord.gg/d8SkCzKwnP), Facebook Page (https://www.facebook.com/agentcircle/), Facebook Group (https://www.facebook.com/groups/aiagentcircle/), Twitter/X (https://x.com/agent_circle), YouTube (https://www.youtube.com/@agentcircle), LinkedIn (https://www.linkedin.com/company/agentcircle)                                                                                                                                            | Support and community resources provided in sticky note.                                                                            |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.