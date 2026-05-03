Create YouTube Videos Daily from Google Sheets using MagicHour + Gemini

https://n8nworkflows.xyz/workflows/create-youtube-videos-daily-from-google-sheets-using-magichour---gemini-9843


# Create YouTube Videos Daily from Google Sheets using MagicHour + Gemini

### 1. Workflow Overview

This workflow automates the creation, enrichment, and publishing of YouTube videos daily based on new prompts added in a Google Sheet. It leverages AI models (Google Gemini via LangChain) to generate detailed parameters for video creation using the MagicHour Text-to-Video API and to produce optimized YouTube metadata for better SEO and engagement. It handles the entire lifecycle: detecting new prompts, generating videos, polling for video completion, downloading and optionally mixing audio, uploading to YouTube, and updating the original Google Sheet with results.

**Target Use Cases:**  
- Content creators aiming to generate daily YouTube videos automatically from textual prompts.  
- Automated social media video production workflows requiring AI enrichment and metadata optimization.  
- Workflows integrating Google Sheets as prompt input and YouTube as output platform.

**Logical Blocks:**  
- **1.1 Input Reception and Normalization:** Detect new prompts from Google Sheets or chat, normalize input data.  
- **1.2 Video Parameter Generation (AI Processing):** Use Google Gemini AI to convert prompts to MagicHour-compatible JSON video parameters.  
- **1.3 Video Creation and Monitoring:** Send requests to MagicHour API, initialize retry logic, poll until video generation completes.  
- **1.4 Video Download and Processing:** Download completed video and optionally mix background audio.  
- **1.5 YouTube Metadata Generation (AI Processing):** Generate SEO-optimized titles, descriptions, and tags using Google Gemini AI.  
- **1.6 Video Upload and Sheet Update:** Upload video to YouTube, then update the Google Sheet with YouTube URLs and metadata.  
- **1.7 Auxiliary and Control Nodes:** Include retry counters, waits, limits, conditional checks, and helper nodes for data handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Normalization

**Overview:**  
This block receives new video prompt inputs either from a Google Sheets trigger or via a chat webhook, then normalizes the input fields for downstream processing.

**Nodes Involved:**  
- Sheet Row Added  
- Chat Input  
- Normalize Input  
- If  
- Limit  

**Node Details:**  

- **Sheet Row Added**  
  - Type: Google Sheets Trigger  
  - Role: Triggers workflow when a new row is added to a specific Google Sheet (sheet `gid=0` in document ID provided).  
  - Config: Polls every minute for new rows; OAuth2 credentials for Google Sheets.  
  - Inputs: None (trigger)  
  - Outputs: JSON representing new sheet row data  
  - Edge Cases: Authentication failure, API quota limits, empty or malformed rows.  

- **Chat Input**  
  - Type: LangChain Chat Trigger (webhook)  
  - Role: Receive chat inputs as alternative prompt source.  
  - Config: Webhook ID configured for external integration.  
  - Inputs: Incoming chat JSON  
  - Outputs: JSON with chatInput field  
  - Edge Cases: Missing chatInput field; webhook connectivity issues.  

- **Normalize Input**  
  - Type: Set node  
  - Role: Normalize incoming prompt data from either source into unified fields: `Prompt` (string) and `source` (string: 'sheet' or 'chat').  
  - Config: Uses expressions to check if `$json.Prompt` exists, else fallback to `$json.chatInput`.  
  - Inputs: Output from Sheet Row Added or Chat Input  
  - Outputs: JSON with `Prompt` and `source` fields  
  - Edge Cases: Missing prompt data; expression failures if fields are absent.  

- **If**  
  - Type: If condition  
  - Role: Filters out rows that already have a Download URL to avoid duplicate processing.  
  - Config: Checks if `Download URL` field is empty.  
  - Inputs: Sheet Row Added output  
  - Outputs: Passes only new rows without download URL downstream  
  - Edge Cases: Fields missing or misnamed may cause incorrect filtering.  

- **Limit**  
  - Type: Limit  
  - Role: Limits the number of items processed per workflow run (default is 1) to avoid overload.  
  - Inputs: If node output  
  - Outputs: Limited subset passed to Normalize Input  
  - Edge Cases: Limits may cause backlog if many rows added simultaneously.

---

#### 2.2 Video Parameter Generation (AI Processing)

**Overview:**  
This block uses Google Gemini AI via LangChain to transform raw prompts into a strict JSON schema suitable for MagicHour's Text-to-Video API, applying defaults and inferencing.

**Nodes Involved:**  
- Normalize Input  
- Generate Video Parameters (LangChain agent)  
- Google Gemini Chat Model (LangChain LLM)  
- Video Params Parser  

**Node Details:**  

- **Generate Video Parameters**  
  - Type: LangChain Agent node  
  - Role: Sends prompt text to AI model with instructions to return ONLY a JSON object matching MagicHour API schema.  
  - Config: Detailed system message with rules and output schema; uses Google Gemini model credentials.  
  - Inputs: Normalized prompt JSON  
  - Outputs: AI-generated JSON string in `output` field  
  - Edge Cases: AI output not valid JSON, API errors, improper prompt interpretation, rate limits.  

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini LLM  
  - Role: Provides the language model backend for Generate Video Parameters.  
  - Config: Uses Google API credentials for Gemini model  
  - Inputs: Prompt text from Generate Video Parameters  
  - Outputs: AI text response  
  - Edge Cases: API key invalid, quota exceeded, network issues.  

- **Video Params Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI output into structured JSON to enforce schema compliance.  
  - Config: Example JSON schema provided for validation.  
  - Inputs: AI raw text output  
  - Outputs: Parsed JSON object for downstream HTTP request  
  - Edge Cases: Parsing errors, malformed AI output, missing fields.

---

#### 2.3 Video Creation and Monitoring

**Overview:**  
This block submits the video creation request to MagicHour API, initializes retry counters, and polls the API until the video status is complete.

**Nodes Involved:**  
- Generate Video Parameters (output)  
- Create Video (HTTP Request)  
- Initialize Retry Counter (Set)  
- Check Video Status (HTTP Request)  
- Edit Fields (Set)  
- Video Complete? (If)  
- Wait & Retry (Wait)  

**Node Details:**  

- **Create Video**  
  - Type: HTTP Request  
  - Role: POST the JSON video parameters to MagicHour Text-to-Video API to start video generation.  
  - Config: URL `https://api.magichour.ai/v1/text-to-video`, Bearer token credentials, JSON body from AI output.  
  - Inputs: Parsed MagicHour JSON  
  - Outputs: JSON response with `id` for video project  
  - Edge Cases: Auth failure, API downtime, invalid parameters, network timeout.  

- **Initialize Retry Counter**  
  - Type: Set  
  - Role: Stores `video_id` from Create Video, initializes `retry_count`=0 and `max_retries`=20 for polling logic.  
  - Inputs: Create Video output  
  - Outputs: JSON with control variables  
  - Edge Cases: Missing video_id if previous step failed.  

- **Check Video Status**  
  - Type: HTTP Request  
  - Role: GET request to MagicHour API to check progress/status of video generation.  
  - Config: URL built with video_id, Bearer token credentials, 10s timeout.  
  - Inputs: JSON with video_id and retry counters  
  - Outputs: Video status JSON including `status` and download URLs if ready  
  - Edge Cases: API errors, timeout, invalid video_id, auth issues.  

- **Edit Fields**  
  - Type: Set  
  - Role: Passes along `video_id` for subsequent status checks; maintains retry state.  
  - Inputs: Wait & Retry output  
  - Outputs: Updated JSON with video_id and retry count  
  - Edge Cases: Data loss or missing fields from improper input.  

- **Video Complete?**  
  - Type: If  
  - Role: Checks if MagicHour video status is `"complete"`.  
  - Inputs: Check Video Status output  
  - Outputs: Branches workflow to download or retry wait  
  - Edge Cases: Unexpected status values, missing status field.  

- **Wait & Retry**  
  - Type: Wait node  
  - Role: Pauses workflow for 30 seconds before retrying status check.  
  - Inputs: Video Complete? node (false branch)  
  - Outputs: Loop back to Edit Fields for retry  
  - Edge Cases: Workflow timeout if video takes too long; max retries enforced externally.

---

#### 2.4 Video Download and Processing

**Overview:**  
Once the video is ready, this block downloads the video file and optionally mixes it with background audio before preparing metadata.

**Nodes Involved:**  
- Video Complete?  
- Download Video (HTTP Request)  
- Code in JavaScript  
- MixAudio audio (MediaFX)  
- Merge  

**Node Details:**  

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads the video file from MagicHour using the first URL in the `downloads` array.  
  - Config: Response format set to file binary.  
  - Inputs: Video Complete? node (true branch)  
  - Outputs: Binary video data  
  - Edge Cases: URL invalid, file unavailable, network issues.  

- **Code in JavaScript**  
  - Type: Code node (JavaScript)  
  - Role: Converts binary video data into a base64 data URL for downstream video processing.  
  - Code extracts binary data and MIME type, constructs data URL string in JSON field `videoUrl`.  
  - Inputs: Download Video binary output  
  - Outputs: JSON with `videoUrl` string  
  - Edge Cases: Missing or corrupt binary data, code errors.  

- **MixAudio audio**  
  - Type: MediaFX node (mix audio)  
  - Role: Mixes the downloaded video with a free background audio file to enhance video.  
  - Config: Uses fixed audio source URL (https://freepd.com/music/Adventure.mp3) and mixes with video URL from JavaScript node.  
  - Inputs: JavaScript code node output (videoUrl)  
  - Outputs: Combined video with audio  
  - Edge Cases: Free audio URL down, mixing failure, requires n8n Pro or local hosting.  

- **Merge**  
  - Type: Merge  
  - Role: Combines streams from MixAudio and other data sources, ensuring all info is available for next steps.  
  - Inputs: MixAudio audio and Generate YouTube Data outputs  
  - Outputs: Combined JSON for metadata and upload  
  - Edge Cases: Mismatched data lengths or missing inputs.

---

#### 2.5 YouTube Metadata Generation (AI Processing)

**Overview:**  
This block uses Google Gemini AI to generate SEO-optimized YouTube metadata: title, description, and tags based on original prompt and video description.

**Nodes Involved:**  
- Prepare Metadata (Set)  
- Generate YouTube Data (LangChain Agent)  
- Gemini AI Model (LangChain LLM)  
- YouTube JSON Parser  

**Node Details:**  

- **Prepare Metadata**  
  - Type: Set  
  - Role: Extracts the original prompt and enriched video description for AI input.  
  - Inputs: Download Video output (style prompt) and Normalize Input output (Prompt)  
  - Outputs: JSON with `original_prompt` and `video_description`  
  - Edge Cases: Missing fields from previous steps.  

- **Generate YouTube Data**  
  - Type: LangChain Agent node  
  - Role: Sends metadata generation request to AI with instructions to return exact JSON with title, description, and tags.  
  - Config: System message instructs to only output valid JSON for YouTube SEO.  
  - Inputs: Prepare Metadata output  
  - Outputs: AI-generated JSON metadata  
  - Edge Cases: AI output invalid JSON, API errors, rate limits.  

- **Gemini AI Model**  
  - Type: LangChain Google Gemini LLM  
  - Role: Backend language model used for metadata generation.  
  - Inputs: Prompt from Generate YouTube Data  
  - Outputs: AI text response  
  - Edge Cases: API key issues, network problems.  

- **YouTube JSON Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI metadata output to structured JSON object.  
  - Inputs: Raw AI output  
  - Outputs: Parsed metadata JSON for upload and sheet update  
  - Edge Cases: Parsing failures due to malformed AI response.

---

#### 2.6 Video Upload and Sheet Update

**Overview:**  
Uploads the processed video to YouTube using generated metadata and updates the Google Sheet row with YouTube URLs, tags, descriptions, and upload timestamps.

**Nodes Involved:**  
- Merge  
- Wait  
- Upload a video (YouTube node)  
- Prepare Sheet Update (Set)  
- Update Results (Google Sheets)  

**Node Details:**  

- **Wait**  
  - Type: Wait node  
  - Role: Short delay (20s) before upload to ensure readiness.  
  - Inputs: Merge output  
  - Outputs: Passes combined data after wait  
  - Edge Cases: Workflow timeout if repeated.  

- **Upload a video**  
  - Type: YouTube node  
  - Role: Uploads the video binary to YouTube using metadata fields for title, description, tags, privacy, category, and region.  
  - Config: OAuth2 YouTube credentials, public privacy, categoryId 17 (Sports), regionCode IN.  
  - Inputs: Wait output with video and metadata  
  - Outputs: YouTube API response with uploadId (video ID)  
  - Edge Cases: OAuth token expiration, upload failures, quota exceeded.  

- **Prepare Sheet Update**  
  - Type: Set  
  - Role: Prepares fields to update original Google Sheet row with YouTube URL, status, tags, description, download URL, and timestamp.  
  - Inputs: Upload a video output, Check Video Status, Normalize Input, and Generate YouTube Data outputs  
  - Outputs: JSON with all update fields mapped  
  - Edge Cases: Missing fields from previous nodes causing incomplete updates.  

- **Update Results**  
  - Type: Google Sheets node  
  - Role: Updates the original Google Sheet row matching by the `Prompt` field with new metadata and URLs.  
  - Config: Uses OAuth2 credential, sheet `gid=0`, operation `update` with matching column `Prompt`.  
  - Inputs: Prepare Sheet Update output  
  - Outputs: Confirmation of update  
  - Edge Cases: Sheet permissions, row matching failures, API quota.

---

#### 2.7 Auxiliary and Control Nodes

**Overview:**  
Supporting nodes to manage retries, data merging, and error handling.

**Nodes Involved:**  
- Sticky Note (multiple)  
- Merge  
- Edit Fields  
- Wait & Retry  

**Node Details:**  

- **Sticky Notes**  
  - Provide guidance on Google Sheets setup, API credentials for Google Gemini, MagicHour, and YouTube OAuth2, and notes on audio usage.  
  - Cover multiple nodes for contextual instructions.  

- **Merge**  
  - Combines AI-generated metadata and video/audio data streams before upload.  

- **Edit Fields**  
  - Passes `video_id` and retry counters during polling cycles.  

- **Wait & Retry**  
  - Implements wait and retry for polling video status from MagicHour API.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                              | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                                       |
|-------------------------|---------------------------------|----------------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Sheet Row Added         | Google Sheets Trigger            | Trigger on new prompt row in Google Sheet   | None                          | If                            | Google Sheet setup instructions, OAuth setup details, prompt formats for Shorts, credential usage notes.                         |
| Chat Input             | LangChain Chat Trigger (Webhook) | Receive prompts via chat input webhook       | None                          | Normalize Input               |                                                                                                                                    |
| Normalize Input        | Set                             | Normalize prompt input fields                 | Sheet Row Added, Chat Input    | Generate Video Parameters      |                                                                                                                                    |
| If                     | If                              | Filter out rows with existing Download URL   | Sheet Row Added                | Limit                         |                                                                                                                                    |
| Limit                  | Limit                           | Limit number of processed items               | If                            | Normalize Input               |                                                                                                                                    |
| Generate Video Parameters | LangChain Agent                | Generate MagicHour video parameters JSON      | Normalize Input               | Create Video                  | Credential notes for Google Gemini API key setup.                                                                                |
| Google Gemini Chat Model | LangChain LLM                  | AI model backend for video parameters         | Generate Video Parameters      | Generate Video Parameters      |                                                                                                                                    |
| Video Params Parser    | LangChain Output Parser          | Parse AI output into structured JSON          | Generate Video Parameters      | Create Video                  |                                                                                                                                    |
| Create Video            | HTTP Request                    | Submit video creation request to MagicHour    | Video Params Parser           | Initialize Retry Counter       | MagicHour API key setup notes; bearer token authentication.                                                                       |
| Initialize Retry Counter | Set                            | Initialize retry counters and store video_id  | Create Video                  | Check Video Status            |                                                                                                                                    |
| Check Video Status      | HTTP Request                   | Poll MagicHour API for video generation status| Initialize Retry Counter, Edit Fields | Video Complete?          | MagicHour API key setup notes; bearer token authentication.                                                                       |
| Video Complete?         | If                             | Determine if video generation is complete     | Check Video Status             | Download Video (true), Wait & Retry (false) |                                                                                                                            |
| Wait & Retry            | Wait                           | Wait before retrying video status check       | Video Complete? (false branch) | Edit Fields                   |                                                                                                                                    |
| Download Video          | HTTP Request                   | Download the completed video file              | Video Complete? (true branch)  | Prepare Metadata, Code in JavaScript |                                                                                                                                |
| Code in JavaScript      | Code                           | Convert binary video to base64 data URL        | Download Video                | MixAudio audio                |                                                                                                                                    |
| MixAudio audio          | MediaFX                        | Mix background audio with video                 | Code in JavaScript            | Merge                        | Note about free audio usage requiring n8n Pro or local hosting.                                                                   |
| Prepare Metadata        | Set                            | Extract original prompt and video description | Download Video                | Generate YouTube Data         |                                                                                                                                    |
| Generate YouTube Data   | LangChain Agent                | Generate YouTube SEO metadata JSON             | Prepare Metadata              | Merge                        | Google Gemini API key setup note.                                                                                                |
| Gemini AI Model         | LangChain LLM                 | AI model backend for YouTube metadata          | Generate YouTube Data         | Generate YouTube Data          |                                                                                                                                    |
| YouTube JSON Parser     | LangChain Output Parser        | Parse YouTube metadata AI output                | Generate YouTube Data         | Generate YouTube Data          |                                                                                                                                    |
| Merge                  | Merge                         | Combine video and metadata data streams         | MixAudio audio, Generate YouTube Data | Wait                    |                                                                                                                                    |
| Wait                   | Wait                          | Short delay before uploading video              | Merge                        | Upload a video                |                                                                                                                                    |
| Upload a video          | YouTube                       | Upload video to YouTube with metadata           | Wait                         | Prepare Sheet Update           | YouTube OAuth2 credential setup note.                                                                                            |
| Prepare Sheet Update    | Set                           | Prepare data for updating Google Sheet          | Upload a video, Check Video Status, Normalize Input, Generate YouTube Data | Update Results             |                                                                                                                                    |
| Update Results          | Google Sheets                 | Update original Google Sheet row with results   | Prepare Sheet Update          | None                        |                                                                                                                                    |
| Edit Fields            | Set                           | Maintain video_id and retry_count in polling    | Wait & Retry                  | Check Video Status            |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node ("Sheet Row Added")**  
   - Type: Google Sheets Trigger  
   - Trigger event: Row Added  
   - Sheet name: Use the sheet GID (e.g., `gid=0`)  
   - Document ID: Your Google Sheet ID containing prompts  
   - Poll interval: Every minute  
   - Credentials: Google Sheets OAuth2 (configure with Google Cloud Console OAuth client, enable Sheets and Drive APIs)  

2. **Create LangChain Chat Trigger node ("Chat Input")**  
   - Type: LangChain Chat Trigger (Webhook)  
   - Configure webhook with unique ID or URL for external prompt input  

3. **Create Set node ("Normalize Input")**  
   - Create two fields:  
     - `Prompt`: If `$json.Prompt` exists use it, else fallback to `$json.chatInput`  
     - `source`: String "sheet" if Prompt from sheet, else "chat"  

4. **Create If node ("If")**  
   - Condition: Check if field `Download URL` is empty (filter rows not yet processed)  

5. **Create Limit node ("Limit")**  
   - No special config, default to limit 1 item per run to control processing load  

6. **Connect nodes:**  
   - Sheet Row Added → If → Limit → Normalize Input  
   - Chat Input → Normalize Input  

7. **Create LangChain Agent node ("Generate Video Parameters")**  
   - Input: Normalized prompt `Prompt` field  
   - System message: Instructions to produce strict MagicHour Text-to-Video JSON schema  
   - Model: Google Gemini (credential required)  
   - Enable structured output parser  

8. **Create LangChain Google Gemini Chat Model node ("Google Gemini Chat Model")**  
   - Configure with Google API credentials (API key from Google AI Studio/Vertex AI)  

9. **Create LangChain Output Parser node ("Video Params Parser")**  
   - Provide MagicHour JSON schema example for validation  

10. **Connect Normalize Input → Generate Video Parameters → Video Params Parser → Create Video**  

11. **Create HTTP Request node ("Create Video")**  
    - Method: POST  
    - URL: https://api.magichour.ai/v1/text-to-video  
    - Body: JSON from parsed AI output  
    - Authentication: HTTP Bearer (MagicHour API key)  

12. **Create Set node ("Initialize Retry Counter")**  
    - Save `video_id` from Create Video response  
    - Initialize `retry_count` = 0, `max_retries` = 20  

13. **Create HTTP Request node ("Check Video Status")**  
    - Method: GET  
    - URL: https://api.magichour.ai/v1/video-projects/{{ $json.video_id }}  
    - Authentication: HTTP Bearer (same as Create Video)  
    - Timeout: 10 seconds  

14. **Create If node ("Video Complete?")**  
    - Condition: `$json.status == "complete"`  

15. **Create Wait node ("Wait & Retry")**  
    - Wait 30 seconds before retrying  

16. **Create Set node ("Edit Fields")**  
    - Pass `video_id` forward for retry loop  

17. **Connect looping:**  
    - Initialize Retry Counter → Check Video Status → Video Complete?  
    - If false → Wait & Retry → Edit Fields → Check Video Status  
    - If true → Download Video  

18. **Create HTTP Request node ("Download Video")**  
    - Method: GET  
    - URL: `$json.downloads[0].url`  
    - Response: File binary format  

19. **Create Code node ("Code in JavaScript")**  
    - Convert binary video data to base64 data URL string for processing  

20. **Create MediaFX node ("MixAudio audio")**  
    - Operation: Mix Audio  
    - Mix source audio URL: https://freepd.com/music/Adventure.mp3  
    - Mix video source URL: from Code node output  

21. **Create Merge node ("Merge")**  
    - Combine MixAudio audio output and Generate YouTube Data output  

22. **Create Set node ("Prepare Metadata")**  
    - Extract original prompt and video description for AI metadata generation  

23. **Create LangChain Agent node ("Generate YouTube Data")**  
    - Input: original prompt and video description  
    - System message: Generate YouTube SEO metadata JSON only  
    - Model: Google Gemini (same credentials)  
    - Enable structured output parser  

24. **Create LangChain Output Parser node ("YouTube JSON Parser")**  
    - Provide example JSON schema for YouTube metadata  

25. **Connect Prepare Metadata → Generate YouTube Data → YouTube JSON Parser → Merge**  

26. **Create Wait node ("Wait")**  
    - Wait 20 seconds before upload  

27. **Create YouTube node ("Upload a video")**  
    - Operation: Upload video  
    - Map title, description, tags from AI output  
    - Privacy: public  
    - CategoryId: 17 (Sports)  
    - RegionCode: IN  
    - OAuth2 credentials: YouTube Data API v3 (configured in Google Cloud Console)  

28. **Create Set node ("Prepare Sheet Update")**  
    - Prepare fields: YouTube URL, upload status, title, tags, description, download URL, upload timestamp, and original prompt  

29. **Create Google Sheets node ("Update Results")**  
    - Operation: Update row matching by original prompt  
    - Sheet: same as trigger sheet  
    - OAuth2 credentials: same as trigger node  

30. **Connect Upload a video → Prepare Sheet Update → Update Results**  

31. **Create connections for final workflow:**  
    - Download Video → Code in JavaScript → MixAudio audio → Merge  
    - Gemini AI Model nodes assigned for LangChain agents (video parameters and YouTube metadata)  

32. **Add sticky notes on Google Sheets setup, API credentials for Google Gemini, MagicHour, YouTube OAuth2, and audio usage instructions for clarity.**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Google Sheet should have columns: Youtube URL, Youtube Title, Youtube Tags, Youtube Description, Download URL, matched by original Prompt for updates. For Shorts, mention "shorts" in prompt; AI handles orientation and metadata. | Sticky Note near Sheet Row Added node                                                                                        |
| Configure Google Cloud Console project: enable Google Sheets API, Google Drive API, and YouTube Data API v3. Configure OAuth consent and client credentials with n8n redirect URI.                                                  | Sticky Notes near Google Sheets Trigger and YouTube nodes                                                                    |
| Create API key in Google AI Studio or Vertex AI for Google Gemini. Add key in n8n Credentials under Google PaLM/Gemini.                                                                                                            | Sticky Note near Google Gemini Chat Model nodes                                                                               |
| Sign up on MagicHour, create API key, and add as HTTP Bearer credential in n8n. Attach this credential to MagicHour HTTP Request nodes.                                                                                           | Sticky Note near Create Video and Check Video Status nodes                                                                    |
| Free background audio source: https://freepd.com/music/Adventure.mp3. Mixing requires n8n Pro or self-hosting.                                                                                                                      | Sticky Note near MixAudio audio node                                                                                          |

---

**Disclaimer:**  
The text above is exclusively generated from an automated n8n workflow export. All data handled are legal and public, respecting content policies. No illegal, offensive, or protected content is included.