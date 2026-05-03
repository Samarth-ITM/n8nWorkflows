Gather Complete YouTube Channel Data to Google Sheets

https://n8nworkflows.xyz/workflows/gather-complete-youtube-channel-data-to-google-sheets-5728


# Gather Complete YouTube Channel Data to Google Sheets

### 1. Workflow Overview

This workflow automates the process of gathering comprehensive data from YouTube channels and storing it into a Google Sheets spreadsheet. It is designed for users who want to collect detailed YouTube channel metrics such as title, description, custom URL, subscriber count, view count, video count, country, keywords, thumbnails, and channel ID for multiple channels efficiently.

The workflow is logically structured into these main blocks:

- **1.1 Input Reception and Filtering:** Reading channel URLs marked as "Ready" from a Google Sheet to determine which channels to process.
- **1.2 URL Type Detection and API Request Routing:** Identifying whether each URL is a full YouTube channel URL or a custom handle URL and directing the request accordingly.
- **1.3 YouTube API Data Fetching:** Making authenticated HTTP requests to the YouTube Data API to retrieve channel information based on the URL type.
- **1.4 Response Validation and Result Updating:** Checking the API response for success and updating the Google Sheet rows with fetched data or error status accordingly.
- **1.5 Iteration Looping:** Processing each eligible channel URL row in batches to handle multiple entries.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

**Overview:**  
This block initiates the workflow manually and fetches channel URLs from the Google Sheet where the status is marked as "ready". It prepares data for processing only for those rows.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Google Sheets - Get Channel URLs  
- Loop Over Items  
- Sticky Note1  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow manually.  
  - *Configuration:* No parameters; triggers workflow execution on demand.  
  - *Inputs:* None  
  - *Outputs:* Triggers Google Sheets node.  
  - *Edge Cases:* None. Reliant on manual start.

- **Google Sheets - Get Channel URLs**  
  - *Type:* Google Sheets  
  - *Role:* Reads channel URLs from the "Channel Urls" tab where status = "ready".  
  - *Configuration:*  
    - Document and sheet IDs linked to a specific Google Sheet template.  
    - Filter applied: `status` column must equal "ready".  
  - *Expressions:* None dynamic, filters on static "ready" value.  
  - *Inputs:* Trigger node output.  
  - *Outputs:* List of rows (items) matching criteria.  
  - *Edge Cases:*  
    - Authentication failures with Google Sheets credentials.  
    - Empty or no rows matching "ready" status.  
  - *Credentials:* Google Sheets OAuth2.  

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each row fetched from Google Sheets individually for processing.  
  - *Configuration:* Defaults, no batch size defined, processes all.  
  - *Inputs:* Output from Google Sheets - Get Channel URLs.  
  - *Outputs:* Single item per iteration to next node.  
  - *Edge Cases:* Handles empty input gracefully by no iteration.  

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Documentation for block 1 explaining reading URLs.  
  - *Content Summary:* Explains reading channel URLs marked "Ready" and looping over them.

---

#### 2.2 URL Type Detection and API Request Routing

**Overview:**  
Determines whether each channel URL is a full channel ID URL or a custom handle URL, then routes the flow to call the appropriate YouTube API endpoint accordingly.

**Nodes Involved:**  
- Switch - Detect URL Type  
- HTTP Request - Get Channel Info By Full URL  
- HTTP Request - Get Channel Info By Custom URL  
- Sticky Note2  

**Node Details:**

- **Switch - Detect URL Type**  
  - *Type:* Switch  
  - *Role:* Uses regex conditions to classify the URL type.  
  - *Configuration:*  
    - Rule 1: If the URL matches `/UC[A-Za-z0-9_-]{22}/` pattern (standard YouTube channel ID format).  
    - Rule 2: If the URL contains a custom handle pattern `/@([^/]+)/`.  
  - *Inputs:* Single channel URL item.  
  - *Outputs:* Two branches: full channel ID or custom URL.  
  - *Edge Cases:*  
    - URLs that do not match either regex will not be processed further.  
    - Regex failures or unexpected URL formats.

- **HTTP Request - Get Channel Info By Full URL**  
  - *Type:* HTTP Request  
  - *Role:* Fetch channel data using YouTube API by channel ID extracted from URL.  
  - *Configuration:*  
    - GET request to `https://www.googleapis.com/youtube/v3/channels` endpoint.  
    - Query parameters: `part=id,snippet,statistics,contentDetails,brandingSettings`.  
    - `id` parameter extracted via regex from channel_url field.  
    - Authentication via YouTube OAuth2 credentials.  
  - *Expressions:* Extract channel ID from URL with regex: `={{ $json.channel_url.match(/channel\/([A-Za-z0-9_-]+)/)[1] || ''}}`.  
  - *Inputs:* Output from Switch node (full URL branch).  
  - *Outputs:* API response with channel info.  
  - *Edge Cases:*  
    - OAuth token expiration or invalid credentials.  
    - API rate limits or quota exceeded.  
    - Missing or malformed channel ID extraction causing empty API requests.

- **HTTP Request - Get Channel Info By Custom URL**  
  - *Type:* HTTP Request  
  - *Role:* Fetch channel data using YouTube API by custom handle (forHandle parameter).  
  - *Configuration:*  
    - GET request to same endpoint as above.  
    - Query parameter: `forHandle` extracted from URL after "@" symbol.  
    - Authentication via YouTube OAuth2 credentials.  
  - *Expressions:* Extract handle via `={{ $json.channel_url.split('@')[1] || ''}}`.  
  - *Inputs:* Output from Switch node (custom URL branch).  
  - *Outputs:* API response with channel info.  
  - *Edge Cases:*  
    - Same as above plus invalid or non-existing handles.

- **Sticky Note2**  
  - *Type:* Sticky Note  
  - *Role:* Documentation for block 2 explaining URL detection and API calls.

---

#### 2.3 Response Validation and Result Updating

**Overview:**  
Validates the API response to determine if data retrieval succeeded. It updates the Google Sheet row with the channel data if successful or marks it as an error otherwise.

**Nodes Involved:**  
- If - Check Success Response  
- Google Sheets - Update Data  
- Google Sheets - Update Data - Error  
- Sticky Note3  

**Node Details:**

- **If - Check Success Response**  
  - *Type:* If  
  - *Role:* Checks if the API response contains more than zero results.  
  - *Configuration:*  
    - Condition: `$json.body.pageInfo.totalResults > 0` (numeric comparison).  
  - *Inputs:* API response from YouTube HTTP Request nodes.  
  - *Outputs:*  
    - True branch: successful response.  
    - False branch: failure or no results.  
  - *Edge Cases:*  
    - API response structure changes can break expression.  
    - Zero results can indicate invalid channel or handle.

- **Google Sheets - Update Data**  
  - *Type:* Google Sheets  
  - *Role:* Updates the corresponding row in the Google Sheet with fetched channel info and marks status as "finish".  
  - *Configuration:*  
    - Document and sheet IDs same as initial read.  
    - Uses `row_number` from initial input to update correct row.  
    - Maps multiple channel data fields: title, country, keywords, channel_id, custom_url, thumbnail URL, view count, description, video count, published date, subscriber count, and last fetched time.  
    - Status column set to "finish".  
  - *Inputs:* True branch from If node.  
  - *Outputs:* Loops back to process next item.  
  - *Edge Cases:*  
    - Google Sheets API failures or permission issues.  
    - Data type mismatches in columns.

- **Google Sheets - Update Data - Error**  
  - *Type:* Google Sheets  
  - *Role:* Updates the corresponding row with status "error" and last fetched time if API call failed.  
  - *Configuration:*  
    - Updates only `status`, `row_number`, and `last_fetched_time` fields.  
    - Marks status as "error".  
  - *Inputs:* False branch from If node.  
  - *Outputs:* Loops back to continue processing.  
  - *Edge Cases:* Same as above.

- **Sticky Note3**  
  - *Type:* Sticky Note  
  - *Role:* Documentation for block 3 explaining results update and status marking.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                   | Input Node(s)                   | Output Node(s)                                  | Sticky Note                                                                                                     |
|-----------------------------------|---------------------|-------------------------------------------------|--------------------------------|------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger      | Starts workflow manually                         | None                           | Google Sheets - Get Channel URLs                |                                                                                                                 |
| Google Sheets - Get Channel URLs   | Google Sheets       | Reads channel URLs with status "ready"          | When clicking ‘Test workflow’  | Loop Over Items                                 |                                                                                                                 |
| Loop Over Items                    | Split In Batches    | Iterates over each channel URL row               | Google Sheets - Get Channel URLs | Switch - Detect URL Type                        |                                                                                                                 |
| Switch - Detect URL Type           | Switch              | Detects URL type: full channel ID or custom URL | Loop Over Items                | HTTP Request - Get Channel Info By Full URL, HTTP Request - Get Channel Info By Custom URL |                                                                                                                 |
| HTTP Request - Get Channel Info By Full URL | HTTP Request       | Fetches channel info by full channel ID URL     | Switch - Detect URL Type       | If - Check Success Response                      |                                                                                                                 |
| HTTP Request - Get Channel Info By Custom URL | HTTP Request       | Fetches channel info by custom handle URL        | Switch - Detect URL Type       | If - Check Success Response                      |                                                                                                                 |
| If - Check Success Response       | If                  | Validates API response success                    | HTTP Request - Get Channel Info nodes | Google Sheets - Update Data (true), Google Sheets - Update Data - Error (false) |                                                                                                                 |
| Google Sheets - Update Data        | Google Sheets       | Updates Google Sheet row with channel data       | If - Check Success Response (true) | Loop Over Items                                 |                                                                                                                 |
| Google Sheets - Update Data - Error | Google Sheets       | Updates Google Sheet row with error status       | If - Check Success Response (false) | Loop Over Items                                 |                                                                                                                 |
| Sticky Note1                      | Sticky Note         | Describes reading channel URLs block             | None                           | None                                           | ## 1. Read Channel URLs From Google Sheets - We’ll start by pulling a list of channel URLs or custom channel URLs from your connected Google Sheet. The loop iterates over each row that's marked as **Ready** in the status **Column A** for processing. |
| Sticky Note2                      | Sticky Note         | Describes URL detection and API calls             | None                           | None                                           | ## 2. Detect Input Type And Fetch Channel Data Using YouTube API - We’ll check whether each entry is a full channel URL or a custom channel URL; then, route it to the appropriate YouTube API call. |
| Sticky Note3                      | Sticky Note         | Describes Google Sheets update with results       | None                           | None                                           | ## 3. Update Google Sheets With Results - If the response is successful, all collected metrics for that channel are inserted into connected Google Sheet. Its row's status in **Column A** is marked as **Finished**. Otherwise, marked as **Error**. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Google Sheets Node to Get Channel URLs**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Document ID: Use your Google Sheets document ID for channel URLs.  
   - Sheet Name: The tab where channel URLs are stored (e.g., "Channel Urls").  
   - Filter: Add filter where `status` column equals "ready".  
   - Credentials: Configure with Google Sheets OAuth2 credentials.

3. **Add Split In Batches Node**  
   - Type: Split In Batches  
   - Purpose: To process each channel URL row individually.

4. **Add Switch Node to Detect URL Type**  
   - Type: Switch  
   - Add two regex rules:  
     - Rule 1: Matches full YouTube channel ID URLs using regex `/UC[A-Za-z0-9_-]{22}/`.  
     - Rule 2: Matches custom handle URLs using regex `/@([^/]+)/`.  
   - Connect Split In Batches output to this node.

5. **Add HTTP Request Node for Full Channel ID URLs**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/channels`  
   - Query Parameters:  
     - `part`: `id,snippet,statistics,contentDetails,brandingSettings`  
     - `id`: Extract channel ID from URL using expression: `{{$json.channel_url.match(/channel\/([A-Za-z0-9_-]+)/)[1] || ''}}`  
   - Authentication: Use YouTube OAuth2 credentials configured in your n8n instance.  
   - Connect first branch of Switch node to this node.

6. **Add HTTP Request Node for Custom Handle URLs**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/channels`  
   - Query Parameters:  
     - `part`: `id,snippet,statistics,contentDetails,brandingSettings`  
     - `forHandle`: Extract handle from URL using expression: `{{$json.channel_url.split('@')[1] || ''}}`  
   - Authentication: Same YouTube OAuth2 credentials.  
   - Connect second branch of Switch node to this node.

7. **Add If Node to Check Success Response**  
   - Type: If  
   - Condition: Numeric greater than check  
   - Expression: `$json.body.pageInfo.totalResults > 0`  
   - Connect outputs of both HTTP Request nodes to this node.

8. **Add Google Sheets Node to Update Data (Success)**  
   - Type: Google Sheets  
   - Operation: Update row  
   - Document ID and Sheet Name: Use same as in step 2.  
   - Mapping Columns: Map all channel info fields from API response, including:  
     - title, status ("finish"), country, keywords, channel_id, custom_url, thumbnail URL, view_count, description, video_count, published_at (formatted), subscriber_count, row_number (from original item), last_fetched_time (current timestamp).  
   - Matching Columns: Use `row_number` to identify the row to update.  
   - Credentials: Google Sheets OAuth2.  
   - Connect true branch of If node to this node.  
   - Connect output back to Split In Batches for next iteration.

9. **Add Google Sheets Node to Update Data (Error)**  
   - Type: Google Sheets  
   - Operation: Update row  
   - Document ID and Sheet Name: Same as above.  
   - Mapping Columns: Only update `status` to "error", `row_number`, and `last_fetched_time`.  
   - Matching Columns: `row_number`.  
   - Credentials: Google Sheets OAuth2.  
   - Connect false branch of If node to this node.  
   - Connect output back to Split In Batches for next iteration.

10. **Connect final loop back to Split In Batches**  
    - This ensures the workflow continues to the next item.

11. **Set up Credentials**  
    - For Google Sheets nodes: configure OAuth2 credentials with access to your Google Sheet.  
    - For HTTP Request nodes: configure YouTube OAuth2 credentials with permissions for YouTube Data API.

12. **Run Workflow**  
    - Populate your Google Sheet with channel URLs or custom handles.  
    - Set status column to "ready" for rows to process.  
    - Trigger workflow manually or configure a time trigger as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is provided by Agent Circle and demonstrates a practical use case for collecting YouTube channel data automatically into Google Sheets, suitable for MCNs, agencies, analysts, and consultants.                                                                                                                                                                                                                                                                                                                                                                                           | https://www.agentcircle.ai/                                                                                                         |
| For best results, duplicate the provided Google Sheets template: [YouTube - Get Channel Information Google Sheet Template](https://docs.google.com/spreadsheets/d/1easnNMrm8ovxhlZQwPUge6UbPnUVFKBeaQY5EmmG1gM/edit#gid=426418282)                                                                                                                                                                                                                                                                                                                                                                                  | Google Sheets Template Link                                                                                                         |
| YouTube API credentials require OAuth2 setup with enabled YouTube Data API v3 access. Google Sheets credentials require OAuth2 with write permission on the target spreadsheet.                                                                                                                                                                                                                                                                                                                                                                                                           | Google Cloud Console - OAuth2 setup instructions                                                                                     |
| To automate workflow execution, consider adding a Google Sheets trigger that reacts to new or updated rows marked "ready". This allows scheduled or event-driven data collection.                                                                                                                                                                                                                                                                                                                                                                                                           | N8N Google Sheets Trigger Node                                                                                                      |
| Community and support channels for Agent Circle include Discord, Facebook groups, YouTube, LinkedIn, and social media for additional help and updates.                                                                                                                                                                                                                                                                                                                                                                                                                                   | Discord: https://discord.gg/d8SkCzKwnP, YouTube: https://www.youtube.com/@agentcircle, LinkedIn: https://www.linkedin.com/company/agentcircle |

---

**Disclaimer:**  
The provided content is derived exclusively from an automated workflow created with n8n, respecting all applicable content policies and handling only legal and public data.