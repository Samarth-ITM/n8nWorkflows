YouTube Trends Finding Agent

https://n8nworkflows.xyz/workflows/youtube-trends-finding-agent-7888


# YouTube Trends Finding Agent

---

### 1. Workflow Overview

The **YouTube Trends Finder** workflow is designed to automate the discovery and reporting of trending YouTube videos within a specific niche and recent timeframe. It targets content creators, marketers, and analysts seeking to identify high-engagement videos related to a user-specified topic.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Collects user inputs via a web form specifying the niche/topic and the time range (number of days) for trend analysis.
- **1.2 Date Calculation & Spreadsheet Initialization**: Converts user input into a YouTube API-compatible date filter and creates a new Google Spreadsheet to store results.
- **1.3 Video Retrieval & Data Enrichment**: Searches YouTube for relevant videos, fetches detailed video statistics, and calculates engagement rates.
- **1.4 Filtering & Deduplication**: Applies thresholds on engagement rate, views, upload time, and title content to filter videos, and removes duplicates.
- **1.5 Data Formatting & Appending**: Formats filtered video data and appends it as rows into the created spreadsheet.
- **1.6 Result Delivery**: Sends a completion message with a link to the generated Google Spreadsheet back to the user via the form interface.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures user inputs "Topic Name" and "Last How Many Days?" through a form, triggering the workflow upon submission.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Role: Starts workflow on form completion  
    - Configuration:  
      - Form Title: "Find YT Trends in Your Niche"  
      - Fields:  
        - "Topic Name" (required, text)  
        - "Last How Many Days?" (required, number)  
      - Response mode: "lastNode" (sends response after last node)  
      - Webhook ID configured for external form submission  
    - Inputs: HTTP request from form webhook  
    - Outputs: JSON object with user inputs  
    - Edge Cases: Missing required fields, malformed input, form submission failures

#### 1.2 Date Calculation & Spreadsheet Initialization

- **Overview:**  
  Calculates the ISO datetime string for filtering videos published after the specified number of days ago; creates a new Google Spreadsheet titled based on user inputs.

- **Nodes Involved:**  
  - Edit Fields  
  - Create spreadsheet

- **Node Details:**  
  - **Edit Fields**  
    - Type: Set  
    - Role: Calculate "Published After" ISO timestamp for YouTube API  
    - Configuration: Uses expression to subtract user input days from current date and formats as ISO datetime string  
    - Inputs: Output from form submission  
    - Outputs: JSON with "Published After" field  
    - Edge Cases: Invalid date calculation if user inputs zero or negative days

  - **Create spreadsheet**  
    - Type: Google Sheets (Spreadsheet)  
    - Role: Creates a new Google Spreadsheet to collect trending videos  
    - Configuration: Title dynamically built from form inputs (Topic and Time range)  
    - Inputs: Output from Edit Fields  
    - Outputs: JSON containing spreadsheet ID for further operations  
    - Credential: Requires Google Sheets OAuth2 credentials  
    - Edge Cases: API quota limits, permission errors, title conflicts  

#### 1.3 Video Retrieval & Data Enrichment

- **Overview:**  
  Searches YouTube for videos matching the topic, fetches detailed statistics for each video, and computes engagement rates.

- **Nodes Involved:**  
  - Get many videos  
  - Get Data  
  - Code  
  - Engagement Rate Check

- **Node Details:**  
  - **Get many videos**  
    - Type: YouTube node  
    - Role: Retrieve a list of videos relevant to the topic, published after calculated date  
    - Configuration:  
      - Search query: user input "Topic Name"  
      - Region: US (modifiable)  
      - Published after: dynamic ISO date from Edit Fields  
      - Order: relevance  
      - Safe search: moderate  
      - Returns all videos matching criteria  
    - Inputs: Output from Create spreadsheet  
    - Outputs: List of video IDs and snippets  
    - Credential: YouTube OAuth2 required  
    - Edge Cases: API limits, no results found, network failures  

  - **Get Data**  
    - Type: HTTP Request  
    - Role: Fetch detailed video information (contentDetails, snippet, statistics) for each video ID from Get many videos  
    - Configuration:  
      - URL: YouTube Data API v3 videos endpoint  
      - Query parameters: API key (replace placeholder), part parameters, video ID from previous node  
    - Inputs: Output from Get many videos  
    - Outputs: Detailed video JSON data  
    - Edge Cases: API key invalid, request throttling, missing data  

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Calculate engagement rate per video: ((likes + comments) / views) * 100  
    - Configuration: Custom JavaScript iterating over input items  
    - Inputs: Output from Get Data  
    - Outputs: Array of JSON objects including video id, title, engagement rate (2 decimals), views, likes, comments  
    - Edge Cases: Missing statistics fields, zero views causing division by zero, malformed data  

  - **Engagement Rate Check**  
    - Type: If  
    - Role: Filters videos with engagementRate >= 2% (threshold)  
    - Inputs: Output from Code node  
    - Outputs: Pass (engagementRate >= 2), Fail (otherwise)  
    - Edge Cases: Engagement rate missing or non-numeric, threshold tuning required  

#### 1.4 Filtering & Deduplication

- **Overview:**  
  Applies additional filters on video views, upload time range, and title content; removes duplicate video URLs.

- **Nodes Involved:**  
  - Data Format  
  - If  
  - Remove Duplicates  
  - No Operation, do nothing1  
  - No Operation, do nothing

- **Node Details:**  
  - **Data Format**  
    - Type: Set  
    - Role: Formats video data for filtering and output  
    - Configuration: Sets fields like Video Title, Video URL, Engagement Rate, Views, Likes, Comments, and calculates "Uploaded Hours Ago"  
    - Inputs: Pass output from Engagement Rate Check  
    - Outputs: Enhanced JSON for filtering  
    - Edge Cases: Errors in date calculation for hours ago, missing fields  

  - **If**  
    - Type: If  
    - Role: Filters videos by:  
      - Views > 1000  
      - Uploaded Hours Ago <= (User input days * 24)  
      - Video title does NOT contain "#" (to exclude hashtag-heavy titles)  
    - Inputs: Output from Data Format  
    - Outputs: Pass (meets all conditions), Fail (otherwise)  
    - Edge Cases: Title containing special characters, views or upload time missing or zero  

  - **Remove Duplicates**  
    - Type: Remove Duplicates  
    - Role: Ensures unique video URLs in the dataset  
    - Configuration: Compares by "Video URL" field  
    - Inputs: Pass output from If node  
    - Outputs: Unique list of filtered videos  
    - Edge Cases: Inconsistent URLs, edge cases with different URL formats  

  - **No Operation, do nothing1** and **No Operation, do nothing**  
    - Type: No Op  
    - Role: Placeholder or fallback nodes for failed conditions or unused branches  
    - Edge Cases: None  

#### 1.5 Data Formatting & Appending

- **Overview:**  
  Appends filtered and deduplicated video data as rows into the created Google Spreadsheet.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**  
  - **Append row in sheet**  
    - Type: Google Sheets (Row Append)  
    - Role: Insert video data rows into the first sheet of the created spreadsheet  
    - Configuration:  
      - Maps video data fields (Title, URL, Views, Likes, Comments, Uploaded date) automatically  
      - Document ID linked from Create spreadsheet node  
      - Sheet name: first sheet by GID=0  
      - Cell format: USER_ENTERED (to respect Google Sheets data types)  
    - Inputs: Output from Remove Duplicates  
    - Credential: Google Sheets OAuth2 required  
    - Edge Cases: API limits, sheet permissions, data format mismatches  

#### 1.6 Result Delivery

- **Overview:**  
  Generates a completion message with a link to the generated spreadsheet and sends it back to the user via the form interface.

- **Nodes Involved:**  
  - Output Format  
  - Form

- **Node Details:**  
  - **Output Format**  
    - Type: Set  
    - Role: Creates a message string containing the Google Spreadsheet URL using the spreadsheetId from earlier  
    - Inputs: Output from Append row in sheet  
    - Outputs: JSON with "OutPut" field containing URL  
    - Edge Cases: Spreadsheet ID missing or invalid  

  - **Form**  
    - Type: Form (completion)  
    - Role: Sends the final response back to the user completing the form with the spreadsheet link  
    - Configuration:  
      - Completion Title: "🤖 Here's Your Analytics Data 🔽"  
      - Completion Message: Dynamic link from "OutPut" field  
      - Webhook ID linked to previous form submission  
    - Inputs: Output from Output Format  
    - Edge Cases: Form webhook communication failures  

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                            | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                          |
|-------------------------|------------------------|--------------------------------------------|---------------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger            | Receives user input (topic, days)           | (Trigger)                 | Edit Fields               | Setup guide covers form setup and input customization                                              |
| Edit Fields             | Set                     | Calculates "Published After" date           | On form submission        | Create spreadsheet        | Setup guide includes date calculation explanation                                                 |
| Create spreadsheet      | Google Sheets (Spreadsheet) | Creates spreadsheet for results             | Edit Fields               | Get many videos           | Setup guide details Google Sheets connection                                                      |
| Get many videos         | YouTube                 | Retrieves videos matching topic & date      | Create spreadsheet        | Get Data                  | Setup guide explains YouTube API credentials and regionCode setting                               |
| Get Data                | HTTP Request            | Fetches detailed video info                  | Get many videos           | Code                      | Setup guide explains API key replacement                                                          |
| Code                    | Code                    | Calculates engagement rate                   | Get Data                  | Engagement Rate Check     |                                                                                                |
| Engagement Rate Check   | If                      | Filters videos by engagement threshold       | Code                      | Data Format, No Operation | Setup guide includes engagement rate threshold                                                   |
| Data Format             | Set                     | Formats video data and calculates hours ago | Engagement Rate Check     | If                        |                                                                                                |
| If                      | If                      | Filters by views, upload time, title content | Data Format               | Remove Duplicates, No Operation 1 | Setup guide describes video filters (views, time, title)                                        |
| Remove Duplicates       | Remove Duplicates        | Removes duplicate video URLs                 | If                        | Append row in sheet       |                                                                                                |
| Append row in sheet     | Google Sheets (Row Append) | Appends video data to spreadsheet            | Remove Duplicates          | Output Format             | Setup guide covers Google Sheets integration                                                     |
| Output Format           | Set                     | Prepares spreadsheet URL message             | Append row in sheet       | Form                      |                                                                                                |
| Form                    | Form (completion)       | Sends completion message with spreadsheet URL | Output Format             | (End)                    |                                                                                                |
| No Operation, do nothing1| No Operation           | Placeholder for filtered out videos          | If (fail branch)          | (End)                     |                                                                                                |
| No Operation, do nothing | No Operation            | Placeholder for videos failing engagement    | Engagement Rate Check (fail)| (End)                    |                                                                                                |
| Sticky Note             | Sticky Note             | Setup Guide and Instructions                  | (None)                   | (None)                    | Contains detailed setup and configuration instructions, including API and OAuth2 linking          |
| Sticky Note3            | Sticky Note             | YouTube Tutorial Link                         | (None)                   | (None)                    | Provides external video tutorial link: https://youtu.be/AIOmu5OwZMs?si=GlCPT5y6YW-yStb5            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n named "YouTube Trends Finder".**

2. **Add a Form Trigger node ("On form submission"):**  
   - Set Form Title: "Find YT Trends in Your Niche"  
   - Add two fields:  
     - "Topic Name" (Text, Required)  
     - "Last How Many Days?" (Number, Required)  
   - Set Response Mode to "lastNode"  
   - Note the webhook URL for external form submission  

3. **Add a Set node ("Edit Fields"):**  
   - Configure a new field "Published After" (string)  
   - Use expression to calculate date:  
     ```
     {{$today.minus({days: $json['Last How Many Days?']}).toFormat('yyyy-MM-dd\'T\'HH:mm:ss')}}
     ```  
   - Connect output of "On form submission" to "Edit Fields"  

4. **Add a Google Sheets node ("Create spreadsheet"):**  
   - Operation: Create Spreadsheet  
   - Title expression:  
     ```
     Trending YouTube Videos on {{$('On form submission').item.json['Topic Name']}} in Last {{$('On form submission').item.json['Last How Many Days?']}} days
     ```  
   - Connect "Edit Fields" output to "Create spreadsheet"  
   - Configure Google Sheets OAuth2 credentials  

5. **Add a YouTube node ("Get many videos"):**  
   - Resource: Video, Operation: Search  
   - Query:  
     ```
     {{$('On form submission').item.json['Topic Name']}}
     ```  
   - Region Code: "US" (customize as needed)  
   - Published After:  
     ```
     {{$('Edit Fields').item.json["Published After"]}}
     ```  
   - Order: Relevance, SafeSearch: Moderate  
   - Return All: true  
   - Connect "Create spreadsheet" output to "Get many videos"  
   - Configure YouTube OAuth2 credentials  

6. **Add HTTP Request node ("Get Data"):**  
   - URL: `https://www.googleapis.com/youtube/v3/videos`  
   - Method: GET  
   - Query Parameters:  
     - key: Your YouTube API Key (replace placeholder)  
     - part: "contentDetails, snippet, statistics"  
     - id: `{{$json.id.videoId}}` (dynamic from previous node)  
   - Connect "Get many videos" output to "Get Data"  

7. **Add Code node ("Code"):**  
   - Paste the JavaScript code to calculate engagement rate:  
     ```javascript
     const items = $input.all();

     const engagementRates = items.map((item) => {
       const video = item.json.items?.[0];
       const stats = video?.statistics || {};

       const likeCount = parseInt(stats.likeCount || 0, 10);
       const commentCount = parseInt(stats.commentCount || 0, 10);
       const viewCount = parseInt(stats.viewCount || 0, 10);

       let engagementRate = 0;
       if (viewCount > 0) {
         engagementRate = ((likeCount + commentCount) / viewCount) * 100;
       }

       return {
         json: {
           id: video?.id || null,
           title: video?.snippet?.title || null,
           engagementRate: engagementRate.toFixed(2),
           viewCount,
           likeCount,
           commentCount
         }
       };
     });

     return engagementRates;
     ```  
   - Connect "Get Data" output to "Code"  

8. **Add If node ("Engagement Rate Check"):**  
   - Condition: Numeric, engagementRate >= 2  
   - Connect "Code" output to "Engagement Rate Check"  

9. **Add Set node ("Data Format"):**  
   - Assign fields:  
     - Video Title: `{{$json.title}}`  
     - Video URL: `https://www.youtube.com/watch?v={{$json.id}}`  
     - Engagement Rate: `{{$json.engagementRate}}`  
     - Views: `{{$json.viewCount}}`  
     - Likes: `{{$json.likeCount}}`  
     - Comments: `{{$json.commentCount}}`  
     - Uploaded Hours Ago: Use JavaScript expression to calculate hours since publish date from "Get Data" node  
       ```javascript
       (() => {
         const published = new Date($('Get Data').item.json.items[0].snippet.publishedAt);
         const now = new Date();
         const diffMs = now - published;
         const diffHours = Math.floor(diffMs / (1000 * 60 * 60));
         return diffHours;
       })()
       ```  
   - Connect "Engagement Rate Check" pass output to "Data Format"  

10. **Add If node ("If"):**  
    - Conditions (all true):  
      - Views > 1000  
      - Uploaded Hours Ago <= (User input days * 24)  
      - Video Title does NOT contain "#"  
    - Connect "Data Format" output to "If"  

11. **Add Remove Duplicates node ("Remove Duplicates"):**  
    - Compare by field: "Video URL"  
    - Connect "If" pass output to "Remove Duplicates"  

12. **Add Google Sheets node ("Append row in sheet"):**  
    - Operation: Append  
    - Document ID: from "Create spreadsheet" node `{{$('Create spreadsheet').item.json.spreadsheetId}}`  
    - Sheet name: GID=0 (default first sheet)  
    - Map columns: Auto-map fields from input (Video Title, Video URL, Views, Likes, Comments, Uploaded)  
    - Connect "Remove Duplicates" output to "Append row in sheet"  
    - Configure Google Sheets OAuth2 credentials  

13. **Add Set node ("Output Format"):**  
    - Create field "OutPut" with value:  
      ```
      https://docs.google.com/spreadsheets/d/{{$('Create spreadsheet').item.json.spreadsheetId}}
      ```  
    - Connect "Append row in sheet" output to "Output Format"  

14. **Add Form node ("Form"):**  
    - Operation: Completion  
    - Completion Title: "🤖 Here's Your Analytics Data 🔽"  
    - Completion Message: `{{$json.OutPut}}`  
    - Connect "Output Format" output to "Form"  
    - Use same webhook ID as "On form submission" to complete the form response  

15. **Add No Operation nodes ("No Operation, do nothing" and "No Operation, do nothing1")**  
    - Connect to fail branches of respective If nodes for graceful failure handling  

16. **Add Sticky Note nodes for documentation and tutorial links as desired.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup guide with detailed instructions on form configuration, YouTube Data API key integration, Google Sheets OAuth2 linking, and filter customization.                                                                           | Sticky Note in workflow covering nodes from form to spreadsheet                                     |
| Video tutorial explaining how to build the YouTube Trends Finder Agent in n8n.                                                                                                                                                      | [YouTube Tutorial](https://youtu.be/AIOmu5OwZMs?si=GlCPT5y6YW-yStb5)                               |
| YouTube Data API documentation for advanced parameter tuning and quotas.                                                                                                                                                            | https://developers.google.com/youtube/v3                                                             |
| Google Sheets API and OAuth2 setup instructions for automated spreadsheet creation and modification.                                                                                                                               | https://developers.google.com/sheets/api                                                            |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---