Track Top YouTube Videos by View Count with Google Sheets

https://n8nworkflows.xyz/workflows/track-top-youtube-videos-by-view-count-with-google-sheets-5668


# Track Top YouTube Videos by View Count with Google Sheets

### 1. Workflow Overview

This workflow automates the retrieval and tracking of the most popular YouTube videos by view count from specified channels and logs this data into Google Sheets. It is designed for content analysts, digital marketers, or channel managers who need to monitor top-performing videos across multiple channels efficiently.

The workflow is logically divided into two largely identical parallel blocks (likely for different channel inputs or testing), each performing the following main functions:

- **1.1 Input Reception:** Reading a list of YouTube channel IDs and the number of top videos to retrieve from a configured Google Sheet.
- **1.2 YouTube API Data Fetching:** Querying the YouTube Data API v3 to get the most-viewed videos for each channel.
- **1.3 Data Processing:** Splitting the fetched video list into individual video entries.
- **1.4 Output Writing:** Appending detailed video information (title, video ID, link, channel name) into a designated Google Sheet.

Additional components include manual triggering and comprehensive setup instructions embedded as sticky notes for user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception Block

- **Overview:**  
  Reads input data from a Google Sheet containing YouTube channel IDs and the number of top videos to fetch per channel. This data drives the subsequent API requests.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Read Channel Info from Sheet` (Google Sheets)  
  - `Read Channel Info from Sheet1` (Google Sheets) *(parallel block)*

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point to start the workflow manually  
     - Configuration: Default manual trigger without parameters  
     - Inputs: None  
     - Outputs: Connects to `Read Channel Info from Sheet`  
     - Potential Failures: None (manual action required)  
     - Notes: Allows user-controlled execution

  2. **Read Channel Info from Sheet**  
     - Type: Google Sheets (Read operation)  
     - Role: Retrieves channel IDs and requested video counts from a user-specified sheet  
     - Configuration:  
       - Spreadsheet ID: User-provided  
       - Sheet Name: User-provided input sheet name  
       - Reads all rows by default (no filter set)  
     - Expressions: None explicitly, but dynamic sheet and document IDs used  
     - Inputs: From manual trigger  
     - Outputs: JSON array of channel info objects, each with `ChannelID` and `video_num_to_get` fields  
     - Failures: Authentication errors, invalid spreadsheet ID, empty or malformed sheet data  
     - Version: 4.6 (Google Sheets node)  
     - Parallel node: `Read Channel Info from Sheet1` performs the same function in the parallel block  

#### 2.2 YouTube API Data Fetching Block

- **Overview:**  
  Uses YouTube Data API v3 to fetch the most-viewed videos for each channel specified in the input. Uses channel ID and video count from input data.

- **Nodes Involved:**  
  - `Fetch Most-Viewed Videos via YouTube API` (HTTP Request)  
  - `Fetch Most-Viewed Videos via YouTube API1` (HTTP Request) *(parallel block)*

- **Node Details:**

  1. **Fetch Most-Viewed Videos via YouTube API**  
     - Type: HTTP Request  
     - Role: Executes the YouTube API call to retrieve video data  
     - Configuration:  
       - HTTP Method: GET (default)  
       - URL: `https://www.googleapis.com/youtube/v3/search`  
       - Query Parameters:  
         - `key`: API key (replace `YOUR_YOUTUBE_API_KEY`)  
         - `part`: `snippet` (fetches metadata)  
         - `channelId`: From input JSON (`$json.ChannelID`)  
         - `maxResults`: Number of videos to fetch (`$json.video_num_to_get`)  
         - `order`: `viewCount` (sort by views)  
         - `type`: `video` (filter to videos only)  
     - Inputs: Receives channel info JSON from Google Sheets node  
     - Outputs: YouTube API response containing video list in `items` array  
     - Failures:  
       - Invalid API Key or quota exceeded (HTTP 403)  
       - Malformed requests or invalid channel ID (HTTP 400)  
       - Network timeouts  
     - Version: 4.2  
     - Parallel node: `Fetch Most-Viewed Videos via YouTube API1` in the second logical block  

#### 2.3 Data Processing Block

- **Overview:**  
  Splits the YouTube API's array of videos into individual items for downstream processing and appending to sheets.

- **Nodes Involved:**  
  - `Split Into Individual Videos` (SplitOut)  
  - `Split Into Individual Videos1` (SplitOut) *(parallel block)*

- **Node Details:**

  1. **Split Into Individual Videos**  
     - Type: SplitOut  
     - Role: Transforms the `items` array of videos into separate workflow items, one per video  
     - Configuration:  
       - Field to split out: `items` (array of video objects)  
       - Include all other fields unchanged (`include: allOtherFields`)  
     - Inputs: Receives full API response  
     - Outputs: Individual video data objects, enabling batch append operations  
     - Failures: Empty `items` array results in no output items (handled gracefully)  
     - Version: 1  
     - Parallel node: `Split Into Individual Videos1` performs the same function  

#### 2.4 Output Writing Block

- **Overview:**  
  Appends the processed video details (title, video ID, video link, channel name) into a specified Google Sheet for record-keeping and analysis.

- **Nodes Involved:**  
  - `Append Video Details to Sheet` (Google Sheets)  
  - `Append Video Details to Sheet1` (Google Sheets) *(parallel block)*

- **Node Details:**

  1. **Append Video Details to Sheet**  
     - Type: Google Sheets (Append operation)  
     - Role: Adds a new row per video entry with relevant metadata  
     - Configuration:  
       - Spreadsheet ID: User-provided (same as input sheet)  
       - Sheet Name: User-provided output sheet name  
       - Columns to insert:  
         - `title`: Video title from `$json.items.snippet.title`  
         - `videoId`: Video ID from `$json.items.id.videoId`  
         - `videoLink`: Constructed YouTube URL `https://www.youtube.com/watch?v={{videoId}}`  
         - `channelName`: Channel title from `$json.items.snippet.channelTitle`  
       - Mapping Mode: Manual definition with explicit columns  
       - Operation: Append (adds rows without overwriting)  
     - Inputs: Individual video JSON objects from SplitOut node  
     - Outputs: None (end node)  
     - Failures:  
       - Authentication errors with Google Sheets credentials  
       - Incorrect spreadsheet or sheet name  
       - API quota limits  
     - Version: 4.6  
     - Parallel node: `Append Video Details to Sheet1`  

#### 2.5 Setup Instructions & Guide Block

- **Overview:**  
  Provides user-facing documentation and setup instructions embedded as sticky notes directly within the workflow.

- **Nodes Involved:**  
  - `Setup Instructions & Guide` (Sticky Note)  
  - `Setup Instructions & Guide1` (Sticky Note) *(detailed)*

- **Node Details:**

  1. **Setup Instructions & Guide**  
     - Type: Sticky Note  
     - Role: Placeholder note for basic usage instructions  
     - Content: Markdown text prompting to add usage instructions  
     - Position: Top-left corner (for visibility)  

  2. **Setup Instructions & Guide1**  
     - Type: Sticky Note  
     - Role: Detailed setup instructions including credential setup, Google Sheets template links, and configuration guidelines  
     - Content Highlights:  
       - Steps to create Google Sheets credentials in n8n  
       - How to obtain and configure YouTube Data API key with link to Google Cloud Console  
       - Google Sheets template for input and output with required columns  
       - Node configuration instructions  
     - Size: Large (500x1460) for readability  
     - Position: Near the middle of the workflow for accessibility  

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                      | Input Node(s)                  | Output Node(s)                       | Sticky Note                                                                                                   |
|-----------------------------------|---------------------|-----------------------------------|-------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger      | Manual start of workflow           | None                          | Read Channel Info from Sheet       |                                                                                                              |
| Read Channel Info from Sheet       | Google Sheets       | Reads input channel list           | When clicking ‘Execute workflow’| Fetch Most-Viewed Videos via YouTube API |                                                                                                              |
| Fetch Most-Viewed Videos via YouTube API | HTTP Request        | Fetches top videos from YouTube   | Read Channel Info from Sheet   | Split Into Individual Videos        |                                                                                                              |
| Split Into Individual Videos       | SplitOut            | Splits video array into items      | Fetch Most-Viewed Videos via YouTube API | Append Video Details to Sheet       |                                                                                                              |
| Append Video Details to Sheet      | Google Sheets       | Appends video details to output sheet | Split Into Individual Videos   | None                              |                                                                                                              |
| Setup Instructions & Guide         | Sticky Note         | Basic usage instructions           | None                          | None                              | "## this is note\n\nplease write how to use this node in markdown here!"                                     |
| Read Channel Info from Sheet1      | Google Sheets       | Reads input channel list (parallel) | None                          | Fetch Most-Viewed Videos via YouTube API1 |                                                                                                              |
| Fetch Most-Viewed Videos via YouTube API1 | HTTP Request        | Fetches top videos from YouTube (parallel) | Read Channel Info from Sheet1  | Split Into Individual Videos1       |                                                                                                              |
| Split Into Individual Videos1      | SplitOut            | Splits video array into items (parallel) | Fetch Most-Viewed Videos via YouTube API1 | Append Video Details to Sheet1      |                                                                                                              |
| Append Video Details to Sheet1     | Google Sheets       | Appends video details (parallel)   | Split Into Individual Videos1  | None                              |                                                                                                              |
| Setup Instructions & Guide1        | Sticky Note         | Detailed setup and configuration guide | None                          | None                              | Detailed instructions including:\n- YouTube API key setup\n- Google Sheets template links\n- Node config guide\nhttps://console.cloud.google.com/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`. No special parameters needed.

2. **Create Google Sheets Node to Read Channel Info**  
   - Add a **Google Sheets** node named `Read Channel Info from Sheet`.  
   - Set operation to **Read**.  
   - Configure credentials with your Google Sheets credential.  
   - Enter your Google Spreadsheet ID.  
   - Enter the sheet/tab name containing input data (e.g., `Input`).  
   - This sheet must have columns:  
     - `ChannelID` (YouTube channel identifier)  
     - `video_num_to_get` (number of top videos to fetch)  
   - Connect output of manual trigger to this node.

3. **Create HTTP Request Node to Fetch Videos**  
   - Add an **HTTP Request** node named `Fetch Most-Viewed Videos via YouTube API`.  
   - Set method to GET.  
   - URL: `https://www.googleapis.com/youtube/v3/search`  
   - Under Query Parameters, add:  
     - `key`: Your YouTube Data API v3 key (replace placeholder).  
     - `part`: `snippet`  
     - `channelId`: Expression `{{$json.ChannelID}}` (from previous node’s output)  
     - `maxResults`: Expression `{{$json.video_num_to_get}}`  
     - `order`: `viewCount`  
     - `type`: `video`  
   - Connect output of Google Sheets reading node to this node.

4. **Create SplitOut Node to Process Videos**  
   - Add a **SplitOut** node named `Split Into Individual Videos`.  
   - Configure it to split the field `items` (the array returned by the YouTube API).  
   - Set "Include all other fields" to true.  
   - Connect output of HTTP Request node to this node.

5. **Create Google Sheets Node to Append Video Details**  
   - Add a **Google Sheets** node named `Append Video Details to Sheet`.  
   - Set operation to **Append**.  
   - Configure with the same Google Sheets credentials as before.  
   - Enter the Spreadsheet ID (same as input sheet or separate output sheet).  
   - Enter output sheet/tab name (e.g., `Output`).  
   - Define columns as follows, using expressions:  
     - `title`: `{{$json.items.snippet.title}}`  
     - `videoId`: `{{$json.items.id.videoId}}`  
     - `videoLink`: `=https://www.youtube.com/watch?v={{$json.items.id.videoId}}` (note the `=` to treat as formula)  
     - `channelName`: `{{$json.items.snippet.channelTitle}}`  
   - Connect output of SplitOut node to this node.

6. **(Optional) Duplicate Steps for Parallel Block**  
   - Repeat steps 2 to 5 with new nodes suffixed with `1` for parallel processing or testing alternative inputs.

7. **Add Sticky Notes for Setup Instructions**  
   - Add **Sticky Note** nodes to document the workflow usage and setup guide, including links to:  
     - Google Cloud Console for API key generation  
     - Google Sheets templates  
   - Paste detailed instructions in markdown format for user reference.

8. **Validate Credentials and Permissions**  
   - Ensure your Google Sheets credentials have edit access to the target sheets.  
   - Confirm your YouTube API key has access to YouTube Data API v3 and is not restricted.

9. **Test Execution**  
   - Execute the workflow manually via the trigger node.  
   - Monitor node execution for errors related to credentials, API quota, or data formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Google Sheets Input Template contains columns `ChannelID` and `video_num_to_get` specifying which channels to query and how many top videos to retrieve.                                                                                                                                                                                                                             | [Google Sheets Input Template](https://docs.google.com/spreadsheets/d/1EwALToSWKp4EyzPCx4q3rhMHMaJ3MMfwXQoZmC2BWZk/edit?usp=sharing) |
| To obtain YouTube Data API v3 key, use Google Cloud Console to enable the API and create an API key, following official Google instructions.                                                                                                                                                                                                                                         | [Google Cloud Console](https://console.cloud.google.com/)                                       |
| The workflow relies on dynamic expressions to pass channel IDs and video counts to the YouTube API HTTP request node, ensuring flexible input handling.                                                                                                                                                                                                                               | n8n Expressions documentation                                                                    |
| The `videoLink` field is constructed as a formula to automatically generate clickable YouTube URLs within Google Sheets.                                                                                                                                                                                                                                                               | Google Sheets formula syntax                                                                     |
| The workflow includes two identical parallel blocks that can be used for different input sheets or testing purposes; they are independent and can be merged or removed as needed.                                                                                                                                                                                                    | Workflow design note                                                                             |
| Potential errors include API quota exhaustion, incorrect Google Sheets credentials or permissions, invalid channel IDs, or empty input sheets. Workflow should be monitored and credentials refreshed as needed.                                                                                                                                                                        | Operational considerations                                                                       |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.