YouTube Competitor Video Analytics to Google Sheets

https://n8nworkflows.xyz/workflows/youtube-competitor-video-analytics-to-google-sheets-9641


# YouTube Competitor Video Analytics to Google Sheets

### 1. Workflow Overview

This workflow, titled **"YouTube Competitor Video Analytics to Google Sheets"**, automates the process of extracting trending video data from a specified YouTube channel and appends the summarized analytics to a Google Sheet. It is designed for users who want to analyze competitors' video performance metrics such as views, likes, comments, and video descriptions, enabling trend spotting and content strategy optimization.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives user input via a form including the YouTube channel name, video duration filter, and number of videos to fetch.
- **1.2 Channel Identification:** Translates the channel name into the corresponding YouTube Channel ID.
- **1.3 Video Fetching:** Retrieves video IDs from the specified channel filtered by view count and video duration.
- **1.4 Video Data Extraction:** Gathers detailed video metadata and statistics for each video ID.
- **1.5 Performance Calculation and Data Enrichment:** Calculates a custom "performance" metric and creates a descriptive snippet ("hook") from the video description.
- **1.6 Data Output:** Appends the processed video data as rows in a Google Sheets spreadsheet for ongoing tracking and analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block captures user input parameters via a web form to specify the YouTube channel and the scope of data extraction.
- **Nodes Involved:**  
  - On form submission1  
  - Set Parameters1  
  - Sticky Note (Write the Channel name)
  
- **Node Details:**  

  - **On form submission1**  
    - Type: Form Trigger  
    - Role: Entry point that listens for form submissions from users.  
    - Configuration: Form titled "YouTube Channel Analyzer" with fields:  
      - Format (dropdown: short, medium, long)  
      - Number of Videos (number input, required)  
      - Channel Name (text input, required)  
    - Inputs: HTTP request from form submission  
    - Outputs: JSON with form data  
    - Edge cases: Missing or invalid input fields could cause downstream errors. Validation is enforced by required flags.  
    - Sticky Note: "## Write the Channel name"

  - **Set Parameters1**  
    - Type: Set  
    - Role: Normalizes and stores input parameters into workflow variables for reuse.  
    - Configuration: Assigns the API key (placeholder), and copies form inputs into variables: `format`, `videoLimit`, `channel_name`.  
    - Inputs: Output of the form submission node  
    - Outputs: JSON containing all parameters for API calls  
    - Edge cases: API key must be replaced by a valid key; missing or malformed inputs may affect later HTTP requests.

---

#### 2.2 Channel Identification

- **Overview:** Converts the provided YouTube channel name into its official Channel ID via the YouTube Data API.
- **Nodes Involved:**  
  - Get Channel ID1  
  - Extract Channel ID1  
  - Sticky Note1 (Get Channel ID's)

- **Node Details:**  

  - **Get Channel ID1**  
    - Type: HTTP Request  
    - Role: Calls YouTube API `search` endpoint filtered by type=channel and query=channel_name, to find channel details.  
    - Configuration:  
      - URL: https://www.googleapis.com/youtube/v3/search  
      - Query parameters: part=snippet, type=channel, q=channel_name, key=API key  
    - Inputs: Parameterized input from Set Parameters1  
    - Outputs: API JSON response with channel data  
    - Edge cases:  
      - API key invalid or quota exceeded  
      - Channel name not found returns empty items array  
      - Network issues or API timeout

  - **Extract Channel ID1**  
    - Type: Code  
    - Role: Extracts the first Channel ID from the API response JSON.  
    - Configuration: JavaScript code returns `items[0].id.channelId`.  
    - Inputs: Output of Get Channel ID1  
    - Outputs: JSON with `channelId` string  
    - Edge cases: Empty or malformed API response causes undefined errors. Should handle no `items` gracefully.

---

#### 2.3 Video Fetching

- **Overview:** Fetches video IDs from the identified channel, filtered by video duration and sorted by view count.
- **Nodes Involved:**  
  - Get Video IDs1  
  - Extract IDs1  
  - Sticky Note2 (Get Video ID's)

- **Node Details:**  

  - **Get Video IDs1**  
    - Type: HTTP Request  
    - Role: Calls YouTube API `search` endpoint to get videos for the channel.  
    - Configuration:  
      - URL: https://www.googleapis.com/youtube/v3/search  
      - Query parameters include:  
        - part=snippet  
        - maxResults=videoLimit (default 5)  
        - order=viewCount  
        - channelId=extracted channelId  
        - type=video  
        - videoDuration=selected format (short, medium, long)  
        - regionCode=US  
        - key=API key  
    - Inputs: Output from Extract Channel ID1  
    - Outputs: JSON listing videos fitting criteria  
    - Edge cases:  
      - API quota exceeded  
      - No videos found (empty items array)  
      - Invalid channelId

  - **Extract IDs1**  
    - Type: Code  
    - Role: Extracts video IDs from the API response array  
    - Configuration: Iterates over all items and maps to `{ videoid: videoId }` objects  
    - Inputs: Output from Get Video IDs1  
    - Outputs: Array of video ID JSON objects  
    - Edge cases: Handles missing or null video IDs by assigning null; downstream nodes should verify.

---

#### 2.4 Video Data Extraction

- **Overview:** Retrieves detailed metadata and statistics for each video ID.
- **Nodes Involved:**  
  - Get Video Data1  
  - Extract Video Data1  
  - Sticky Note3 (Extract the video data Like Views , Likes , Title & Description)

- **Node Details:**  

  - **Get Video Data1**  
    - Type: HTTP Request  
    - Role: Calls YouTube API `videos` endpoint for detailed video info.  
    - Configuration:  
      - URL: https://www.googleapis.com/youtube/v3/videos  
      - Query parameters:  
        - part=snippet,statistics  
        - id=videoid (from previous node)  
        - key=API key  
    - Inputs: Output from Extract IDs1  
    - Outputs: Detailed JSON for each video  
    - Edge cases:  
      - Invalid videoid or deleted videos may return errors or empty data  
      - API limits or failures

  - **Extract Video Data1**  
    - Type: Code  
    - Role: Extracts relevant fields: channelTitle, title, description, viewCount, likeCount, commentCount, videoURL, thumbnail  
    - Configuration: Parses the JSON response, maps fields, includes description and thumbnail URL  
    - Inputs: Output from Get Video Data1  
    - Outputs: Structured JSON per video with key metrics  
    - Edge cases: Missing fields default to empty strings or zero; large description text could be truncated downstream.

---

#### 2.5 Performance Calculation and Data Enrichment

- **Overview:** Calculates a custom "performance" metric for each video and extracts a short "hook" snippet from the description for easy reference.
- **Nodes Involved:**  
  - Video Performance1  
  - Hook and Description  
  - Sticky Note (Save Data in a Sheet) (related to next block but visually nearby)

- **Node Details:**  

  - **Video Performance1**  
    - Type: Code  
    - Role: Calculates performance as percentage: ((likes + comments) / views) * 100  
    - Configuration: Parses string counts into integers, safely handles zero views to avoid division by zero  
    - Inputs: Output from Hook and Description (which is downstream of Extract Video Data1)  
    - Outputs: Adds `performance` as a fixed-point string number to JSON  
    - Edge cases: Views=0 or missing stats handled gracefully; very low view counts yield zero or low performance.

  - **Hook and Description**  
    - Type: Code  
    - Role: Extracts the first 15 words of the video description as a "hook" summary  
    - Configuration: Splits description by spaces, joins first 15 words  
    - Inputs: Output from Extract Video Data1  
    - Outputs: JSON with added `hook` field  
    - Edge cases: Empty or very short descriptions handled smoothly; no words results in empty string.

---

#### 2.6 Data Output

- **Overview:** Appends the final enriched video data into a specified Google Sheet for storage and analysis.
- **Nodes Involved:**  
  - Append row in sheet1  
  - Sticky Note4 (Save Data in a Sheet)

- **Node Details:**  

  - **Append row in sheet1**  
    - Type: Google Sheets  
    - Role: Appends a new row with video analytics data to the Google Sheet  
    - Configuration:  
      - Operation: Append  
      - Document ID: Linked Google Sheet ID  
      - Sheet Name: Sheet1 (gid=0)  
      - Columns mapped: hook, title, videoURL, likeCount, thumbnail, viewCount, description, performance, channelTitle, commentCount  
      - Uses OAuth2 credentials linked to "My Account"  
    - Inputs: Output from Video Performance1  
    - Outputs: Confirmation of row append operation  
    - Edge cases:  
      - Google API quota or auth failures  
      - Mismatched schema or missing columns  
      - Network interruptions

---

### 3. Summary Table

| Node Name            | Node Type         | Functional Role                               | Input Node(s)          | Output Node(s)           | Sticky Note                               |
|----------------------|-------------------|-----------------------------------------------|------------------------|--------------------------|-------------------------------------------|
| On form submission1   | Form Trigger      | Capture user input for channel and parameters | -                      | Set Parameters1          | ## Write the Channel name                 |
| Set Parameters1       | Set               | Normalize and assign parameters               | On form submission1     | Get Channel ID1          |                                           |
| Get Channel ID1       | HTTP Request      | Retrieve Channel ID from YouTube API          | Set Parameters1         | Extract Channel ID1       | ## Get Channel ID's                       |
| Extract Channel ID1   | Code              | Extract Channel ID from API response          | Get Channel ID1         | Get Video IDs1           | ## Get Channel ID's                       |
| Get Video IDs1        | HTTP Request      | Fetch video IDs by channel, duration, order  | Extract Channel ID1     | Extract IDs1             | ## Get Video ID's                         |
| Extract IDs1          | Code              | Extract video IDs list                         | Get Video IDs1          | Get Video Data1          | ## Get Video ID's                         |
| Get Video Data1       | HTTP Request      | Fetch detailed video metadata and stats       | Extract IDs1            | Extract Video Data1      | ## Extract the video data Like Views , Likes , Title & Description |
| Extract Video Data1   | Code              | Extract relevant video fields                  | Get Video Data1         | Hook and Description     | ## Extract the video data Like Views , Likes , Title & Description |
| Hook and Description  | Code              | Create a short "hook" from video description  | Extract Video Data1     | Video Performance1       |                                           |
| Video Performance1    | Code              | Calculate performance metric                    | Hook and Description    | Append row in sheet1     | ## Save Data in a Sheet                   |
| Append row in sheet1  | Google Sheets     | Append video data row to Google Sheet          | Video Performance1      | -                        | ## Save Data in a Sheet                   |
| Sticky Note           | Sticky Note       | Instructional text                             | -                      | -                        | ## Write the Channel name                 |
| Sticky Note1          | Sticky Note       | Instructional text                             | -                      | -                        | ## Get Channel ID's                       |
| Sticky Note2          | Sticky Note       | Instructional text                             | -                      | -                        | ## Get Video ID's                         |
| Sticky Note3          | Sticky Note       | Instructional text                             | -                      | -                        | ## Extract the video data Like Views , Likes , Title & Description |
| Sticky Note4          | Sticky Note       | Instructional text                             | -                      | -                        | ## Save Data in a Sheet                   |
| Sticky Note5          | Sticky Note       | Workflow title and purpose                     | -                      | -                        | # Get Viral videos data of your Competitor |
| Sticky Note6          | Sticky Note       | Workflow overview and use cases                | -                      | -                        | # Get Viral Videos Data of Your Competitor ... Tip about OpenAI extension |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Node Type: Form Trigger  
   - Title: "YouTube Channel Analyzer"  
   - Fields:  
     - Dropdown "Format" with options: short, medium, long (required)  
     - Number input "Number of Videos" (required)  
     - Text input "Channel Name" (required)  
   - Position appropriately (e.g., [0, 384])  
   - This node listens to form submissions to start the workflow.

2. **Add Set Node (Set Parameters1)**  
   - Node Type: Set  
   - Assign variables:  
     - `api_key` (string): Set placeholder `<api-key>` (replace with actual YouTube API key)  
     - `format`: Expression `{{$json.Format}}`  
     - `videoLimit`: Expression `{{$json['Number of Videos']}}`  
     - `channel_name`: Expression `{{$json['Channel Name']}}`  
   - Connect output of Form Trigger to this node.

3. **Add HTTP Request Node (Get Channel ID1)**  
   - Node Type: HTTP Request  
   - URL: `https://www.googleapis.com/youtube/v3/search`  
   - Query Parameters:  
     - part = snippet  
     - type = channel  
     - q = `{{$json.channel_name}}`  
     - key = use the stored API key (reference from Set Parameters1)  
   - Method: GET  
   - Connect from Set Parameters1.

4. **Add Code Node (Extract Channel ID1)**  
   - Node Type: Code  
   - JavaScript:  
     ```js
     return [{ json: { channelId: $json.items[0].id.channelId } }];
     ```  
   - Connect from Get Channel ID1.

5. **Add HTTP Request Node (Get Video IDs1)**  
   - Node Type: HTTP Request  
   - URL: `https://www.googleapis.com/youtube/v3/search`  
   - Query Parameters:  
     - part = snippet  
     - maxResults = `{{$json.videoLimit || 5}}`  
     - order = viewCount  
     - channelId = `{{$json.channelId}}`  
     - type = video  
     - videoDuration = `{{$json.format}}`  
     - regionCode = US  
     - key = API key  
   - Method: GET  
   - Connect from Extract Channel ID1 node.

6. **Add Code Node (Extract IDs1)**  
   - Node Type: Code  
   - JavaScript:  
     ```js
     const items = $input.all();
     return items.flatMap(item => (item.json?.items || []).map(videoItem => ({ json: { videoid: videoItem?.id?.videoId || null } })));
     ```  
   - Connect from Get Video IDs1.

7. **Add HTTP Request Node (Get Video Data1)**  
   - Node Type: HTTP Request  
   - URL: `https://www.googleapis.com/youtube/v3/videos`  
   - Query Parameters:  
     - part = snippet,statistics  
     - id = `{{$json.videoid}}`  
     - key = API key  
   - Method: GET  
   - Connect from Extract IDs1.

8. **Add Code Node (Extract Video Data1)**  
   - Node Type: Code  
   - JavaScript:  
     ```js
     const items = $input.all();
     return items.flatMap(item =>
       (item.json?.items || []).map(videoItem => ({
         json: {
           channelTitle: videoItem.snippet?.channelTitle || '',
           title: videoItem.snippet?.title || '',
           description: videoItem.snippet?.description || '',
           viewCount: videoItem.statistics?.viewCount || 0,
           likeCount: videoItem.statistics?.likeCount || 0,
           commentCount: videoItem.statistics?.commentCount || 0,
           videoURL: `https://www.youtube.com/watch?v=${videoItem.id || ''}`,
           thumbnail: videoItem.snippet?.thumbnails?.high?.url || ''
         }
       }))
     );
     ```
   - Connect from Get Video Data1.

9. **Add Code Node (Hook and Description)**  
   - Node Type: Code  
   - JavaScript:  
     ```js
     const items = $input.all();
     return items.map(item => {
       const desc = item.json.description || "";
       const hook = desc.split(" ").slice(0, 15).join(" ");
       return { json: { ...item.json, hook } };
     });
     ```
   - Connect from Extract Video Data1.

10. **Add Code Node (Video Performance1)**  
    - Node Type: Code  
    - JavaScript:  
      ```js
      const items = $input.all();
      return items.map(item => {
        const viewCount = parseInt(item.json.viewCount || '0', 10);
        const likeCount = parseInt(item.json.likeCount || '0', 10);
        const commentCount = parseInt(item.json.commentCount || '0', 10);
        let performance = 0;
        if (viewCount > 0) {
          performance = ((likeCount + commentCount) / viewCount) * 100;
        }
        return { json: { ...item.json, performance: performance.toFixed(2) } };
      });
      ```
    - Connect from Hook and Description.

11. **Add Google Sheets Node (Append row in sheet1)**  
    - Node Type: Google Sheets  
    - Operation: Append  
    - Document ID: Enter your Google Sheet ID (ensure API access)  
    - Sheet Name: "Sheet1" (gid=0)  
    - Columns to append:  
      - hook, title, videoURL, likeCount, thumbnail, viewCount, description, performance, channelTitle, commentCount  
    - Credentials: Configure with OAuth2 Google Sheets credentials ("My Account")  
    - Connect from Video Performance1.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow automatically extracts YouTube competitor video data for analysis by entering a channel name, fetching channel ID, retrieving video IDs and stats, and saving to Google Sheets. It can be extended with OpenAI for content summarization. | Sticky Note6 content at workflow start                                                              |
| The API key placeholders must be replaced with valid YouTube Data API keys, and Google Sheets credentials must be properly configured for data writing.                                                                                           | Credential requirements                                                                             |
| The workflow assumes the Google Sheet has the appropriate columns matching the JSON keys for smooth append operations.                                                                                                                           | Google Sheets mapping details                                                                       |
| For better reliability, add error handling for API quota limits, missing data, or malformed inputs.                                                                                                                                               | Suggested workflow improvement                                                                     |
| To enhance analysis, consider integrating OpenAI nodes to summarize video content or detect trending topics automatically.                                                                                                                       | Mentioned in Sticky Note6                                                                           |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.