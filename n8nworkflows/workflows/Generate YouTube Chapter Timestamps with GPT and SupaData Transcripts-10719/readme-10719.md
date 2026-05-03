Generate YouTube Chapter Timestamps with GPT and SupaData Transcripts

https://n8nworkflows.xyz/workflows/generate-youtube-chapter-timestamps-with-gpt-and-supadata-transcripts-10719


# Generate YouTube Chapter Timestamps with GPT and SupaData Transcripts

### 1. Workflow Overview

This n8n workflow automates the generation and posting of YouTube chapter timestamps as comments or description updates for newly published videos on a YouTube channel. It is designed for content creators who want to add structured chapters to their videos without manual effort.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Video Detection:** Watches a YouTube channel’s RSS feed for new video uploads.
- **1.2 Video Deduplication Check:** Checks if the video ID has already been processed using Airtable to avoid duplicate chapter posts.
- **1.3 Transcript Retrieval:** Fetches the transcript for the new video from the SupaData transcript API.
- **1.4 Transcript Formatting and Duration Calculation:** Converts the raw transcript into a formatted text with timestamps and calculates video length.
- **1.5 AI Chapter Generation:** Sends the formatted transcript to an OpenAI GPT model to generate YouTube-style chapter timestamps.
- **1.6 YouTube Description Update:** Fetches the current video description and appends the generated chapters, then updates the YouTube video description.

The workflow includes optional Airtable integration for deduplication and uses OAuth2 credentials for YouTube API access, SupaData API keys for transcription, and OpenAI credentials for AI processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Video Detection

- **Overview:**  
  This block triggers the workflow whenever a new video is published on the YouTube channel by polling its RSS feed.

- **Nodes Involved:**  
  - Get New YouTube Video (RSS Feed Read Trigger)

- **Node Details:**

  - **Get New YouTube Video**  
    - Type: RSS Feed Read Trigger  
    - Role: Watches the YouTube channel's RSS feed to detect new videos.  
    - Configuration:  
      - RSS feed URL: `https://www.youtube.com/feeds/videos.xml?channel_id=UCDILbVG2rHR_BwfUAqrtuug&nocache=1`  
      - Polling: every minute  
    - Inputs: Trigger (no input)  
    - Outputs: Video feed items, each representing a new video  
    - Edge Cases:  
      - RSS feed unavailable or malformed feed  
      - Network timeouts or delays  
      - Channel ID incorrect or changed  
    - Notes: Runs on a schedule; can be adjusted to less frequent intervals.

---

#### 2.2 Video Deduplication Check

- **Overview:**  
  Checks if the newly detected video has already been processed by searching its ID in an Airtable base to prevent duplicate posts.

- **Nodes Involved:**  
  - Check Airtable  
  - Is New Video? (IF)  
  - Save to Airtable

- **Node Details:**

  - **Check Airtable**  
    - Type: Airtable  
    - Role: Searches Airtable base for the video ID.  
    - Configuration:  
      - Base ID: `appUJQfAXniGZzwL8` (YouTube Video IDS)  
      - Table ID: `tblgRW5IYgTz1bUxB`  
      - Operation: Search by formula: `=FIND("{{ $json.id.split(":").pop().trim() }}", {Video ID}) > 0`  
    - Inputs: Output from RSS feed node  
    - Outputs: Search results (empty if new)  
    - Edge Cases:  
      - API rate limits or auth failures  
      - Incorrect formula leading to false negatives or positives  
      - Airtable base/table changes or deletions

  - **Is New Video?**  
    - Type: IF  
    - Role: Checks if the Airtable search result is empty (indicating a new video).  
    - Configuration: Condition tests if `Video ID` field is empty  
    - Inputs: Output of Check Airtable  
    - Outputs: Branches to next nodes based on condition  
    - Edge Cases: Expression evaluation errors

  - **Save to Airtable**  
    - Type: Airtable  
    - Role: Adds new video ID to Airtable to mark it as processed.  
    - Configuration:  
      - Same base and table as Check Airtable  
      - Operation: Create record with Video ID extracted from RSS node  
    - Inputs: IF node’s “true” branch (new video)  
    - Outputs: Confirmation of record creation  
    - Edge Cases: Airtable API errors, duplicate entries

---

#### 2.3 Transcript Retrieval

- **Overview:**  
  Fetches the transcript of the new video from the SupaData API, which returns timestamps and chapters.

- **Nodes Involved:**  
  - Format Video ID  
  - Fetch Transcript

- **Node Details:**

  - **Format Video ID**  
    - Type: Set  
    - Role: Prepares a simplified object with video title and videoId for the SupaData API request.  
    - Configuration:  
      - Assigns `snippet.title` from RSS feed title  
      - Extracts `id.videoId` from Airtable’s Video ID field  
    - Inputs: Output from Save to Airtable  
    - Outputs: JSON with videoId and title  
    - Edge Cases: Missing or malformed video ID

  - **Fetch Transcript**  
    - Type: HTTP Request  
    - Role: Calls SupaData transcript API for the video URL  
    - Configuration:  
      - URL template: `https://api.supadata.ai/v1/transcript?url=https://youtu.be/{{ $json.id.videoId }}`  
      - Authentication: HTTP header with SupaData API key  
    - Inputs: Output from Format Video ID  
    - Outputs: Transcript response JSON  
    - Edge Cases:  
      - API key invalid or missing  
      - Network errors or timeouts  
      - Transcript unavailable or video not supported  
    - Requires valid HTTP credentials for SupaData API.

---

#### 2.4 Transcript Formatting and Duration Calculation

- **Overview:**  
  Converts the raw transcript into a formatted string with timestamps and calculates the video duration in seconds.

- **Nodes Involved:**  
  - Prepare Transcript + Duration

- **Node Details:**

  - **Prepare Transcript + Duration**  
    - Type: Code (JavaScript)  
    - Role:  
      - Parses transcript JSON to create a formatted multiline string with timestamps (mm:ss) and text lines.  
      - Computes video length from the last transcript offset and duration.  
    - Key Code Logic:  
      - Converts offset milliseconds to mm:ss timestamps  
      - Joins lines with timestamp and text  
      - Returns `formattedTranscript` and `maxDurationSeconds` JSON fields  
    - Inputs: SupaData API transcript data  
    - Outputs: JSON with formatted transcript and video length  
    - Edge Cases:  
      - Transcript JSON missing or malformed  
      - Transcript array empty  
      - Unexpected data types causing runtime errors

---

#### 2.5 AI Chapter Generation

- **Overview:**  
  Sends the formatted transcript and video length to an OpenAI GPT model to generate chapter timestamps with titles.

- **Nodes Involved:**  
  - AI: Generate Chapters  
  - OpenAI Chat Model (connected as AI language model for Agent)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat  
    - Role: Provides backend GPT model (gpt-4o) to the agent node  
    - Configuration: Uses cached GPT-4o model with n8n OpenAI API credentials  
    - Inputs: From n8n agent node (AI: Generate Chapters) via ai_languageModel connection  
    - Outputs: GPT completion results  
    - Edge Cases:  
      - API quota or key errors  
      - Model unavailability or latency

  - **AI: Generate Chapters**  
    - Type: LangChain Agent  
    - Role: Constructs prompt and calls OpenAI model to generate chapters  
    - Configuration:  
      - Prompt includes formatted transcript and video length  
      - Requests output only in the format of timestamped chapters (e.g., "00:00 Intro")  
      - Rules enforce starting at 00:00, no timestamps beyond video length, no extra commentary  
    - Inputs: Output from Prepare Transcript + Duration  
    - Outputs: Text with generated chapters  
    - Edge Cases:  
      - GPT returns unexpected format or no timestamps  
      - Prompt execution errors

---

#### 2.6 YouTube Description Update

- **Overview:**  
  Fetches the current description of the YouTube video and appends the AI-generated chapters, then updates the video description on YouTube.

- **Nodes Involved:**  
  - Fetch Current Description  
  - Update Description

- **Node Details:**

  - **Fetch Current Description**  
    - Type: YouTube  
    - Role: Retrieves current video metadata including description  
    - Configuration:  
      - Operation: get video by ID  
      - Video ID: from Airtable saved record  
      - Credentials: YouTube OAuth2 (AIChris)  
    - Inputs: Output from AI: Generate Chapters  
    - Outputs: Video metadata JSON  
    - Edge Cases:  
      - OAuth token expiration or permission errors  
      - Video unavailable or private  
      - API quota limits

  - **Update Description**  
    - Type: YouTube  
    - Role: Updates the video description by appending the generated chapters  
    - Configuration:  
      - Operation: update video metadata  
      - Fields:  
        - Title (unchanged)  
        - Description: original description + double newlines + generated chapters text  
      - Credentials: same YouTube OAuth2 as above  
    - Inputs: Output of Fetch Current Description  
    - Outputs: Confirmation of update  
    - Edge Cases:  
      - API errors or rate limits  
      - Invalid description length or formatting issues

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                     | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                     |
|----------------------------|----------------------------------|-----------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                | Sticky Note                      | Documentation snippet             | -                           | -                          | ### Requirements - YouTube Data API (OAuth2) - SupaData API Key - Airtable Token (if using base for deduplication) - RSS Feed from your YouTube Channel |
| Sticky Note1               | Sticky Note                      | Documentation snippet             | -                           | -                          | ## YouTube Chapter Auto-Comment (RSS to YouTube) ... [Airtable base link](https://airtable.com/appUJQfAXniGZzwL8/shraiQrFHLF3xxhX7)                           |
| Sticky Note2               | Sticky Note                      | Documentation snippet             | -                           | -                          | This node triggers when your YouTube channel publishes a new video via its RSS feed. You can get your RSS feed from: https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID Runs on a daily schedule (e.g., every 4am). |
| Sticky Note3               | Sticky Note                      | Documentation snippet             | -                           | -                          | Before continuing, this workflow checks if the video ID already exists in Airtable. This prevents double-posting chapters on the same video. - If found → workflow stops - If not found → workflow continues and adds the video ID to Airtable |
| Sticky Note4               | Sticky Note                      | Documentation snippet             | -                           | -                          | This node sends the video to SupaData's transcript API. URL format: https://api.supadata.ai/v1/transcript?url=https://youtu.be/{{ videoId }} SupaData returns a structured response with timestamps and chapters. You must use a valid API key (set in HTTP credentials). |
| Sticky Note5               | Sticky Note                      | Documentation snippet             | -                           | -                          | ### Add Chapters to YouTube Description ... Note: No comment is posted. This step only updates the video description with chapter markers.                                   |
| Sticky Note7               | Sticky Note                      | Documentation snippet             | -                           | -                          | This step uses OpenAI to automatically generate YouTube-style chapter timestamps based on the full video transcript. What it does: • Reads the transcript from the SupaData API • Sends it to GPT-4 or GPT-3.5 via OpenAI • Returns timestamped chapters like: |
| Get New YouTube Video      | RSS Feed Read Trigger            | Detect new YouTube videos         | -                           | Check Airtable              |                                                                                                                                |
| Check Airtable             | Airtable                        | Check if video already processed  | Get New YouTube Video       | Is New Video?               |                                                                                                                                |
| Is New Video?              | IF                              | Branch if video is new or known   | Check Airtable              | Save to Airtable            |                                                                                                                                |
| Save to Airtable           | Airtable                        | Mark video as processed           | Is New Video?               | Format Video ID             |                                                                                                                                |
| Format Video ID            | Set                             | Prepare videoId and title         | Save to Airtable            | Fetch Transcript            |                                                                                                                                |
| Fetch Transcript           | HTTP Request                    | Get transcript from SupaData API  | Format Video ID             | Prepare Transcript + Duration |                                                                                                                                |
| Prepare Transcript + Duration | Code (JavaScript)             | Format transcript and calc length | Fetch Transcript            | AI: Generate Chapters       |                                                                                                                                |
| OpenAI Chat Model          | LangChain OpenAI Chat           | GPT model backend                 | AI: Generate Chapters (ai_languageModel) | AI: Generate Chapters |                                                                                                                                |
| AI: Generate Chapters      | LangChain Agent                 | Generate chapter timestamps       | Prepare Transcript + Duration | Fetch Current Description  |                                                                                                                                |
| Fetch Current Description  | YouTube                        | Get current video description     | AI: Generate Chapters       | Update Description          |                                                                                                                                |
| Update Description         | YouTube                        | Append chapters to description    | Fetch Current Description   | -                          |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add an RSS Feed Read Trigger node:**  
   - Name: `Get New YouTube Video`  
   - Type: RSS Feed Read Trigger  
   - Parameters:  
     - Feed URL: `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID&nocache=1` (replace YOUR_CHANNEL_ID)  
     - Polling interval: every minute or preferred schedule  
   - No credentials needed.

3. **Add an Airtable node for deduplication check:**  
   - Name: `Check Airtable`  
   - Type: Airtable  
   - Operation: Search  
   - Base: Select or enter your Airtable base ID (e.g., `appUJQfAXniGZzwL8`)  
   - Table: Select your table (e.g., `tblgRW5IYgTz1bUxB`)  
   - Filter by formula:  
     ```
     =FIND("{{ $json.id.split(":").pop().trim() }}", {Video ID}) > 0
     ```  
   - Credentials: Set up Airtable API token credentials  
   - Connect output of `Get New YouTube Video` to this node.

4. **Add an IF node:**  
   - Name: `Is New Video?`  
   - Condition: Check if Airtable search result is empty (i.e., `Video ID` field is empty)  
   - Connect output of `Check Airtable` to this node.

5. **Add an Airtable node to save new video:**  
   - Name: `Save to Airtable`  
   - Operation: Create  
   - Base and Table: same as above  
   - Map field `Video ID` to expression:  
     ```
     {{$json["id"].split(":")[2]}}
     ```  
   - Credentials: Airtable token  
   - Connect IF node’s "true" branch (new video) to this node.

6. **Add a Set node to format video ID and title:**  
   - Name: `Format Video ID`  
   - Assign:  
     - `snippet.title` = `{{$node["Get New YouTube Video"].json["title"]}}`  
     - `id.videoId` = `{{$json.fields["Video ID"]}}`  
   - Connect output of `Save to Airtable` to this node.

7. **Add an HTTP Request node:**  
   - Name: `Fetch Transcript`  
   - Method: GET  
   - URL:  
     ```
     https://api.supadata.ai/v1/transcript?url=https://youtu.be/{{$json.id.videoId}}
     ```  
   - Authentication: HTTP Header Auth with SupaData API key  
   - Connect output of `Format Video ID` to this node.

8. **Add a Code node:**  
   - Name: `Prepare Transcript + Duration`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const transcript = $json.content;

     const lines = transcript.map(line => {
       const seconds = Math.floor(line.offset / 1000);
       const minutes = Math.floor(seconds / 60);
       const remaining = seconds % 60;
       const timestamp = `${String(minutes).padStart(2, '0')}:${String(remaining).padStart(2, '0')}`;
       return `${timestamp} ${line.text}`;
     });

     const formattedTranscript = lines.join('\n');

     const last = transcript[transcript.length - 1];
     const maxDurationSeconds = Math.floor((last.offset + last.duration) / 1000);

     return [{
       json: {
         formattedTranscript,
         maxDurationSeconds
       }
     }];
     ```
   - Connect output of `Fetch Transcript` to this node.

9. **Add a LangChain Agent node:**  
   - Name: `AI: Generate Chapters`  
   - Prompt:  
     ```
     Based on the transcript below, generate chapter timestamps with short titles.

     Formatted transcript: {{ $json.formattedTranscript }}

     Full video length: {{ $json.maxDurationSeconds }}

     Only return the chapters in this format:

     00:00 Intro  
     01:10 Key Concept  
     02:45 Main Topic  
     04:30 Explanation  
     ...

     Do not include any intro text, explanation, or extra commentary — only the list of chapters.

     #Rules
     1. The timestamps have to always start at 00:00 to work on YouTube.
     2. Don't include timestamps beyond {{ $json.maxDurationSeconds }} seconds.
     3. Use clear, descriptive titles.
     4. Group together lines close in time.

     00:00 Intro  
     01:10 Key Concept  
     02:45 Main Topic  
     04:30 Explanation

     If we're missing the first 00:00 it won't be valid
     ```
   - Connect output of `Prepare Transcript + Duration` to this node.

10. **Add a LangChain OpenAI Chat node:**  
    - Name: `OpenAI Chat Model`  
    - Model: GPT-4o (or preferred GPT model)  
    - Credentials: OpenAI API key  
    - Connect as `ai_languageModel` input to the `AI: Generate Chapters` node.

11. **Add a YouTube node to fetch current video description:**  
    - Name: `Fetch Current Description`  
    - Operation: Get video by ID  
    - Video ID:  
      ```
      {{$node["Save to Airtable"].json.fields["Video ID"].trim()}}
      ```  
    - Credentials: YouTube OAuth2 with proper scopes  
    - Connect output of `AI: Generate Chapters` to this node.

12. **Add a YouTube node to update video description:**  
    - Name: `Update Description`  
    - Operation: Update video metadata  
    - Video ID: same as above  
    - Title:  
      ```
      {{$node["Get New YouTube Video"].json.title}}
      ```  
    - Description:  
      ```
      {{$json.snippet.description}}

      {{$node["AI: Generate Chapters"].json.output}}
      ```  
    - Credentials: same YouTube OAuth2  
    - Connect output of `Fetch Current Description` to this node.

13. **Connect IF node’s “false” branch (video already processed) to no further actions (workflow stops).**

14. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| YouTube API does not allow pinning comments programmatically. The workflow posts chapters as description updates instead of pinned comments.                  | Sticky Note1, Sticky Note5                                                                           |
| Airtable base template for tracking processed videos: Duplicate available at https://airtable.com/appUJQfAXniGZzwL8/shraiQrFHLF3xxhX7                      | Sticky Note1                                                                                         |
| Obtain YouTube RSS feed URL using: `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID`                                                      | Sticky Note2                                                                                         |
| SupaData transcript API requires a valid API key, passed via HTTP Header Authentication                                                                      | Sticky Note4                                                                                         |
| OpenAI model used is GPT-4o with configured n8n OpenAI API credentials                                                                                        | Node: OpenAI Chat Model                                                                              |
| The transcript is formatted into mm:ss timestamped lines before being sent to AI for chapter generation                                                      | Node: Prepare Transcript + Duration                                                                 |
| Workflow runs best with OAuth2 credentials for YouTube with sufficient scopes: video read/write, comment posting (if used)                                  | YouTube credentials required for Fetch Current Description and Update Description nodes              |

---

**Disclaimer:**  
The text provided here originates exclusively from an automated workflow created with n8n, an integration and automation platform. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.