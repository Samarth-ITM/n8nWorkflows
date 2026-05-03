Extract YouTube Channel Videos to Google Sheets with Metadata Tracking

https://n8nworkflows.xyz/workflows/extract-youtube-channel-videos-to-google-sheets-with-metadata-tracking-5767


# Extract YouTube Channel Videos to Google Sheets with Metadata Tracking

---

## 1. Workflow Overview

This workflow automates the extraction of videos from specified YouTube channels and logs detailed metadata into a Google Sheet for tracking and analysis purposes. It is designed for users who want to monitor YouTube channel content systematically, such as marketers, content strategists, researchers, or automation professionals.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**  
  Reads channel URLs or IDs from a Google Sheet, filtering for entries marked "Ready" to process.

- **1.2 Input Normalization and Channel ID Resolution**  
  Determines whether the input is a direct channel ID, a full channel URL, or a custom handle, and converts all inputs into canonical YouTube channel IDs via API calls if needed.

- **1.3 Video Retrieval via YouTube API**  
  Fetches a list of videos from the channel using the YouTube Data API, with a configurable video count limit.

- **1.4 Response Validation and Data Storage**  
  Checks API success, then splits, cleans, and writes video metadata to a "Videos" tab in Google Sheets. Updates the status of each channel row depending on success or failure.

- **1.5 Error Handling and Status Updates**  
  Marks failed channel rows explicitly for follow-up.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by reading the list of YouTube channel inputs from the Google Sheet, selecting only those rows marked with status "Ready". This ensures that only channels queued for processing are fetched.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Google Sheets - Get Channel URLs  
- Loop (SplitInBatches)  
- Wait

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual user interaction.  
  - Configuration: Default; no parameters.  
  - Connections: Outputs to Google Sheets - Get Channel URLs.  
  - Potential Failures: None typical; triggered manually.

- **Google Sheets - Get Channel URLs**  
  - Type: Google Sheets  
  - Role: Reads rows from the "Channel URLs" tab where `status` equals `"ready"`.  
  - Configuration: Filters rows by status column; reads from a specific Google Sheet document and tab.  
  - Credentials: Google Sheets OAuth2 with appropriate access.  
  - Connections: Outputs to Loop node.  
  - Potential Failures: Authentication errors, API quota limits, empty or malformed data.

- **Loop (SplitInBatches)**  
  - Type: SplitInBatches  
  - Role: Processes channel URLs row-by-row to avoid API rate limits.  
  - Configuration: Default batch size (implicit).  
  - Connections: Outputs to Switch - Detect Channel ID or Channel URL and Wait.  
  - Potential Failures: Empty batch, slow processing if batch size is too small.

- **Wait**  
  - Type: Wait  
  - Role: Introduces a short delay (0.01 minutes ~ 0.6 seconds) to manage API rate limits or pacing.  
  - Connections: Loops back to Loop node for next batch.  
  - Potential Failures: Timeout misconfigurations.

---

### 2.2 Input Normalization and Channel ID Resolution

**Overview:**  
This block detects the input type for each channel entry and normalizes it to a canonical channel ID. It uses regex rules to distinguish among direct channel IDs, full URLs, custom URLs, or handles. If needed, it calls the YouTube API to convert usernames or handles into channel IDs.

**Nodes Involved:**  
- Switch - Detect Channel ID or Channel URL  
- Fields - Set Channel ID 1  
- Fields - Set Channel Username  
- HTTP Request - Get Channel ID  
- Fields - Set Channel ID 2

**Node Details:**

- **Switch - Detect Channel ID or Channel URL**  
  - Type: Switch  
  - Role: Routes input based on regex pattern matching the `channel_url` field.  
  - Configuration:  
    - Conditions include:  
      - Full channel URL format: `https://www.youtube.com/channel/CHANNEL_ID`  
      - Channel ID format: `UC` followed by 22 characters.  
      - Custom URL or handle formats: `https://www.youtube.com/@username` or `@username` or just `username`.  
  - Connections:  
    - Direct channel ID → Fields - Set Channel ID 1  
    - Full channel URL → Fields - Set Channel ID 1  
    - Custom URL or handle → Fields - Set Channel Username  
  - Potential Failures: Incorrect input formats not matching any condition; could cause misrouting.

- **Fields - Set Channel ID 1**  
  - Type: Set  
  - Role: Extracts and assigns `channel_id` directly from `channel_url` using regex.  
  - Configuration: Uses regex to extract channel ID pattern from the string.  
  - Connections: Outputs to HTTP Request - Get Channel Videos.  
  - Potential Failures: Empty or malformed channel_url causing empty channel_id.

- **Fields - Set Channel Username**  
  - Type: Set  
  - Role: Extracts `channel_username` by stripping URL prefixes and '@' symbols from the `channel_url`.  
  - Connections: Outputs to HTTP Request - Get Channel ID.  
  - Potential Failures: Unexpected URL formats causing incorrect username extraction.

- **HTTP Request - Get Channel ID**  
  - Type: HTTP Request  
  - Role: Calls YouTube API `channels` endpoint with `forHandle` query parameter to get channel ID from username.  
  - Configuration:  
    - Method: GET  
    - Parameters: `part=id`, `forHandle={{ $json.channel_username }}`  
    - Authentication: OAuth2 for YouTube API.  
  - Connections: Outputs to Fields - Set Channel ID 2.  
  - Potential Failures: API quota exceeded, invalid username, network errors, authentication failure.

- **Fields - Set Channel ID 2**  
  - Type: Set  
  - Role: Extracts `channel_id` from API response JSON path `body.items[0].id`.  
  - Connections: Outputs to HTTP Request - Get Channel Videos.  
  - Potential Failures: Missing or empty items array in API response causing empty channel_id.

---

### 2.3 Video Retrieval via YouTube API

**Overview:**  
Using the resolved channel ID, this block requests the list of videos from that channel’s public uploads. The number of videos fetched is controlled by a limit specified in the Google Sheet or defaults to 10.

**Nodes Involved:**  
- HTTP Request - Get Channel Videos

**Node Details:**

- **HTTP Request - Get Channel Videos**  
  - Type: HTTP Request  
  - Role: Calls YouTube API `search` endpoint to list videos of the specified channel.  
  - Configuration:  
    - Method: GET  
    - Query parameters:  
      - `part=snippet` (fetches video metadata)  
      - `channelId={{ $json.channel_id }}`  
      - `order=date` (newest first)  
      - `type=video`  
      - `maxResults={{ $('Google Sheets - Get Channel URLs').item.json.limit || 10 }}` (limit from sheet or 10 default)  
    - Authentication: OAuth2 YouTube API credential.  
  - Connections: Outputs to If - Check Success Response.  
  - Potential Failures: API rate limit, network issues, invalid channel ID, quota exceeded.

---

### 2.4 Response Validation and Data Storage

**Overview:**  
Checks if the API response contains videos. If successful, splits video items and appends or updates them into the "Videos" tab in Google Sheets. Also updates the channel row status to "finish" with timestamp. If failure, marks status as "error".

**Nodes Involved:**  
- If - Check Success Response  
- Split Out - Videos  
- Google Sheets - Update Data  
- Google Sheets - Update Data - Success  
- Google Sheets - Update Data - Error

**Node Details:**

- **If - Check Success Response**  
  - Type: If  
  - Role: Validates that the API response has `pageInfo.totalResults > 0` to confirm videos were found.  
  - Connections:  
    - True branch → Split Out - Videos and Google Sheets - Update Data - Success  
    - False branch → Google Sheets - Update Data - Error  
  - Potential Failures: Misinterpretation if API response structure changes.

- **Split Out - Videos**  
  - Type: SplitOut  
  - Role: Splits the array `body.items` into individual video items for processing.  
  - Connections: Outputs to Google Sheets - Update Data.  
  - Potential Failures: Empty or malformed `items` field.

- **Google Sheets - Update Data**  
  - Type: Google Sheets  
  - Role: Appends or updates video metadata into the "Videos" tab.  
  - Configuration:  
    - Columns mapped: title, video_url (constructed from videoId), thumbnails, channel_url (from channel URLs sheet), description, published_at (formatted).  
    - Matching column for update: video_url.  
  - Credentials: Google Sheets OAuth2 with access to the target spreadsheet.  
  - Potential Failures: Authentication, API limits, data conversion issues.

- **Google Sheets - Update Data - Success**  
  - Type: Google Sheets  
  - Role: Updates the original channel URL row’s status to "finish" and sets last_fetched_time timestamp.  
  - Configuration: Matches on row_number to update the channel URL row in "Channel URLs" tab.  
  - Connections: None (end).  
  - Potential Failures: Incorrect row_number leading to wrong updates.

- **Google Sheets - Update Data - Error**  
  - Type: Google Sheets  
  - Role: Updates the original channel URL row’s status to "error" and sets last_fetched_time timestamp on failure.  
  - Configuration: Matches on row_number in "Channel URLs" tab.  
  - Connections: None (end).  
  - Potential Failures: Same as above.

---

## 3. Summary Table

| Node Name                      | Node Type            | Functional Role                                   | Input Node(s)                          | Output Node(s)                              | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
|--------------------------------|----------------------|-------------------------------------------------|--------------------------------------|---------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger       | Starts the workflow manually                      | —                                    | Google Sheets - Get Channel URLs             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Google Sheets - Get Channel URLs| Google Sheets        | Reads channel URLs with status "ready"            | When clicking ‘Test workflow’         | Loop                                         | See Sticky Note1: Explains reading channel URLs/IDs from Google Sheets filtered by status "Ready".                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Loop                           | SplitInBatches       | Processes channel URLs individually in batches   | Google Sheets - Get Channel URLs      | Switch - Detect Channel ID or Channel URL, Wait | See Sticky Note1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Wait                           | Wait                 | Adds delay between batch processing               | Loop                                 | Loop                                         | See Sticky Note1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Switch - Detect Channel ID or Channel URL | Switch               | Detects input type and routes accordingly         | Loop                                 | Fields - Set Channel ID 1, Fields - Set Channel Username | See Sticky Note2: Details detection and normalization of input types (channel ID, full URL, or username).                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Fields - Set Channel ID 1       | Set                  | Extracts channel ID from channel URL or ID        | Switch - Detect Channel ID or Channel URL | HTTP Request - Get Channel Videos             | See Sticky Note2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Fields - Set Channel Username   | Set                  | Extracts username from custom or full channel URL | Switch - Detect Channel ID or Channel URL | HTTP Request - Get Channel ID                  | See Sticky Note2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| HTTP Request - Get Channel ID   | HTTP Request         | Resolves username to channel ID using YouTube API | Fields - Set Channel Username          | Fields - Set Channel ID 2                      | See Sticky Note2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Fields - Set Channel ID 2       | Set                  | Extracts channel ID from API response              | HTTP Request - Get Channel ID          | HTTP Request - Get Channel Videos             | See Sticky Note2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| HTTP Request - Get Channel Videos| HTTP Request         | Fetches channel videos metadata                     | Fields - Set Channel ID 1, Fields - Set Channel ID 2 | If - Check Success Response                    | See Sticky Note4: Fetches videos using YouTube API with limit per channel.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| If - Check Success Response     | If                   | Validates API response success                      | HTTP Request - Get Channel Videos      | Split Out - Videos (Success), Google Sheets - Update Data - Error (Failure) | See Sticky Note3: Checks API response; routes to success or error update nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Split Out - Videos              | SplitOut             | Splits video list into individual items             | If - Check Success Response            | Google Sheets - Update Data                    | See Sticky Note3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Google Sheets - Update Data     | Google Sheets        | Appends/updates video data to "Videos" tab          | Split Out - Videos                     | —                                             | See Sticky Note3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Google Sheets - Update Data - Success | Google Sheets        | Updates "Channel URLs" row status to "finish" on success | If - Check Success Response (true)     | —                                             | See Sticky Note3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Google Sheets - Update Data - Error   | Google Sheets        | Updates "Channel URLs" row status to "error" on failure | If - Check Success Response (false)    | —                                             | See Sticky Note3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Sticky Note                    | Sticky Note          | Contains detailed workflow description and usage   | —                                    | —                                             | Please refer to the full Sticky Note node content in the JSON for extensive workflow explanation and usage instructions.                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Sticky Note1                   | Sticky Note          | Explains block 1 about reading channel URLs         | —                                    | —                                             | Describes reading channel URLs from Google Sheets where status = "ready".                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Sticky Note2                   | Sticky Note          | Explains block 2 about input detection and normalization | —                                    | —                                             | Describes detection of input types and normalization to channel IDs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Sticky Note3                   | Sticky Note          | Explains block 4 about API response checking and updating | —                                    | —                                             | Describes success/error status updates and video data storage in Google Sheets.                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Sticky Note4                   | Sticky Note          | Explains block 3 about fetching videos from YouTube API | —                                    | —                                             | Describes fetching videos using YouTube API with video count limits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Node Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No special configuration.

2. **Add Google Sheets Node to Get Channel URLs**  
   - Node Name: `Google Sheets - Get Channel URLs`  
   - Operation: Read Rows  
   - Sheet Name: "Channel Urls" tab (sheet ID 426418282)  
   - Document ID: Your Google Sheet ID (e.g., `1GIdiUUx1PtEZXOzSUP3aJkrDaVcJdxCGRmCOT94BytA`)  
   - Filter: Only rows where `status` column equals `"ready"`  
   - Credentials: Google Sheets OAuth2 with read access.

3. **Add SplitInBatches Node**  
   - Node Name: `Loop`  
   - Operation: Default (splitting input into batches)  
   - Connect output of Google Sheets - Get Channel URLs to this node.

4. **Add Wait Node**  
   - Node Name: `Wait`  
   - Parameters: Delay for 0.01 minutes (about 0.6 seconds)  
   - Connect output of `Loop` to `Wait`  
   - Connect output of `Wait` back to `Loop` to process next batch.

5. **Add Switch Node to Detect Input Type**  
   - Node Name: `Switch - Detect Channel ID or Channel URL`  
   - Rules: Use regex conditions on `channel_url` field to detect:  
     - Full channel URL: `^https:\/\/www\.youtube\.com\/channel\/[A-Za-z0-9_-]+\/?$`  
     - Channel ID format: `^UC[\w-]{22}$`  
     - Custom URL: `^https:\/\/www\.youtube\.com\/@[\w.-]+$` or `^@[\w.-]+$` or `^[\w.-]+$`  
   - Connect output of `Loop` to this node.

6. **Add Set Node to Extract Channel ID (Direct)**  
   - Node Name: `Fields - Set Channel ID 1`  
   - Assign variable `channel_id` using regex extraction:  
     - `={{ $json.channel_url.match(/UC[\w-]{22}/)?.[0] || '' }}`  
   - Connect Switch outputs for full channel URL and channel ID to this node.

7. **Add Set Node to Extract Channel Username**  
   - Node Name: `Fields - Set Channel Username`  
   - Assign variable `channel_username` by stripping URL prefixes and '@':  
     - `={{ $json.channel_url.replace(/^https?:\/\/(www\.)?youtube\.com\/@/, '').replace(/^@/, '') || '' }}`  
   - Connect Switch outputs for custom URLs and handles to this node.

8. **Add HTTP Request Node to Get Channel ID from Username**  
   - Node Name: `HTTP Request - Get Channel ID`  
   - Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/channels`  
   - Query Parameters:  
     - `part=id`  
     - `forHandle={{ $json.channel_username }}`  
   - Authentication: YouTube OAuth2 credentials with access to YouTube Data API.  
   - Connect output of `Fields - Set Channel Username` to this node.

9. **Add Set Node to Extract Channel ID from API Response**  
   - Node Name: `Fields - Set Channel ID 2`  
   - Assign variable `channel_id`:  
     - `={{ $json.body.items[0].id }}`  
   - Connect output of `HTTP Request - Get Channel ID` to this node.

10. **Add HTTP Request Node to Get Channel Videos**  
    - Node Name: `HTTP Request - Get Channel Videos`  
    - Method: GET  
    - URL: `https://www.googleapis.com/youtube/v3/search`  
    - Query Parameters:  
      - `part=snippet`  
      - `channelId={{ $json.channel_id }}`  
      - `order=date`  
      - `type=video`  
      - `maxResults={{ $('Google Sheets - Get Channel URLs').item.json.limit || 10 }}`  
    - Authentication: YouTube OAuth2 credentials  
    - Connect outputs of `Fields - Set Channel ID 1` and `Fields - Set Channel ID 2` to this node.

11. **Add If Node to Check API Success**  
    - Node Name: `If - Check Success Response`  
    - Condition: `{{ $json.body.pageInfo.totalResults > 0 }}` (number greater than 0)  
    - Connect output of `HTTP Request - Get Channel Videos` to this node.

12. **Add SplitOut Node to Split Video Items**  
    - Node Name: `Split Out - Videos`  
    - Field to Split Out: `body.items`  
    - Connect "true" output of If node to this node.

13. **Add Google Sheets Node to Update Video Data**  
    - Node Name: `Google Sheets - Update Data`  
    - Operation: Append or Update  
    - Sheet Name: "Videos" tab (sheet ID 2020351526)  
    - Document ID: Same as above  
    - Columns mapping:  
      - `title` = `{{ $json.snippet.title }}`  
      - `video_url` = `https://www.youtube.com/watch?v={{ $json.id.videoId }}`  
      - `thumbnails` = `{{ $json.snippet.thumbnails.default.url }}`  
      - `channel_url` = `{{ $('Google Sheets - Get Channel URLs').item.json.channel_url }}`  
      - `description` = `{{ $json.snippet.description }}`  
      - `published_at` = `{{ $json.snippet.publishedAt.toString().slice(0, 19).replace('T', ' ') }}`  
    - Matching Columns: `video_url`  
    - Credentials: Google Sheets OAuth2 with write access  
    - Connect output of `Split Out - Videos` to this node.

14. **Add Google Sheets Node to Update Channel Row on Success**  
    - Node Name: `Google Sheets - Update Data - Success`  
    - Operation: Update  
    - Sheet Name: "Channel Urls" tab  
    - Document ID: Same as above  
    - Columns mapping:  
      - `status` = `"finish"`  
      - `row_number` = `{{ $('Loop').item.json.row_number }}`  
      - `last_fetched_time` = `{{ $now.toISO().toString().slice(0, 19).replace('T', ' ') }}`  
    - Matching Columns: `row_number`  
    - Credentials: Google Sheets OAuth2  
    - Connect "true" output of If node (parallel to SplitOut) to this node.

15. **Add Google Sheets Node to Update Channel Row on Error**  
    - Node Name: `Google Sheets - Update Data - Error`  
    - Operation: Update  
    - Sheet Name: "Channel Urls" tab  
    - Document ID: Same  
    - Columns mapping:  
      - `status` = `"error"`  
      - `row_number` = `{{ $('Loop').item.json.row_number }}`  
      - `last_fetched_time` = `{{ $now.toISO().toString().slice(0, 19).replace('T', ' ') }}`  
    - Matching Columns: `row_number`  
    - Credentials: Google Sheets OAuth2  
    - Connect "false" output of If node to this node.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is a template provided by Agent Circle for n8n users to crawl and extract YouTube channel videos with metadata tracking. It is designed to be manually triggered but can be extended for automation (e.g., Google Sheets triggers). The workflow reads channel inputs from Google Sheets, normalizes URLs or IDs, fetches videos from YouTube API, and updates Google Sheets tabs accordingly. It supports configurable video limits per channel. | Full description and instructions are included in the Sticky Note node content within the workflow JSON. |
| To use this workflow, you must configure Google Cloud Console credentials with enabled APIs for YouTube Data API and Google Sheets API. OAuth2 authentication must be set up properly in n8n for these services. | Google Cloud Console: https://console.cloud.google.com/apis/library                               |
| For the Google Sheets template, use the provided sheet or duplicate: [YouTube - Get Channel Videos Google Sheet](https://docs.google.com/spreadsheets/d/1GIdiUUx1PtEZXOzSUP3aJkrDaVcJdxCGRmCOT94BytA/edit#gid=426418282) with tabs "Channel Urls" and "Videos". | Google Sheets URL as above                                                                       |
| Community and support resources for Agent Circle and n8n automation techniques: Website, Etsy, Gumroad, Discord, Facebook, X, YouTube, LinkedIn. | See Sticky Note content for full list of community links.                                       |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly follows applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---