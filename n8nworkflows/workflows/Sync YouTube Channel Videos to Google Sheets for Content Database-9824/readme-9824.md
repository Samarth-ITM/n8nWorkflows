Sync YouTube Channel Videos to Google Sheets for Content Database

https://n8nworkflows.xyz/workflows/sync-youtube-channel-videos-to-google-sheets-for-content-database-9824


# Sync YouTube Channel Videos to Google Sheets for Content Database

### 1. Workflow Overview

This workflow automates the synchronization of videos from a personal YouTube channel into a Google Sheets document, effectively creating and maintaining an organized content database. It is tailored for creators, marketers, or channel managers who want to automatically fetch and store comprehensive video metadata—including titles, tags, captions, publish dates, thumbnails, and privacy statuses—in a spreadsheet for easier content management and analysis.

The workflow's logic is structured into the following main blocks:

- **1.1 Initialization & Channel Data Retrieval:** Starts the workflow manually, retrieves the authenticated user’s YouTube channel details to identify the uploads playlist.
- **1.2 Pagination & Video Listing:** Loops through all pages of the uploads playlist to gather all uploaded video IDs.
- **1.3 Video Metadata Retrieval & Google Sheets Update:** Fetches detailed metadata for each video and appends or updates this data in a Google Sheets document.
- **1.4 Captions Retrieval & Update:** Optionally fetches captions for each video (if available) and updates the spreadsheet accordingly.
- **1.5 Control Logic & Data Management:** Manages pagination variables, aggregates video IDs, and controls conditional branching to ensure full data synchronization.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Channel Data Retrieval

- **Overview:**  
  This block triggers the workflow manually and retrieves the authenticated YouTube channel’s details to obtain the uploads playlist ID, which is essential for listing all uploaded videos.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get My Channel (HTTP Request)  
  - Variables (Set)  
  - Loop Vars (Code)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point allowing manual workflow initiation.  
    - *Configuration:* No parameters; triggers workflow on manual execution.  
    - *Inputs:* None  
    - *Outputs:* To `Get My Channel` node  
    - *Edge Cases:* None (manual trigger).

  - **Get My Channel**  
    - *Type:* HTTP Request  
    - *Role:* Fetch authenticated user's YouTube channel info; specifically requests `contentDetails` part to get playlist IDs.  
    - *Configuration:*  
      - URL: `https://www.googleapis.com/youtube/v3/channels`  
      - Query Parameters: `part=contentDetails`, `mine=true` (authenticated user)  
      - Authentication: Google OAuth2 credential  
    - *Key Expressions:* None dynamic beyond parameters.  
    - *Input:* From manual trigger node.  
    - *Output:* Channel info JSON with uploads playlist ID.  
    - *Edge Cases:*  
      - OAuth token expiry or invalid credentials causing auth errors.  
      - API quota limits or request failures.  
    - *Version:* 4.2

  - **Variables**  
    - *Type:* Set  
    - *Role:* Initialize workflow variables for pagination and playlist ID.  
    - *Configuration:* Sets JSON variables:  
      - `per_page` = 50 (max results per API call)  
      - `current_page` = 0 (start page)  
      - `nextPageToken` = "" (empty, for initial request)  
      - `playlistId` = uploads playlist ID extracted from `Get My Channel` node response  
      - `ids` = "" (empty string to accumulate video IDs)  
    - *Input:* From `Get My Channel`  
    - *Output:* To `Loop Vars`  
    - *Edge Cases:*  
      - Missing or malformed playlistId if the channel data is incomplete.  
    - *Version:* 3.4

  - **Loop Vars**  
    - *Type:* Code (JavaScript)  
    - *Role:* Update pagination variables for the next loop iteration and preserve collected IDs.  
    - *Configuration:*  
      - Reads previous vars or initializes from `Variables` node.  
      - Increments `current_page` by 1.  
      - Updates `nextPageToken` from last `List Uploads` node output if available.  
      - Preserves accumulated video IDs from `Collect ids` node.  
    - *Input:* From `Variables` or itself (looped)  
    - *Output:* To `List Uploads` for next page request  
    - *Edge Cases:*  
      - Error handling if referenced nodes do not have output yet.  
      - Ensures loop termination by handling missing tokens.  
    - *Version:* 2

---

#### 2.2 Pagination & Video Listing

- **Overview:**  
  This block iteratively lists all videos from the channel’s uploads playlist by paging through the YouTube API’s playlistItems endpoint, splitting the results, and updating the Google Sheet with video IDs and published dates.

- **Nodes Involved:**  
  - List Uploads (HTTP Request)  
  - Split Out (Split Out)  
  - Update Posts (Google Sheets)  
  - Collect ids (Code)  
  - If (If condition)  
  - Get row(s) in sheet (Google Sheets)  
  - If2 (If condition)

- **Node Details:**

  - **List Uploads**  
    - *Type:* HTTP Request  
    - *Role:* Fetch a page of videos from the uploads playlist.  
    - *Configuration:*  
      - URL: `https://www.googleapis.com/youtube/v3/playlistItems`  
      - Query Parameters:  
        - `part=contentDetails` (fetch video IDs and publish dates)  
        - `maxResults` from variable `per_page` (50)  
        - `playlistId` from variable  
        - `pageToken` from variable `nextPageToken`  
      - Authentication: Google OAuth2  
    - *Inputs:* From `Loop Vars`  
    - *Outputs:* To `Split Out`  
    - *Edge Cases:*  
      - API quota limits or request failures.  
      - Empty pages or end of pagination (absence of `nextPageToken`).  
    - *Version:* 4.2

  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Splits the array of video items into individual items for processing.  
    - *Configuration:* Splits on the field `items` in API response.  
    - *Input:* From `List Uploads`  
    - *Output:* To `Update Posts`  
    - *Edge Cases:* Empty lists result in no further processing.  
    - *Version:* 1

  - **Update Posts**  
    - *Type:* Google Sheets  
    - *Role:* Append or update video IDs and publish dates into the sheet.  
    - *Configuration:*  
      - Operation: `appendOrUpdate`  
      - Matching Column: `youtube_id`  
      - Columns: `youtube_id` and `published` (from contentDetails.videoId and videoPublishedAt)  
      - Spreadsheet and sheet selected by ID and name ("Videos")  
    - *Input:* From `Split Out`  
    - *Output:* To `Collect ids`  
    - *Edge Cases:*  
      - Sheet access permissions or credential issues.  
      - Data type mismatches handled by disabling type conversion.  
    - *Version:* 4.7

  - **Collect ids**  
    - *Type:* Code (JavaScript)  
    - *Role:* Accumulate unique video IDs from all processed pages for further detailed data retrieval.  
    - *Configuration:*  
      - Reads previous accumulated IDs from `Loop Vars`.  
      - Adds new `youtube_id`s from current batch.  
      - Removes duplicates and stores as a comma-separated string.  
    - *Input:* From `Update Posts`  
    - *Output:* To `If` node  
    - *Edge Cases:* Empty or malformed IDs handled gracefully.  
    - *Version:* 2

  - **If**  
    - *Type:* If  
    - *Role:* Check if current page number has reached or exceeded the total pages, determining loop termination.  
    - *Configuration:*  
      - Condition compares `current_page` >= total pages calculated from API response (`totalResults` / `resultsPerPage`).  
    - *Input:* From `Collect ids`  
    - *Output:*  
      - True branch: Ends looping by continuing to `Get row(s) in sheet`  
      - False branch: Continues looping with `Loop Vars`  
    - *Edge Cases:*  
      - Null or undefined page info could disrupt condition.  
    - *Version:* 2.2

  - **Get row(s) in sheet**  
    - *Type:* Google Sheets  
    - *Role:* Retrieve existing rows from the sheet to check for missing or incomplete data for further processing.  
    - *Configuration:*  
      - Filters on columns `captions`, `privacyStatus`, `uploadStatus` (used later for conditional logic)  
      - Targets same document and sheet as previous Google Sheets node.  
    - *Input:* From `If` (true branch)  
    - *Output:* To `If2`  
    - *Edge Cases:*  
      - Empty sheet returns empty data; handled downstream.  
    - *Version:* 4.7

  - **If2**  
    - *Type:* If  
    - *Role:* Checks whether any rows were returned by `Get row(s) in sheet` to determine if further video details need fetching.  
    - *Configuration:*  
      - Condition tests if JSON returned is not empty.  
    - *Input:* From `Get row(s) in sheet`  
    - *Output:*  
      - True branch: To `Get Video Details` node  
      - False branch: Loop ends or no action.  
    - *Edge Cases:* Empty or invalid data could cause false negatives.  
    - *Version:* 2.2

---

#### 2.3 Video Metadata Retrieval & Google Sheets Update

- **Overview:**  
  For videos identified in the previous block, this block fetches detailed metadata (title, tags, description, category, thumbnails, status) for each video and updates the spreadsheet accordingly.

- **Nodes Involved:**  
  - Get Video Details (HTTP Request)  
  - Split Out1 (Split Out)  
  - Update Posts1 (Google Sheets)  
  - Get captions ID (HTTP Request)  
  - If1 (If condition)

- **Node Details:**

  - **Get Video Details**  
    - *Type:* HTTP Request  
    - *Role:* Retrieve detailed video metadata using the video IDs collected.  
    - *Configuration:*  
      - URL: `https://www.googleapis.com/youtube/v3/videos`  
      - Query Parameters: `part=snippet,status`, `id={{ $json.youtube_id }}` (dynamic per video)  
      - Auth: Google OAuth2  
    - *Input:* From `If2` (videos needing details)  
    - *Output:* To `Split Out1`  
    - *Edge Cases:*  
      - API limits or invalid video IDs causing errors.  
      - Missing fields in response.  
    - *Version:* 4.2

  - **Split Out1**  
    - *Type:* Split Out  
    - *Role:* Splits the detailed videos array into individual video items.  
    - *Configuration:* Splits on `items` field of API response.  
    - *Input:* From `Get Video Details`  
    - *Output:* To `Update Posts1`  
    - *Edge Cases:* None specific.  
    - *Version:* 1

  - **Update Posts1**  
    - *Type:* Google Sheets  
    - *Role:* Append or update detailed video metadata in Google Sheets.  
    - *Configuration:*  
      - Operation: `appendOrUpdate`  
      - Matching column: `youtube_id`  
      - Columns mapped: `tags`, `title`, `maxres` (thumbnail URL), `categoryId`, `description`, `uploadStatus`, `privacyStatus`, `youtube_id`  
      - Spreadsheet and sheet consistent with previous nodes.  
    - *Input:* From `Split Out1`  
    - *Output:* To `Get captions ID`  
    - *Edge Cases:* Sheet access or data mismatch errors possible.  
    - *Version:* 4.7

  - **Get captions ID**  
    - *Type:* HTTP Request  
    - *Role:* Requests captions metadata for a video to identify if captions exist and their ID.  
    - *Configuration:*  
      - URL: `https://www.googleapis.com/youtube/v3/captions`  
      - Query Parameter: `videoId={{ $json.youtube_id }}`  
      - Auth: Google OAuth2  
    - *Input:* From `Update Posts1`  
    - *Output:* To `If1`  
    - *Edge Cases:*  
      - Videos without captions return empty results.  
      - API errors or quota limits.  
    - *Version:* 4.2

  - **If1**  
    - *Type:* If  
    - *Role:* Checks if captions exist by testing if the first caption ID is non-empty.  
    - *Configuration:*  
      - Condition: `items[0].id` is not empty.  
    - *Input:* From `Get captions ID`  
    - *Output:* True branch to `Get captions text` node (fetch captions content)  
    - *Edge Cases:* Empty or missing captions data.  
    - *Version:* 2.2

---

#### 2.4 Captions Retrieval & Update

- **Overview:**  
  If captions exist, this block fetches the captions text in SRT format and appends this information to the Google Sheets content database.

- **Nodes Involved:**  
  - Get captions text (HTTP Request)  
  - Update Posts2 (Google Sheets)

- **Node Details:**

  - **Get captions text**  
    - *Type:* HTTP Request  
    - *Role:* Fetch the captions content in `.srt` subtitle format for a given captions ID.  
    - *Configuration:*  
      - URL: `https://www.googleapis.com/youtube/v3/captions/{{ $json.items[0].id }}`  
      - Query Params: `tfmt=srt`, `alt=media` (fetch raw caption text)  
      - Headers: `Accept: text/plain; charset=UTF-8`  
      - Response type: text  
      - Auth: Google OAuth2  
    - *Input:* From `If1` (when captions exist)  
    - *Output:* To `Update Posts2`  
    - *Edge Cases:*  
      - Captions may not be accessible due to privacy or removal.  
      - API errors or quota issues.  
    - *Version:* 4.2

  - **Update Posts2**  
    - *Type:* Google Sheets  
    - *Role:* Append or update captions text for the video in Google Sheets.  
    - *Configuration:*  
      - Operation: `appendOrUpdate`  
      - Matching column: `youtube_id`  
      - Columns: `captions` (captions text), `youtube_id`  
      - Sheet and document consistent with prior Google Sheets nodes.  
    - *Input:* From `Get captions text`  
    - *Output:* None (end node for this branch)  
    - *Edge Cases:* Large captions data may cause performance issues; ensure text fits within cell limits.  
    - *Version:* 4.7

---

#### 2.5 Control Logic & Data Management

- **Overview:**  
  This block ensures smooth looping over all pages of playlist videos, manages the accumulation of video IDs, and controls conditional branching to decide when to stop looping and when to fetch detailed video info.

- **Nodes Involved:**  
  - Loop Vars (Code)  
  - Collect ids (Code)  
  - If (If condition)  
  - If2 (If condition)

- **Note:** These nodes are already covered in previous blocks but are essential for controlling the workflow execution flow and data integrity.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                         | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                       |
|-------------------------|---------------------|--------------------------------------------------------|--------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Entry point to start the workflow manually             | None                           | Get My Channel                | Setup Instructions: Connect Google account in HTTP Request and Google Sheets nodes. Replace sheet URL with yours. |
| Get My Channel          | HTTP Request         | Retrieve authenticated user's YouTube channel details | When clicking ‘Execute workflow’ | Variables                     |                                                                                                                  |
| Variables               | Set                  | Initialize variables for pagination and playlist ID    | Get My Channel                 | Loop Vars                     |                                                                                                                  |
| Loop Vars               | Code                 | Update pagination vars for looping                      | Variables, List Uploads, Collect ids | List Uploads                 |                                                                                                                  |
| List Uploads            | HTTP Request         | Fetch playlist videos paginated                         | Loop Vars                     | Split Out                    | Flow logic: loops through all pages of uploads playlist, extracts video details, and appends to Google Sheets.  |
| Split Out               | Split Out            | Split playlist items into individual videos            | List Uploads                  | Update Posts                 |                                                                                                                  |
| Update Posts            | Google Sheets        | Append/update video IDs and publish dates              | Split Out                    | Collect ids                  |                                                                                                                  |
| Collect ids             | Code                 | Accumulate unique video IDs                             | Update Posts                 | If                          |                                                                                                                  |
| If                      | If                   | Checks if all pages processed to control loop          | Collect ids                  | Get row(s) in sheet, Loop Vars |                                                                                                                  |
| Get row(s) in sheet     | Google Sheets        | Fetch existing rows to check for missing details       | If                           | If2                         |                                                                                                                  |
| If2                     | If                   | Checks if detailed video data needs fetching            | Get row(s) in sheet           | Get Video Details            |                                                                                                                  |
| Get Video Details       | HTTP Request         | Fetch detailed metadata for videos                      | If2                          | Split Out1                  |                                                                                                                  |
| Split Out1              | Split Out            | Split detailed video data into individual items        | Get Video Details            | Update Posts1               |                                                                                                                  |
| Update Posts1           | Google Sheets        | Append/update detailed video metadata                   | Split Out1                  | Get captions ID             |                                                                                                                  |
| Get captions ID         | HTTP Request         | Retrieve captions metadata for a video                  | Update Posts1                | If1                        | Captions: Captions (.srt) are fetched for each video if available; can be disabled by disconnecting last branch. |
| If1                     | If                   | Check if captions exist                                 | Get captions ID              | Get captions text           |                                                                                                                  |
| Get captions text       | HTTP Request         | Fetch captions content in .srt format                   | If1                         | Update Posts2               |                                                                                                                  |
| Update Posts2           | Google Sheets        | Append/update captions in spreadsheet                   | Get captions text            | None                       |                                                                                                                  |
| Sticky Note             | Sticky Note          | Workflow purpose description                            | None                        | None                       | Sync your YouTube channel videos to Google Sheets. Automatically fetch video data including titles, tags, captions, etc. @[youtube](vOmqW2-T1Xg) |
| Sticky Note1            | Sticky Note          | Setup instructions                                     | None                        | None                       | Connect your Google account in HTTP Request and Google Sheets nodes. Replace the sheet URL with your own Google Sheet. |
| Sticky Note2            | Sticky Note          | Google Sheets template link                            | None                        | None                       | https://docs.google.com/spreadsheets/d/1xTtkkA8ZWGOzu9eisKyY6CcYM6xF9kzS-5HNGSe-XOs/edit?gid=0#gid=0             |
| Sticky Note3            | Sticky Note          | Flow logic summary                                     | None                        | None                       | The flow loops through all pages of your YouTube uploads playlist, extracts video details, and appends them to Google Sheets. |
| Sticky Note4            | Sticky Note          | Captions information                                  | None                        | None                       | Captions (.srt) are fetched for each video if available. This can be disabled by disconnecting the last branch.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: Start the workflow manually.

2. **Add an HTTP Request Node called `Get My Channel`**  
   - Type: HTTP Request  
   - URL: `https://www.googleapis.com/youtube/v3/channels`  
   - Query Parameters:  
     - `part=contentDetails`  
     - `mine=true`  
   - Authentication: Use Google OAuth2 credential with the scope to access YouTube API.  
   - Connect input from Manual Trigger node.

3. **Add a Set Node called `Variables`**  
   - Set JSON parameters:  
     ```json
     {
       "per_page": 50,
       "current_page": 0,
       "nextPageToken": "",
       "playlistId": "{{ $json.items[0].contentDetails.relatedPlaylists.uploads }}",
       "ids": ""
     }
     ```  
   - Connect input from `Get My Channel`.

4. **Create a Code Node called `Loop Vars`**  
   - JavaScript code:  
     ```js
     let vars;
     try {
       vars = $('Loop Vars').last().json;
     } catch (e) {
       vars = $('Variables').last().json;
     }
     const next = Number(vars?.current_page ?? 1) + 1;
     vars.current_page = next;
     try {
       vars.nextPageToken = $('List Uploads').last().json.nextPageToken;
     } catch (e) {
       vars.nextPageToken = '';
     }
     try {
       ids = $('Collect ids').last().json.ids;
     } catch (e) {
       ids = "";
     }
     return [
       {
         json: { ...vars, current_page: next, ids: ids }
       }
     ];
     ```  
   - Connect input from `Variables`.

5. **Add HTTP Request Node called `List Uploads`**  
   - URL: `https://www.googleapis.com/youtube/v3/playlistItems`  
   - Query Parameters:  
     - `part=contentDetails`  
     - `maxResults` = `{{ $json.per_page }}`  
     - `playlistId` = `{{ $json.playlistId }}`  
     - `pageToken` = `{{ $json.nextPageToken }}`  
   - Authentication: Google OAuth2  
   - Connect input from `Loop Vars`.

6. **Add a Split Out Node called `Split Out`**  
   - Field to split: `items`  
   - Connect input from `List Uploads`.

7. **Add Google Sheets Node called `Update Posts`**  
   - Operation: appendOrUpdate  
   - Matching column: `youtube_id`  
   - Columns to map:  
     - `youtube_id` = `{{ $json.contentDetails.videoId }}`  
     - `published` = `{{ $json.contentDetails.videoPublishedAt }}`  
   - Specify your Google Sheet Document ID and sheet name (e.g., "Videos").  
   - Credentials: Google Sheets OAuth2 account  
   - Connect input from `Split Out`.

8. **Add a Code Node called `Collect ids`**  
   - JavaScript code:  
     ```js
     const vars = $('Loop Vars').last().json;
     const prevIds = (vars.ids || '').split(',').map(s => s.trim()).filter(Boolean);
     const newIds = $input.all().map(i => i.json.youtube_id).filter(Boolean);
     const idsString = [...new Set([...prevIds, ...newIds])].join(',');
     vars.ids = idsString;
     return [{ json: { ...vars, ids: idsString } }];
     ```  
   - Connect input from `Update Posts`.

9. **Add an If Node called `If`**  
   - Condition:  
     - Check if `current_page` (from JSON) >= `Math.ceil(totalResults / resultsPerPage)` extracted from last `List Uploads` response.  
   - Connect input from `Collect ids`.  
   - True branch connects to `Get row(s) in sheet` node; False branch loops back to `Loop Vars`.

10. **Add Google Sheets Node called `Get row(s) in sheet`**  
    - Operation: Read rows  
    - Filters: On columns `captions`, `privacyStatus`, `uploadStatus`  
    - Use same Google Sheets document and sheet.  
    - Credentials: Google Sheets OAuth2 account  
    - Connect input from `If` (true branch).

11. **Add If Node called `If2`**  
    - Condition: Check if the JSON output from `Get row(s) in sheet` is not empty.  
    - Connect input from `Get row(s) in sheet`.  
    - True branch connects to `Get Video Details`.

12. **Add HTTP Request Node called `Get Video Details`**  
    - URL: `https://www.googleapis.com/youtube/v3/videos`  
    - Query Parameters:  
      - `part=snippet,status`  
      - `id={{ $json.youtube_id }}`  
    - Authentication: Google OAuth2  
    - Connect input from `If2`.

13. **Add Split Out Node called `Split Out1`**  
    - Split on `items` field.  
    - Connect input from `Get Video Details`.

14. **Add Google Sheets Node called `Update Posts1`**  
    - Operation: appendOrUpdate  
    - Matching column: `youtube_id`  
    - Map columns: `tags`, `title`, `maxres` (thumbnail URL), `categoryId`, `description`, `uploadStatus`, `privacyStatus`, `youtube_id`  
    - Use same Google Sheets document and sheet.  
    - Credentials: Google Sheets OAuth2 account  
    - Connect input from `Split Out1`.

15. **Add HTTP Request Node called `Get captions ID`**  
    - URL: `https://www.googleapis.com/youtube/v3/captions`  
    - Query Parameter: `videoId={{ $json.youtube_id }}`  
    - Authentication: Google OAuth2  
    - Connect input from `Update Posts1`.

16. **Add If Node called `If1`**  
    - Condition: Check if `items[0].id` is not empty.  
    - Connect input from `Get captions ID`.  
    - True branch connects to `Get captions text`.

17. **Add HTTP Request Node called `Get captions text`**  
    - URL: `https://www.googleapis.com/youtube/v3/captions/{{ $json.items[0].id }}`  
    - Query Parameters: `tfmt=srt`, `alt=media`  
    - Header: `Accept: text/plain; charset=UTF-8`  
    - Response type: text  
    - Authentication: Google OAuth2  
    - Connect input from `If1`.

18. **Add Google Sheets Node called `Update Posts2`**  
    - Operation: appendOrUpdate  
    - Matching column: `youtube_id`  
    - Columns: `captions` (raw SRT text), `youtube_id`  
    - Use same Google Sheets document and sheet.  
    - Credentials: Google Sheets OAuth2 account  
    - Connect input from `Get captions text`.

19. **Connect the false branch of `If` node back to `Loop Vars` to continue paging.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Sync your YouTube channel videos to Google Sheets. Automatically fetch video data (title, tags, captions, publish date, etc.).    | Workflow purpose overview.                                                                                        |
| Setup Instructions: Connect your Google account in both HTTP Request and Google Sheets nodes. Replace the sheet URL with your own.| Important for credentials and spreadsheet access.                                                                |
| Google Sheets Template: https://docs.google.com/spreadsheets/d/1xTtkkA8ZWGOzu9eisKyY6CcYM6xF9kzS-5HNGSe-XOs/edit?gid=0#gid=0       | Recommended starting template for the spreadsheet.                                                               |
| Flow logic: The flow loops through all pages of your YouTube uploads playlist, extracts video details, and appends them to Sheets.| Summary of workflow control logic.                                                                                |
| Captions (.srt) are fetched for each video if available. This can be disabled by disconnecting the last branch.                   | Optional captions feature; to disable, disconnect the last branch from `If1` node.                               |
| YouTube API Quota: Be aware that frequent executions may consume your YouTube API quota, leading to temporary request failures.    | Operational consideration for production deployment.                                                            |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.