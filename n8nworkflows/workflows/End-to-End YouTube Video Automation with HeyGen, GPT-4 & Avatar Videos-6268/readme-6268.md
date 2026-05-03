End-to-End YouTube Video Automation with HeyGen, GPT-4 & Avatar Videos

https://n8nworkflows.xyz/workflows/end-to-end-youtube-video-automation-with-heygen--gpt-4---avatar-videos-6268


# End-to-End YouTube Video Automation with HeyGen, GPT-4 & Avatar Videos

### 1. Workflow Overview

This workflow automates the end-to-end creation and publication of YouTube videos using HeyGen for AI-generated avatar videos, GPT-4 for script and metadata generation, and various integration nodes for managing video files and data. It is designed to take input data, generate a video script, produce an AI avatar video, upload it to YouTube, and update metadata accordingly. The workflow targets content creators or marketers seeking to automate YouTube video production with AI-generated content.

The workflow is logically divided into the following blocks:

- **1.1 Input & Data Retrieval:** Triggering the workflow and fetching input data from Google Sheets.
- **1.2 Data Filtering & Preparation:** Filtering, sorting, and preparing data rows for processing.
- **1.3 AI Script Generation:** Using GPT-4 (OpenAI) to generate a structured video script and transcript.
- **1.4 Video Generation & Download:** Sending the script to HeyGen API to generate avatar videos, waiting for completion, and downloading the video files.
- **1.5 YouTube Upload & Metadata Generation:** Uploading the video, generating YouTube metadata via GPT-4, updating YouTube video details, and recording status back to Google Sheets.
- **1.6 Notifications (Optional/Disabled):** Sending emails after metadata updates (currently disabled).

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Data Retrieval

- **Overview:** This block obtains the initial trigger (manual or scheduled) and retrieves rows from a Google Sheet which presumably contain video topics or parameters.
- **Nodes Involved:** Schedule Trigger (disabled), Get row(s) in sheet
- **Node Details:**

  - **Schedule Trigger**
    - Type: Trigger node to start workflow on a schedule.
    - Disabled: true (not active).
    - Potential failure: Misconfigured schedule or disabled state means the workflow must be triggered manually or by other means.
  
  - **Get row(s) in sheet**
    - Type: Google Sheets node to fetch rows.
    - Configuration: Reads rows from a specified sheet.
    - Inputs: Trigger node (Schedule Trigger).
    - Outputs: Passes data to Filter node.
    - Potential failures: Google Sheets API auth errors, empty or malformed sheet data.
  
---

#### 1.2 Data Filtering & Preparation

- **Overview:** This block filters and sorts the fetched data, applies limits, and prepares it for processing.
- **Nodes Involved:** Filter, Sort, Code, Edit Fields, Limit
- **Node Details:**

  - **Filter**
    - Type: Conditional filter node.
    - Configuration: Filters rows based on criteria (not explicitly detailed).
    - Inputs: Data from Google Sheets.
    - Outputs: Data passed to Sort.
    - Failure points: Expression evaluation errors if filter conditions are incorrect.
  
  - **Sort**
    - Type: Sort node.
    - Configuration: Sorts rows based on defined fields.
    - Inputs: Filter output.
    - Outputs: Passes sorted data to Code node.
  
  - **Code**
    - Type: JavaScript code execution.
    - Configuration: Manipulates or transforms data rows, possibly for formatting or calculation.
    - Inputs: Sorted rows.
    - Outputs: Data passed to Edit Fields.
    - Failure points: Code errors or exceptions if data structure unexpected.
  
  - **Edit Fields**
    - Type: Set node.
    - Configuration: Sets or edits fields, possibly to prepare data for next API calls.
    - Inputs: Code output.
    - Outputs: Passes data to Limit.
  
  - **Limit**
    - Type: Limit node.
    - Configuration: Limits the number of items processed, likely to control batch size.
    - Inputs: Edited data.
    - Outputs: Passes data to HTTP Request node for web scraping or API calls.
  
---

#### 1.3 AI Script Generation

- **Overview:** Uses OpenAI GPT-4 to generate a structured video script/transcript from the input data. Parses the output for structured data.
- **Nodes Involved:** OpenAI Chat Model, Structured Output Parser, AI Video Script (Transcript Generator)
- **Node Details:**

  - **OpenAI Chat Model**
    - Type: Language model node using GPT-4.
    - Configuration: Chat-based prompt to generate video script.
    - Inputs: Data prepared from previous blocks.
    - Outputs: Script passed to Structured Output Parser.
    - Failures: API key/auth errors, rate limits, prompt timeouts.
  
  - **Structured Output Parser**
    - Type: Langchain output parser.
    - Configuration: Parses GPT-4 output into structured JSON or other usable format.
    - Inputs: OpenAI Chat Model output.
    - Outputs: Parsed script passed to AI Video Script (Transcript Generator).
    - Failures: Parsing errors if output format unexpected.
  
  - **AI Video Script (Transcript Generator)**
    - Type: Langchain agent node.
    - Configuration: Generates final transcript/script for video generation.
    - Inputs: Parsed data.
    - Outputs: Passes transcript to Generate the Video node.
    - Failures: Agent execution errors, model timeout.
  
---

#### 1.4 Video Generation & Download

- **Overview:** Sends script to HeyGen API to generate avatar video, waits for generation, retrieves video URL, and downloads the video.
- **Nodes Involved:** Generate the Video, Wait for the Video to be Generated, Get Video URL, Download Video
- **Node Details:**

  - **Generate the Video**
    - Type: HTTP Request.
    - Configuration: Sends script and parameters to HeyGen API to start video generation.
    - Inputs: Transcript from AI Video Script node.
    - Outputs: Passes generation status to Wait node.
    - Failures: API errors, network issues.
  
  - **Wait for the Video to be Generated**
    - Type: Wait node with webhook id.
    - Configuration: Pauses workflow until video generation completes.
    - Inputs: Generation request response.
    - Outputs: Triggers Get Video URL.
    - Failures: Webhook timeout or failure if HeyGen does not call back.
  
  - **Get Video URL**
    - Type: HTTP Request.
    - Configuration: Polls or retrieves the final video URL from HeyGen.
    - Inputs: Wait node.
    - Outputs: Passes URL to Download Video.
    - Failures: API errors, invalid response.
  
  - **Download Video**
    - Type: HTTP Request.
    - Configuration: Downloads the video file using the URL.
    - Inputs: Video URL.
    - Outputs: Passes video file/binary data to Upload a video node.
    - Failures: Download errors, network issues.
  
---

#### 1.5 YouTube Upload & Metadata Generation

- **Overview:** Uploads the video to YouTube, generates descriptive metadata using GPT-4, updates video metadata on YouTube, and updates Google Sheets with status.
- **Nodes Involved:** Upload a video, AI Agent for Meta Data of Youtube, Structured Output Parser1, OpenAI Chat Model1, Update Youtube Meta Data, Update row in sheet
- **Node Details:**

  - **Upload a video**
    - Type: YouTube node.
    - Configuration: Uploads video binary to YouTube.
    - Inputs: Video download node.
    - Outputs: Passes video ID or upload confirmation to AI Agent for Meta Data of Youtube.
    - Failures: OAuth errors, upload failures, quota limits.
  
  - **AI Agent for Meta Data of Youtube**
    - Type: Langchain agent.
    - Configuration: Uses GPT-4 to generate video title, description, tags based on video content or input.
    - Inputs: Video upload info.
    - Outputs: Passes generated metadata to Structured Output Parser1.
    - Failures: Model errors, API rate limits.
  
  - **Structured Output Parser1**
    - Type: Langchain output parser.
    - Configuration: Parses metadata into structured format.
    - Inputs: AI Agent output.
    - Outputs: Passes parsed metadata to Update Youtube Meta Data.
    - Failures: Parsing errors.
  
  - **OpenAI Chat Model1**
    - Type: GPT-4 chat node used by AI Agent for Meta Data.
    - Inputs/Outputs linked internally.
  
  - **Update Youtube Meta Data**
    - Type: YouTube node.
    - Configuration: Updates video metadata fields on YouTube using parsed output.
    - Inputs: Parsed metadata and video ID.
    - Outputs: Triggers Update row in sheet.
    - Failures: Auth errors, YouTube API quota, invalid metadata.
  
  - **Update row in sheet**
    - Type: Google Sheets node.
    - Configuration: Updates the row with status or video URL.
    - Inputs: Confirmation from YouTube metadata update.
    - Outputs: Workflow ends or triggers next step.
    - Failures: API errors, row locking.
  
---

#### 1.6 Notifications (Optional/Disabled)

- **Overview:** Sends email notifications after metadata update, currently disabled.
- **Nodes Involved:** SendAndWait email
- **Node Details:**

  - **SendAndWait email**
    - Type: Email send node.
    - Disabled: true
    - Configuration: Would send notification email after metadata update.
    - Inputs: AI Agent for Meta Data of Youtube.
    - Outputs: Update Youtube Meta Data.
    - Failures: SMTP auth errors, disabled state means no execution.
  
---

### 3. Summary Table

| Node Name                         | Node Type                                  | Functional Role                      | Input Node(s)                 | Output Node(s)                           | Sticky Note                      |
|----------------------------------|--------------------------------------------|------------------------------------|------------------------------|-----------------------------------------|---------------------------------|
| Schedule Trigger                 | Schedule Trigger                           | Trigger workflow on schedule       | -                            | Get row(s) in sheet                     |                                 |
| Get row(s) in sheet             | Google Sheets                             | Fetch input data rows              | Schedule Trigger              | Filter                                  |                                 |
| Filter                         | Filter                                    | Filter rows by condition           | Get row(s) in sheet           | Sort                                    |                                 |
| Sort                           | Sort                                      | Sort filtered rows                 | Filter                       | Code                                    |                                 |
| Code                           | Code                                      | Transform or prepare data          | Sort                         | Edit Fields                             |                                 |
| Edit Fields                    | Set                                       | Edit/add fields for next step     | Code                         | Limit                                   |                                 |
| Limit                         | Limit                                     | Limit number of processed items   | Edit Fields                  | HTTP Request                            |                                 |
| HTTP Request                   | HTTP Request                              | Possibly retrieve or scrape data  | Limit                        | Extract HTML                            |                                 |
| Extract HTML                  | Code                                      | Extract relevant HTML content     | HTTP Request                 | AI Video Script (Transcript Generator) |                                 |
| OpenAI Chat Model              | Langchain OpenAI Chat Model                | Generate video script with GPT-4  | -                           | Structured Output Parser                |                                 |
| Structured Output Parser       | Langchain Output Parser                     | Parse GPT-4 output structurally   | OpenAI Chat Model            | AI Video Script (Transcript Generator) |                                 |
| AI Video Script (Transcript Generator) | Langchain Agent                        | Generate transcript/script for video | Structured Output Parser     | Generate the Video                      |                                 |
| Generate the Video             | HTTP Request                              | Send script to HeyGen API         | AI Video Script              | Wait for the Video to be Generated      |                                 |
| Wait for the Video to be Generated | Wait                                      | Wait for video generation callback| Generate the Video           | Get Video URL                          |                                 |
| Get Video URL                 | HTTP Request                              | Retrieve generated video URL      | Wait for the Video to be Generated | Download Video                      |                                 |
| Download Video               | HTTP Request                              | Download video file from URL      | Get Video URL                | Upload a video                         |                                 |
| Upload a video               | YouTube                                   | Upload video to YouTube           | Download Video               | AI Agent for Meta Data of Youtube       |                                 |
| AI Agent for Meta Data of Youtube | Langchain Agent                          | Generate video metadata via GPT-4 | Upload a video               | SendAndWait email / Structured Output Parser1 |                                 |
| OpenAI Chat Model1           | Langchain OpenAI Chat Model                | Used internally by AI Agent for Meta Data | -                           | AI Agent for Meta Data of Youtube       |                                 |
| Structured Output Parser1    | Langchain Output Parser                     | Parse metadata output             | AI Agent for Meta Data of Youtube | Update Youtube Meta Data             |                                 |
| Update Youtube Meta Data     | YouTube                                   | Update video metadata on YouTube | Structured Output Parser1    | Update row in sheet                     |                                 |
| Update row in sheet          | Google Sheets                             | Update sheet with video status    | Update Youtube Meta Data     | -                                       |                                 |
| SendAndWait email            | Email Send                                | Send notification email (disabled) | AI Agent for Meta Data of Youtube | Update Youtube Meta Data             | Disabled node                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** (optional, disabled by default):
   - Configure schedule as needed or leave disabled for manual trigger.

2. **Add a Google Sheets node ("Get row(s) in sheet"):**
   - Connect input from Schedule Trigger.
   - Configure credentials and select spreadsheet and sheet to fetch rows.

3. **Add a Filter node:**
   - Connect input from Google Sheets node.
   - Define filtering criteria to select relevant rows for processing.

4. **Add a Sort node:**
   - Connect Filter output.
   - Configure sorting fields to order rows.

5. **Add a Code node:**
   - Connect Sort output.
   - Write JavaScript to manipulate or transform data as needed (e.g., formatting).

6. **Add a Set node ("Edit Fields"):**
   - Connect Code output.
   - Configure to set or modify fields for next processing steps.

7. **Add a Limit node:**
   - Connect Set output.
   - Configure to limit the number of items processed per run.

8. **Add an HTTP Request node:**
   - Connect Limit output.
   - Configure to fetch or scrape required HTML or data (e.g., from a URL in the row).

9. **Add a Code node ("Extract HTML"):**
   - Connect HTTP Request output.
   - Write JavaScript to extract needed content from HTML response.

10. **Add an OpenAI Chat Model node:**
    - Connect no direct input from previous nodes (the workflow connects it internally).
    - Configure with OpenAI GPT-4 credentials.
    - Set prompt to generate video script/transcript from extracted content.

11. **Add a Structured Output Parser node:**
    - Connect input from OpenAI Chat Model output.
    - Configure to parse the GPT-4 response into structured data.

12. **Add a Langchain Agent node ("AI Video Script (Transcript Generator)"):**
    - Connect Structured Output Parser output.
    - Configure to finalize transcript/script for video generation.

13. **Add an HTTP Request node ("Generate the Video"):**
    - Connect AI Video Script output.
    - Configure HTTP POST to HeyGen API with transcript and parameters to generate avatar video.
    - Setup authentication and headers as required.

14. **Add a Wait node ("Wait for the Video to be Generated"):**
    - Connect Generate the Video output.
    - Configure with webhook ID to pause workflow until HeyGen calls back signaling video completion.

15. **Add an HTTP Request node ("Get Video URL"):**
    - Connect Wait node output.
    - Configure to poll HeyGen API or retrieve the finished video URL.

16. **Add an HTTP Request node ("Download Video"):**
    - Connect Get Video URL output.
    - Configure to download video binary from URL.

17. **Add a YouTube node ("Upload a video"):**
    - Connect Download Video output.
    - Configure YouTube OAuth2 credentials.
    - Set upload parameters for video title, privacy, etc.

18. **Add a Langchain Agent node ("AI Agent for Meta Data of Youtube"):**
    - Connect Upload a video output.
    - Configure to generate video title, description, tags using GPT-4 based on upload info.

19. **Add an OpenAI Chat Model node (used internally by AI Agent):**
    - Configure GPT-4 credentials and prompts as needed.

20. **Add a Structured Output Parser node:**
    - Connect AI Agent for Meta Data output.
    - Configure to parse metadata into structured format.

21. **Add a YouTube node ("Update Youtube Meta Data"):**
    - Connect Structured Output Parser output.
    - Configure to update video metadata fields on YouTube.

22. **Add a Google Sheets node ("Update row in sheet"):**
    - Connect Update Youtube Meta Data output.
    - Configure to update corresponding row with status or URLs.

23. **Optional: Add an Email Send node ("SendAndWait email"):**
    - Disabled by default.
    - Connect AI Agent for Meta Data output.
    - Configure SMTP or email credentials to send notifications.

24. **Add all connections as per above, respecting the data flow and triggering conditions.**

25. **Test each node individually and then run the full workflow to validate end-to-end operation.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                     |
|------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Workflow automates YouTube video creation using HeyGen avatar video generation and GPT-4 AI.   | Project description                                |
| HeyGen API webhook integration requires correct webhook ID setup in Wait node.                 | HeyGen API documentation                          |
| YouTube OAuth2 credentials must have upload and metadata modification scopes enabled.          | https://developers.google.com/youtube/v3/guides/authentication |
| OpenAI GPT-4 usage requires API key with chat model access and suitable rate limits.           | https://platform.openai.com/docs/models/gpt-4     |
| Google Sheets API credentials require edit permissions for updating video statuses.             | https://developers.google.com/sheets/api/guides/authorizing |
| Disabled Schedule Trigger indicates manual or external triggering preferred.                    | Workflow scheduling options                         |
| Disabled Email node can be enabled for notification emails after upload and metadata update.    | SMTP/email configuration                            |

---

**Disclaimer:**  
The provided content is exclusively from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.