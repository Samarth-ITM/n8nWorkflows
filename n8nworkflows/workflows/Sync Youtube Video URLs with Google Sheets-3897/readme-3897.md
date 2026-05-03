Sync Youtube Video URLs with Google Sheets

https://n8nworkflows.xyz/workflows/sync-youtube-video-urls-with-google-sheets-3897


# Sync Youtube Video URLs with Google Sheets

### 1. Workflow Overview

This workflow automates the synchronization of YouTube video URLs with a Google Sheet, serving as the foundational step in a multi-part YouTube comment sentiment analysis system. It reads YouTube channel IDs from a Google Sheet, fetches the latest videos from each channel via the YouTube Data API, extracts relevant video metadata, and appends or updates this data in another Google Sheet tab. This structured data output enables subsequent workflows to perform detailed comment sentiment analysis and dashboarding.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Channel IDs Retrieval:** Reads YouTube channel IDs from a specified Google Sheet tab.
- **1.3 Video Data Fetching:** For each channel ID, fetches recent videos using YouTube API with pagination support.
- **1.4 Data Formatting:** Extracts and formats video metadata (title, URL, publish date) for Google Sheets.
- **1.5 Data Insertion:** Appends or updates the formatted video data into a designated Google Sheet tab.
- **1.6 Utility:** Splits the array of videos into individual items for processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow manually for testing or on-demand execution.
- **Nodes Involved:**  
  - Manual Trigger (When Clicking 'Test workflow')
- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Starts the workflow execution manually.  
  - **Configuration:** No parameters; simply a trigger node.  
  - **Input/Output:** No input; outputs an empty data object to the next node.  
  - **Edge Cases:** None; manual trigger is user-initiated.  
  - **Sub-workflow:** None.

#### 1.2 Channel IDs Retrieval

- **Overview:** Reads a list of YouTube channel IDs from `Sheet3` of the connected Google Sheet using a service account.
- **Nodes Involved:**  
  - Get Youtube Channel Ids from Google Sheet
- **Node Details:**  
  - **Type:** Google Sheets (Read Operation)  
  - **Role:** Fetches rows from `Sheet3` containing YouTube channel IDs.  
  - **Configuration:**  
    - Document ID linked to the Google Sheet named "Youtube Videos Comments".  
    - Sheet name set to `Sheet3` (gid: 1592454760).  
    - Authentication via Google Service Account credential.  
  - **Key Expressions:** None; reads entire sheet data.  
  - **Input/Output:** Input from manual trigger; outputs array of channel ID objects.  
  - **Edge Cases:**  
    - Google API authentication errors (invalid service account or permissions).  
    - Empty or malformed sheet data (missing channelId field).  
  - **Sub-workflow:** None.

#### 1.3 Video Data Fetching

- **Overview:** For each channel ID, fetches up to 50 latest videos using YouTube Data API v3, supports pagination to retrieve all videos.
- **Nodes Involved:**  
  - Get Youtube Video Urls form specific channel  
  - Split Out
- **Node Details:**  
  - **Get Youtube Video Urls form specific channel:**  
    - **Type:** HTTP Request  
    - **Role:** Calls YouTube Data API `/search` endpoint with parameters:  
      - `channelId` dynamically from input JSON (`{{$json.channelId}}`)  
      - `part=snippet`  
      - `order=date` (latest first)  
      - `maxResults=50` (max allowed per request)  
    - **Authentication:** HTTP Query Auth with a generic credential (likely API key).  
    - **Pagination:** Enabled using `nextPageToken` to fetch all pages until no token remains.  
    - **Input/Output:** Input from Google Sheets node; outputs array of video items per channel.  
    - **Edge Cases:**  
      - API quota limits or rate limiting.  
      - Invalid or revoked API key.  
      - Channels with no videos or private videos.  
  - **Split Out:**  
    - **Type:** Split Out  
    - **Role:** Splits the array of videos into individual items for downstream processing.  
    - **Configuration:** Splits on `items` field of the HTTP response.  
    - **Input/Output:** Input from HTTP Request node; outputs individual video JSON objects.  
    - **Edge Cases:** Empty video list results in no output items.

#### 1.4 Data Formatting

- **Overview:** Extracts and formats relevant video metadata (title, URL, publish date) for insertion into Google Sheets.
- **Nodes Involved:**  
  - Format fields as required to save in google sheet
- **Node Details:**  
  - **Type:** Set Node  
  - **Role:** Creates new fields with formatted values:  
    - `Title` from `$json.snippet.title`  
    - `video_urls` constructed as `https://www.youtube.com/watch?v={{ $json.id.videoId }}`  
    - `published_at` from `$json.snippet.publishedAt`  
  - **Input/Output:** Input from Split Out node (individual video items); outputs formatted JSON objects.  
  - **Edge Cases:**  
    - Missing or malformed video ID or snippet data.  
    - Null or empty fields in YouTube API response.

#### 1.5 Data Insertion

- **Overview:** Appends new video data or updates existing entries in `Sheet2` of the Google Sheet, matching on video URL to avoid duplicates.
- **Nodes Involved:**  
  - Insert & Update Youtube Urls in Google Sheet
- **Node Details:**  
  - **Type:** Google Sheets (Append or Update Operation)  
  - **Role:** Writes formatted video data into `Sheet2` (gid: 760258523) of the same Google Sheet.  
  - **Configuration:**  
    - Columns mapped explicitly: `Title`, `video_urls`, `published_at`.  
    - Matching column: `video_urls` to update existing rows or append new ones.  
    - Authentication via Google Service Account credential.  
  - **Input/Output:** Input from Set node; outputs confirmation of write operation.  
  - **Edge Cases:**  
    - Google Sheets API errors (rate limits, permission issues).  
    - Data format mismatches or schema changes in the sheet.  
    - Concurrent writes causing race conditions (unlikely in this linear workflow).  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                              | Node Type            | Functional Role                                | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                                                  |
|--------------------------------------|----------------------|-----------------------------------------------|-----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger (When Clicking 'Test workflow') | Manual Trigger       | Starts workflow manually                       | -                                 | Get Youtube Channel Ids from Google Sheet |                                                                                                                              |
| Get Youtube Channel Ids from Google Sheet | Google Sheets         | Reads YouTube channel IDs from `Sheet3`       | Manual Trigger                    | Get Youtube Video Urls form specific channel |                                                                                                                              |
| Get Youtube Video Urls form specific channel | HTTP Request          | Fetches latest videos for each channel via YouTube API | Get Youtube Channel Ids from Google Sheet | Split Out                         |                                                                                                                              |
| Split Out                            | Split Out             | Splits video list into individual video items | Get Youtube Video Urls form specific channel | Format fields as required to save in google sheet |                                                                                                                              |
| Format fields as required to save in google sheet | Set                   | Formats video metadata for Google Sheets       | Split Out                        | Insert & Update Youtube Urls in Google Sheet |                                                                                                                              |
| Insert & Update Youtube Urls in Google Sheet | Google Sheets         | Appends or updates video data in `Sheet2`     | Format fields as required to save in google sheet | -                                 |                                                                                                                              |
| Sticky Note                         | Sticky Note           | Workflow title and overview note                | -                                 | -                                 | ## Sync Youtube Video Urls with Google Sheets                                                                                |
| Sticky Note1                        | Sticky Note           | Summary note with key workflow steps and link  | -                                 | -                                 | ## I'm a note \n✅ Reads Channel IDs from `Sheet3`  \n📹 Fetches video URLs using YouTube API  \n📄 Writes video URLs to `Sheet2`  \n\n🔁 Output used in:  \n👉 [Part 2 – YouTube Comment Sentiment Analyzer](https://n8n.io/workflows/3855-youtube-comment-sentiment-analyzer-with-google-sheets-and-openai/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `Manual Trigger (When Clicking 'Test workflow')`.  
   - No configuration needed.

2. **Add Google Sheets Node to Read Channel IDs**  
   - Add a **Google Sheets** node named `Get Youtube Channel Ids from Google Sheet`.  
   - Set operation to **Read Rows**.  
   - Configure:  
     - Document ID: Select or enter your Google Sheet ID (e.g., `1xoCVr_mlwn4jFcnJENtrU-_K5nkIytZ8qBXzxMq55n4`).  
     - Sheet Name: Select `Sheet3`.  
     - Authentication: Use a **Google Service Account** credential with read access to the sheet.  
   - Connect output of Manual Trigger to this node.

3. **Add HTTP Request Node to Fetch Videos**  
   - Add an **HTTP Request** node named `Get Youtube Video Urls form specific channel`.  
   - Configure:  
     - HTTP Method: GET  
     - URL: `https://www.googleapis.com/youtube/v3/search`  
     - Query Parameters:  
       - `channelId` = `={{ $json.channelId }}` (dynamic from input)  
       - `part` = `snippet`  
       - `order` = `date`  
       - `maxResults` = `50`  
     - Authentication: Use **HTTP Query Auth** with your YouTube Data API key credential.  
     - Enable Pagination:  
       - Pagination parameter: `pageToken` with value `={{ $response.body.nextPageToken }}`  
       - Pagination complete when `!$response.body.nextPageToken` (no next page token)  
   - Connect output of Google Sheets node (channel IDs) to this node.

4. **Add Split Out Node**  
   - Add a **Split Out** node named `Split Out`.  
   - Configure to split on the field `items`.  
   - Connect output of HTTP Request node to this node.

5. **Add Set Node to Format Data**  
   - Add a **Set** node named `Format fields as required to save in google sheet`.  
   - Add fields:  
     - `Title` = `={{ $json.snippet.title }}`  
     - `video_urls` = `=https://www.youtube.com/watch?v={{ $json.id.videoId }}`  
     - `published_at` = `={{ $json.snippet.publishedAt }}`  
   - Connect output of Split Out node to this node.

6. **Add Google Sheets Node to Append/Update Video Data**  
   - Add a **Google Sheets** node named `Insert & Update Youtube Urls in Google Sheet`.  
   - Set operation to **Append or Update**.  
   - Configure:  
     - Document ID: Same as above (your Google Sheet).  
     - Sheet Name: `Sheet2`.  
     - Columns Mapping:  
       - `Title` mapped to `Title` field  
       - `video_urls` mapped to `video_urls` field  
       - `published_at` mapped to `published_at` field  
     - Matching Columns: `video_urls` (to avoid duplicates)  
     - Authentication: Use the same Google Service Account credential.  
   - Connect output of Set node to this node.

7. **(Optional) Add Sticky Notes**  
   - Add sticky notes for documentation and overview as desired.

8. **Connect All Nodes in Sequence**  
   - Manual Trigger → Get Youtube Channel Ids from Google Sheet → Get Youtube Video Urls form specific channel → Split Out → Format fields as required to save in google sheet → Insert & Update Youtube Urls in Google Sheet.

9. **Credential Setup**  
   - Ensure you have:  
     - Google Service Account credentials with access to the target Google Sheet.  
     - YouTube Data API key credential configured for HTTP Query Auth.

10. **Test Workflow**  
    - Run the manual trigger to verify the workflow fetches channel IDs, retrieves videos, formats, and updates the Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                                             |
|------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is Part 1 of a 2-part system; Part 2 performs YouTube comment sentiment analysis using OpenAI.     | [Part 2 – YouTube Comment Sentiment Analyzer](https://n8n.io/workflows/3855-youtube-comment-sentiment-analyzer-with-google-sheets-and-openai/) |
| Use a time-based trigger to automate regular fetching of new videos.                                             | Suggested customization in workflow description.                                                                                           |
| Google Sheets must have `Sheet3` with channel IDs and `Sheet2` structured to receive video data as per mapping. |                                                                                                                                            |
| YouTube API quota limits may affect large-scale usage; monitor API usage and consider quota increases if needed. |                                                                                                                                            |
| For extended metadata, modify the HTTP Request query parameters and Google Sheets columns accordingly.           |                                                                                                                                            |