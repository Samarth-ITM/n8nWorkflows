Extract YouTube Transcripts via Google Sheets or API Webhook

https://n8nworkflows.xyz/workflows/extract-youtube-transcripts-via-google-sheets-or-api-webhook-8211


# Extract YouTube Transcripts via Google Sheets or API Webhook

### 1. Workflow Overview

This workflow enables automated extraction of YouTube video transcripts using two distinct input methods: monitoring a Google Sheet for new YouTube URLs, and receiving direct API requests via a webhook. It leverages the YouTube Transcript API to fetch transcripts along with rich video metadata. The workflow is designed for use cases such as content creation, research, SEO analysis, accessibility improvements, and transcript database generation.

The workflow is organized into two main logical blocks:

- **1.1 Google Sheets Automated Processing:** Watches a Google Sheet named "urls" for newly added YouTube video URLs, extracts the video ID, calls the transcript API, parses the transcript text, and appends the results (video title and transcript) back to another sheet named "transcripts".

- **1.2 Webhook Direct Processing:** Exposes an HTTP POST webhook endpoint to receive a YouTube URL, extract its video ID, fetch and parse the transcript and metadata, then respond immediately with a structured JSON containing the transcript and video details.

Both blocks share similar internal logic for video ID extraction, transcript fetching, and parsing, but differ in input and output mechanisms.

---

### 2. Block-by-Block Analysis

#### 2.1 Google Sheets Automated Processing

- **Overview:**  
This block continuously monitors a specific Google Sheet for newly added YouTube URLs. When a new URL is detected, the workflow extracts the video ID, calls the YouTube Transcript API, parses the returned transcript, and appends the transcript and video title back to another sheet for record-keeping.

- **Nodes Involved:**  
  - Monitor Google Sheet for URLs  
  - Extract YouTube Video ID (Sheets)  
  - Fetch Video Transcript Data (Sheets)  
  - Parse Transcript Text (Sheets)  
  - Save Transcript to Sheet  
  - Sheets Workflow Note (sticky note)  
  - Main Template Explanation (sticky note)

- **Node Details:**

  1. **Monitor Google Sheet for URLs**  
     - *Type:* Google Sheets Trigger  
     - *Role:* Watches the "urls" sheet for new rows added every minute.  
     - *Configuration:*  
       - Event: `rowAdded`  
       - Poll interval: every minute  
       - Document & Sheet IDs must be set by user (Google Sheets OAuth2 credentials required).  
     - *Input:* None (trigger node)  
     - *Output:* JSON containing new row data, including the "url" field.  
     - *Edge cases:*  
       - API quota limits or auth token expiry could cause trigger failure.  
       - Missing or malformed URLs in the sheet rows.  

  2. **Extract YouTube Video ID (Sheets)**  
     - *Type:* Code node (JavaScript)  
     - *Role:* Parses various YouTube URL formats (youtube.com, youtu.be, embed) to extract the video ID.  
     - *Key code:* Regex matching `v=`, `youtu.be/`, or `embed/` followed by the video ID string.  
     - *Input:* JSON with `url` field from previous node.  
     - *Output:* JSON with `url`, extracted `videoId`, and a `success` boolean indicating extraction success.  
     - *Edge cases:*  
       - Invalid or missing URL returns `videoId=null` and `success=false`.  
       - Regex may fail with uncommon URL formats or URL parameters.  

  3. **Fetch Video Transcript Data (Sheets)**  
     - *Type:* HTTP Request  
     - *Role:* Sends POST request to YouTube Transcript API endpoint to fetch transcript and metadata using video ID.  
     - *Configuration:*  
       - URL: `https://www.youtube-transcript.io/api/transcripts`  
       - Method: POST  
       - JSON Body: `{ ids: [ videoId ] }`  
       - Headers: Authorization with YouTube Transcript API token credential.  
     - *Input:* JSON containing `videoId`.  
     - *Output:* API JSON response with transcript data and video metadata.  
     - *Edge cases:*  
       - API auth failure or quota exhaustion.  
       - No transcript available for given video ID (empty or missing `tracks`).  
       - Network timeouts or HTTP errors.  

  4. **Parse Transcript Text (Sheets)**  
     - *Type:* Code node (JavaScript)  
     - *Role:* Extracts and concatenates transcript text segments from the API response.  
     - *Key code:* Maps over `tracks[0].transcript` array segments, joining all `text` fields with spaces.  
     - *Input:* API JSON response.  
     - *Output:* JSON with a single field `fullTranscript` containing the combined transcript text.  
     - *Edge cases:*  
       - Missing or empty `tracks` array leads to undefined behavior (no fallback here).  
       - Response structure changes may break parsing logic.  

  5. **Save Transcript to Sheet**  
     - *Type:* Google Sheets node  
     - *Role:* Appends the retrieved transcript and video title to the "transcripts" sheet.  
     - *Configuration:*  
       - Operation: Append row  
       - Columns: "video title" (fetched from the API response's `microformat.playerMicroformatRenderer.title.simpleText`), and "transcript" (from previous node).  
       - Document and sheet IDs must match the user's sheet.  
     - *Input:* JSON containing `fullTranscript` and video title fields.  
     - *Output:* Status of append operation.  
     - *Edge cases:*  
       - Google Sheets API auth or quota errors.  
       - Data type mismatches or sheet schema changes.  

  6. **Sheets Workflow Note**  
     - *Type:* Sticky Note  
     - *Role:* Describes that this block automates monitoring and processing of YouTube URLs added to a Google Sheet.  

  7. **Main Template Explanation**  
     - *Type:* Sticky Note  
     - *Role:* Provides comprehensive explanation and instructions for the entire workflow.

---

#### 2.2 Webhook Direct Processing

- **Overview:**  
Exposes a webhook endpoint to receive POST requests containing YouTube URLs. Upon receiving a URL, it extracts the video ID, fetches transcript data via the API, parses the transcript and metadata, then returns a JSON response containing the transcript and video details immediately.

- **Nodes Involved:**  
  - Webhook Trigger (Direct Input)  
  - Extract YouTube Video ID (Webhook)  
  - Fetch Video Transcript Data (Webhook)  
  - Parse Transcript Text (Webhook)  
  - Return Transcript Response  
  - Webhook Workflow Note (sticky note)  
  - Main Template Explanation (sticky note)

- **Node Details:**

  1. **Webhook Trigger (Direct Input)**  
     - *Type:* Webhook  
     - *Role:* Listens for POST requests at path `/extract-youtube-transcript`.  
     - *Configuration:*  
       - HTTP Method: POST  
       - Response Mode: Response Node (response returned by downstream node)  
       - No authentication configured by default (custom auth can be added)  
     - *Input:* HTTP POST payload expected to contain JSON with `video_url` field.  
     - *Output:* Passes received data to next node.  
     - *Edge cases:*  
       - Missing or malformed POST body.  
       - Unauthorized access if no authentication implemented.  

  2. **Extract YouTube Video ID (Webhook)**  
     - *Type:* Code node (JavaScript)  
     - *Role:* Extracts YouTube video ID from the `video_url` field in the webhook payload.  
     - *Key code:* Same regex pattern as Sheets block to extract video ID.  
     - *Input:* JSON with `body.video_url` field.  
     - *Output:* JSON containing original URL, extracted `videoId`, and extraction `success` boolean.  
     - *Edge cases:*  
       - Missing or invalid `video_url` field returns `videoId=null`.  

  3. **Fetch Video Transcript Data (Webhook)**  
     - *Type:* HTTP Request  
     - *Role:* Calls YouTube Transcript API with extracted video ID to retrieve transcript and metadata.  
     - *Configuration:* Same as Sheets block: POST to transcript API with Authorization header.  
     - *Input:* JSON with `videoId`.  
     - *Output:* API JSON response data.  
     - *Edge cases:* Same as Sheets block, including API errors and missing transcripts.  

  4. **Parse Transcript Text (Webhook)**  
     - *Type:* Code node (JavaScript)  
     - *Role:* Parses API response to extract full transcript text and rich metadata (video ID, title, channel, publish date, duration, category).  
     - *Key code:*  
       - Joins transcript text from `tracks[0].transcript` segments if available.  
       - If no transcript, falls back to video description text.  
       - Extracts metadata from `microformat.playerMicroformatRenderer`.  
     - *Input:* API JSON response.  
     - *Output:* JSON with fields: `id`, `title`, `channel`, `publishDate`, `duration`, `category`, `fullTranscript`, `hasTranscript`.  
     - *Edge cases:*  
       - Missing transcript triggers fallback to description.  
       - Missing metadata fields handled gracefully via optional chaining.  

  5. **Return Transcript Response**  
     - *Type:* Respond to Webhook  
     - *Role:* Sends JSON response back to the webhook caller with success status, video metadata, and full transcript text.  
     - *Configuration:*  
       - Response body template uses expressions to inject parsed data fields.  
     - *Input:* JSON from parse node.  
     - *Output:* HTTP JSON response.  
     - *Edge cases:*  
       - Response template must handle null or missing fields to prevent malformed JSON.  

  6. **Webhook Workflow Note**  
     - *Type:* Sticky Note  
     - *Role:* Describes the webhook-based direct processing path.  

  7. **Main Template Explanation**  
     - *Type:* Sticky Note  
     - *Role:* Same comprehensive explanation as in Sheets block.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                                  | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                           |
|-------------------------------|------------------------|-------------------------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------|
| Main Template Explanation      | Sticky Note            | Explains overall workflow purpose and usage     | None                         | None                        | Contains full template explanation including setup, usage, and customization instructions            |
| Sheets Workflow Note           | Sticky Note            | Describes Google Sheets automated processing    | None                         | None                        | Notes the Google Sheets monitoring and processing logic                                              |
| Webhook Workflow Note          | Sticky Note            | Describes Webhook direct processing             | None                         | None                        | Notes the webhook input and instant transcript response path                                        |
| Monitor Google Sheet for URLs  | Google Sheets Trigger  | Triggers on new URL rows in Google Sheet        | None                         | Extract YouTube Video ID (Sheets) |                                                                                                      |
| Extract YouTube Video ID (Sheets) | Code                 | Extracts video ID from URL in Sheets path       | Monitor Google Sheet for URLs| Fetch Video Transcript Data (Sheets) |                                                                                                      |
| Fetch Video Transcript Data (Sheets) | HTTP Request         | Calls YouTube Transcript API with video ID      | Extract YouTube Video ID (Sheets) | Parse Transcript Text (Sheets) |                                                                                                      |
| Parse Transcript Text (Sheets) | Code                   | Parses transcript text from API response         | Fetch Video Transcript Data (Sheets) | Save Transcript to Sheet   |                                                                                                      |
| Save Transcript to Sheet       | Google Sheets          | Saves transcript and video title into Sheet     | Parse Transcript Text (Sheets) | None                        |                                                                                                      |
| Webhook Trigger (Direct Input) | Webhook                | Receives POST requests with YouTube URLs        | None                         | Extract YouTube Video ID (Webhook) |                                                                                                      |
| Extract YouTube Video ID (Webhook) | Code                 | Extracts video ID from webhook input URL         | Webhook Trigger (Direct Input) | Fetch Video Transcript Data (Webhook) |                                                                                                      |
| Fetch Video Transcript Data (Webhook) | HTTP Request         | Calls YouTube Transcript API with video ID      | Extract YouTube Video ID (Webhook) | Parse Transcript Text (Webhook) |                                                                                                      |
| Parse Transcript Text (Webhook) | Code                   | Parses transcript and metadata from API response | Fetch Video Transcript Data (Webhook) | Return Transcript Response  |                                                                                                      |
| Return Transcript Response     | Respond to Webhook     | Sends JSON response with transcript and metadata | Parse Transcript Text (Webhook) | None                        |                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation**  
   - Add a sticky note titled "Main Template Explanation" with the full text describing workflow purpose, setup, use cases, and customization tips.  
   - Add sticky notes titled "Sheets Workflow Note" and "Webhook Workflow Note" to describe respective blocks.

2. **Set Up Google Sheets Automated Processing Block**  
   a. Create a **Google Sheets Trigger** node:  
      - Name: "Monitor Google Sheet for URLs"  
      - Trigger on event: `rowAdded`  
      - Poll interval: every minute  
      - Set Google Sheet document ID and sheet name `"urls"`  
      - Use Google Sheets OAuth2 credentials with proper access to your sheet.  

   b. Create a **Code** node:  
      - Name: "Extract YouTube Video ID (Sheets)"  
      - Set code to extract the video ID from the input URL using regex matching `v=`, `youtu.be/`, or `embed/`.  
      - Input: `url` field from trigger node.  
      - Output: JSON with `url`, `videoId`, and `success` flag.  

   c. Create an **HTTP Request** node:  
      - Name: "Fetch Video Transcript Data (Sheets)"  
      - Method: POST  
      - URL: `https://www.youtube-transcript.io/api/transcripts`  
      - Body (JSON): `{ ids: [ "{{ $json.videoId }}" ] }`  
      - Headers: `Authorization` with your YouTube Transcript API token credential.  
      - Use credential named e.g. `youtubeTranscriptApi`.  

   d. Create a **Code** node:  
      - Name: "Parse Transcript Text (Sheets)"  
      - Extract and join transcript text segments from `tracks[0].transcript`.  
      - Output JSON field: `fullTranscript`.  

   e. Create a **Google Sheets** node:  
      - Name: "Save Transcript to Sheet"  
      - Operation: Append row to sheet `"transcripts"`  
      - Columns to append:  
        - "video title": Extract from API response path: `microformat.playerMicroformatRenderer.title.simpleText`  
        - "transcript": From `fullTranscript` field of previous node  
      - Connect to Google Sheets OAuth2 credentials.  
      - Set document ID matching your sheet.  

   f. Connect nodes in order:  
      - `Monitor Google Sheet for URLs` → `Extract YouTube Video ID (Sheets)` → `Fetch Video Transcript Data (Sheets)` → `Parse Transcript Text (Sheets)` → `Save Transcript to Sheet`.

3. **Set Up Webhook Direct Processing Block**  
   a. Create a **Webhook** node:  
      - Name: "Webhook Trigger (Direct Input)"  
      - HTTP Method: POST  
      - Path: `extract-youtube-transcript`  
      - Response Mode: `Response Node` (response sent by downstream node)  

   b. Create a **Code** node:  
      - Name: "Extract YouTube Video ID (Webhook)"  
      - Extract video ID from `body.video_url` in the webhook payload using the same regex as Sheets block.  
      - Output JSON: `{ url, videoId, success }`.  

   c. Create an **HTTP Request** node:  
      - Name: "Fetch Video Transcript Data (Webhook)"  
      - Same configuration as Sheets block HTTP Request node.  

   d. Create a **Code** node:  
      - Name: "Parse Transcript Text (Webhook)"  
      - Extract full transcript text by joining `tracks[0].transcript` segments if available; fall back to video description if no transcript.  
      - Extract metadata: `id`, `title`, `channel`, `publishDate`, `duration`, `category`, `hasTranscript`.  
      - Output JSON with all these fields.  

   e. Create a **Respond to Webhook** node:  
      - Name: "Return Transcript Response"  
      - Response Type: JSON  
      - Response Body Template:  
        ```json
        {
          "success": true,
          "video": {
            "id": "{{ $json.id }}",
            "title": "{{ $json.title }}",
            "channel": "{{ $json.channel }}",
            "duration": "{{ $json.duration }}",
            "hasTranscript": {{ $json.hasTranscript }}
          },
          "transcript": "{{ $json.fullTranscript }}"
        }
        ```  
      - Connect to the previous parse node.  

   f. Connect nodes in order:  
      - `Webhook Trigger (Direct Input)` → `Extract YouTube Video ID (Webhook)` → `Fetch Video Transcript Data (Webhook)` → `Parse Transcript Text (Webhook)` → `Return Transcript Response`.

4. **Credentials Setup**  
   - Create and configure Google Sheets OAuth2 credentials with appropriate scopes to read and append rows.  
   - Create API credential for YouTube Transcript API with token or API key under the name `youtubeTranscriptApi`.  

5. **Testing and Validation**  
   - Test Sheets block by adding new YouTube URLs to the "urls" sheet and verify transcripts are appended to "transcripts" sheet.  
   - Test webhook by sending POST requests to `/extract-youtube-transcript` with JSON body `{ "video_url": "<youtube_url>" }` and check JSON response correctness.  

6. **Customization Options**  
   - Adjust regex in code nodes for broader URL format support.  
   - Enhance error handling, e.g., add checks for missing fields or API errors.  
   - Add additional processing steps like transcript summarization or storage in databases.  
   - Add webhook authentication for security.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow requires access to the YouTube Transcript API, which can be obtained via services like youtube-transcript.io or similar providers.                                                                                                                                                                                                                                    | YouTube Transcript API: https://www.youtube-transcript.io/api                                                   |
| Google Sheets must have two sheets: one named "urls" for input URLs and another named "transcripts" to store results, each with appropriate columns ("url" for input; "video title" and "transcript" for output).                                                                                                                                                                    | Google Sheets Setup                                                                                               |
| The workflow demonstrates how to handle multiple YouTube URL formats, including `youtube.com/watch?v=`, `youtu.be/`, and `embed/` URLs, ensuring robust extraction of video IDs.                                                                                                                                                                                                    | Regex logic in code nodes                                                                                         |
| For webhook security, consider adding authentication or IP whitelisting in production environments.                                                                                                                                                                                                                                                                                  | Security best practices for webhooks                                                                              |
| Users can customize the transcript processing code nodes to implement additional features such as summarization, keyword extraction, or integration with other services like CMS or databases.                                                                                                                                                                                      | Customization and extension suggestions                                                                            |
| The sticky note titled "Main Template Explanation" contains detailed setup instructions, use case explanations, and customization tips for reference directly within the n8n editor.                                                                                                                                                                                               | Embedded documentation                                                                                             |

---

**Disclaimer:** The content provided is exclusively derived from an automated n8n workflow. It complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.