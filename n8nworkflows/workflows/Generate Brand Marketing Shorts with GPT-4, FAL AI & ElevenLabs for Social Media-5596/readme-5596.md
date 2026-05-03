Generate Brand Marketing Shorts with GPT-4, FAL AI & ElevenLabs for Social Media

https://n8nworkflows.xyz/workflows/generate-brand-marketing-shorts-with-gpt-4--fal-ai---elevenlabs-for-social-media-5596


# Generate Brand Marketing Shorts with GPT-4, FAL AI & ElevenLabs for Social Media

### 1. Workflow Overview

This n8n workflow is designed to automate the generation of brand marketing short videos optimized for social media platforms such as YouTube, Instagram, and TikTok. It leverages GPT-4 for content creation, FAL AI for image and video generation, and ElevenLabs for audio synthesis. The workflow orchestrates the entire process from story retrieval, AI-driven content generation, media rendering, to publishing across platforms.

The workflow’s logic is divided into the following key blocks:

- **1.1 Scheduled Story Retrieval:** Periodically triggers to fetch brand stories from Google Sheets.
- **1.2 Content Prompt Preparation & AI Processing:** Uses GPT-4 and LangChain agents to generate structured prompts for media creation.
- **1.3 Media Generation and Status Monitoring:** Calls APIs for image, video, and audio generation (FAL AI and ElevenLabs), and manages polling to check generation status.
- **1.4 Media Aggregation and Rendering:** Aggregates generated media elements, renders the final video, and downloads it.
- **1.5 Upload and Distribution:** Uploads the final video to Google Drive and distributes it to social media platforms (YouTube, Instagram, TikTok) and Blotato.
- **1.6 Timing and Wait Controls:** Multiple wait nodes regulate asynchronous API response timings and polling intervals.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Story Retrieval

**Overview:**  
This block triggers the workflow on a schedule and retrieves brand story data from Google Sheets to initiate content generation.

**Nodes Involved:**  
- Schedule Trigger  
- Get Story  
- Set Brands  
- Split Out

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Periodic trigger (likely cron-based or interval) initiating the workflow.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to `Get Story`  
  - Edge Cases: Misconfigured schedule could cause missed runs.

- **Get Story**  
  - Type: Google Sheets  
  - Configuration: Fetches brand story rows from a configured Google Sheets spreadsheet.  
  - Inputs: From `Schedule Trigger`  
  - Outputs: To `Set Brands`  
  - Edge Cases: API auth errors, empty or malformed sheet data.

- **Set Brands**  
  - Type: Set  
  - Configuration: Prepares or formats brand-related data extracted from the sheet for downstream processing.  
  - Inputs: From `Get Story`  
  - Outputs: To `Split Out`  
  - Edge Cases: Missing expected properties in input data.

- **Split Out**  
  - Type: Split Out  
  - Configuration: Splits the input array of brand data into individual items for parallel processing.  
  - Inputs: From `Set Brands`  
  - Outputs: To `Prompt Generator`  
  - Edge Cases: Empty input array results in no downstream execution.

---

#### 1.2 Content Prompt Preparation & AI Processing

**Overview:**  
Generates marketing content prompts using GPT-4 and LangChain agent, parsing structured AI outputs for media generation.

**Nodes Involved:**  
- Prompt Generator  
- Prompts  
- GPT 4.1

**Node Details:**  

- **Prompt Generator**  
  - Type: LangChain Agent  
  - Configuration: Uses GPT-4 via OpenAI API to generate marketing content prompts based on brand input.  
  - Inputs: From `Split Out`  
  - Outputs: To `Generate Images`  
  - Key Variables: Brand-specific data forwarded for prompt generation.  
  - Edge Cases: API rate limits, malformed prompts.

- **Prompts**  
  - Type: LangChain Structured Output Parser  
  - Configuration: Parses GPT-generated text into structured JSON or objects usable for image/video/audio generation.  
  - Inputs: From `GPT 4.1`  
  - Outputs: To `Prompt Generator` (via ai_outputParser) and to `GPT 4.1` (ai_languageModel) forming a loop for chaining.  
  - Edge Cases: Parsing errors if GPT output format changes.

- **GPT 4.1**  
  - Type: LangChain OpenRouter LM Chat node  
  - Configuration: Accesses GPT-4 model to process AI prompts and responses.  
  - Inputs: From `Prompt Generator` (ai_languageModel)  
  - Outputs: To `Prompts` (ai_outputParser)  
  - Edge Cases: Connectivity issues, auth problems.

---

#### 1.3 Media Generation and Status Monitoring

**Overview:**  
Requests and monitors generation of images, videos, and audio using external APIs (FAL AI for visuals, ElevenLabs for audio), controlling timing with wait nodes and conditional checks.

**Nodes Involved:**  
- Generate Images  
- 12 Seconds (wait)  
- Get Status  
- Images Done? (If)  
- Get Images  
- Generate Videos  
- 5 Minutes (wait)  
- Get Video Status  
- Videos Done? (If)  
- Get Videos  
- Generate Audio  
- 10 Seconds (wait)  
- Download Video  
- 60 Seconds (wait)  
- 3 Seconds (wait)  
- 20 Seconds (wait)

**Node Details:**  

- **Generate Images**  
  - Type: HTTP Request  
  - Configuration: Calls FAL AI API to generate images based on prompts.  
  - Inputs: From `Prompt Generator`  
  - Outputs: To `12 Seconds` wait node  
  - Edge Cases: API rate limits, validation errors.

- **12 Seconds**  
  - Type: Wait  
  - Configuration: Pauses workflow for 12 seconds to allow image generation processing.  
  - Inputs: From `Generate Images`  
  - Outputs: To `Get Status`  
  - Edge Cases: None.

- **Get Status**  
  - Type: HTTP Request  
  - Configuration: Polls the status of image generation job from API.  
  - Inputs: From `12 Seconds` and `3 Seconds` waits  
  - Outputs: To `Images Done?`  
  - Edge Cases: Timeout, API errors.

- **Images Done?**  
  - Type: If  
  - Configuration: Checks if images are ready based on status response.  
  - Inputs: From `Get Status`  
  - Outputs: If true → `Get Images`; If false → `3 Seconds` wait (re-poll).  
  - Edge Cases: Incorrect condition logic.

- **Get Images**  
  - Type: HTTP Request  
  - Configuration: Downloads or fetches generated images.  
  - Inputs: From `Images Done?` true branch  
  - Outputs: To `Generate Videos`  
  - Edge Cases: Network or file errors.

- **Generate Videos**  
  - Type: HTTP Request  
  - Configuration: Requests video generation with images and prompts.  
  - Inputs: From `Get Images`  
  - Outputs: To `5 Minutes` wait node  
  - Edge Cases: API errors, invalid video parameters.

- **5 Minutes**  
  - Type: Wait  
  - Configuration: Waits 5 minutes for video generation.  
  - Inputs: From `Generate Videos`  
  - Outputs: To `Get Video Status`  
  - Edge Cases: None.

- **Get Video Status**  
  - Type: HTTP Request  
  - Configuration: Polls video generation status.  
  - Inputs: From `5 Minutes` and `20 Seconds` wait nodes  
  - Outputs: To `Videos Done?`  
  - Edge Cases: Timeout, API failures.

- **Videos Done?**  
  - Type: If  
  - Configuration: Checks if video is ready.  
  - Inputs: From `Get Video Status`  
  - Outputs: If true → `Get Videos`; If false → `20 Seconds` wait (re-poll).  
  - Edge Cases: Incorrect condition evaluation.

- **Get Videos**  
  - Type: HTTP Request  
  - Configuration: Retrieves generated video files.  
  - Inputs: From `Videos Done?` true branch  
  - Outputs: To `Generate Audio`  
  - Edge Cases: File retrieval errors.

- **Generate Audio**  
  - Type: HTTP Request  
  - Configuration: Sends text to ElevenLabs API to synthesize voiceover audio.  
  - Inputs: From `Get Videos`  
  - Outputs: To `Upload to Drive`  
  - Edge Cases: API limits, authentication errors.

- **10 Seconds**  
  - Type: Wait  
  - Configuration: Waits 10 seconds, used after downloading video for timing control.  
  - Inputs: From `Download Video`  
  - Outputs: To `Download Video` (loop for repeated downloads if needed)  
  - Edge Cases: None.

- **Download Video**  
  - Type: HTTP Request  
  - Configuration: Downloads the final rendered video from rendering service.  
  - Inputs: From `60 Seconds` and `10 Seconds` waits  
  - Outputs: To `Google Sheets` and loops to `10 Seconds`  
  - On Error: Continues execution (continueErrorOutput)  
  - Edge Cases: Network errors, partial downloads.

- **60 Seconds**  
  - Type: Wait  
  - Configuration: Waits 60 seconds after rendering before downloading video.  
  - Inputs: From `Render Video`  
  - Outputs: To `Download Video`  
  - Edge Cases: None.

- **3 Seconds**  
  - Type: Wait  
  - Configuration: 3-second pause used to throttle polling loops.  
  - Inputs: From `Images Done?` false branch  
  - Outputs: To `Get Status`  
  - Edge Cases: None.

- **20 Seconds**  
  - Type: Wait  
  - Configuration: 20-second pause used to throttle video status polling.  
  - Inputs: From `Videos Done?` false branch  
  - Outputs: To `Get Video Status`  
  - Edge Cases: None.

---

#### 1.4 Media Aggregation and Rendering

**Overview:**  
Aggregates media elements (images, audio) and triggers rendering of the final video.

**Nodes Involved:**  
- Grab Elements  
- Aggregate  
- Render Video

**Node Details:**  

- **Grab Elements**  
  - Type: Set  
  - Configuration: Collects relevant media files and metadata into a structured format for aggregation.  
  - Inputs: From `Share File`  
  - Outputs: To `Aggregate`  
  - Edge Cases: Missing or incomplete media references.

- **Aggregate**  
  - Type: Aggregate  
  - Configuration: Combines multiple media elements into one collection for rendering input.  
  - Inputs: From `Grab Elements`  
  - Outputs: To `Render Video`  
  - Edge Cases: Data aggregation inconsistencies.

- **Render Video**  
  - Type: HTTP Request  
  - Configuration: Calls rendering API to produce final video from aggregated media.  
  - Inputs: From `Aggregate`  
  - Outputs: To `60 Seconds` wait node  
  - Edge Cases: Rendering failure, API timeouts.

---

#### 1.5 Upload and Distribution

**Overview:**  
Uploads completed media to Google Drive and distributes videos to social platforms and Blotato.

**Nodes Involved:**  
- Upload to Drive  
- Share File  
- Upload to Blotato  
- YouTube  
- Instagram  
- TikTok  
- Google Sheets

**Node Details:**  

- **Upload to Drive**  
  - Type: Google Drive  
  - Configuration: Uploads generated audio and video files to Google Drive storage.  
  - Inputs: From `Generate Audio`  
  - Outputs: To `Share File`  
  - Edge Cases: Permission issues, quota limits.

- **Share File**  
  - Type: Google Drive  
  - Configuration: Shares uploaded files and prepares data for distribution.  
  - Inputs: From `Upload to Drive`  
  - Outputs: To `Grab Elements`  
  - Edge Cases: Sharing permission errors.

- **Upload to Blotato**  
  - Type: HTTP Request  
  - Configuration: Uploads final video to Blotato platform for social media posting.  
  - Inputs: From `Google Sheets`  
  - Outputs: To `YouTube`, `Instagram`, and `TikTok` nodes simultaneously.  
  - Edge Cases: Upload failures, API errors.

- **YouTube**  
  - Type: HTTP Request  
  - Configuration: Posts video to YouTube channel using API.  
  - Inputs: From `Upload to Blotato`  
  - Outputs: None  
  - Edge Cases: Auth errors, quota limits.

- **Instagram**  
  - Type: HTTP Request  
  - Configuration: Uploads video to Instagram account.  
  - Inputs: From `Upload to Blotato`  
  - Outputs: None  
  - Edge Cases: API call limits, format restrictions.

- **TikTok**  
  - Type: HTTP Request  
  - Configuration: Uploads video to TikTok profile.  
  - Inputs: From `Upload to Blotato`  
  - Outputs: None  
  - Edge Cases: API throttling.

- **Google Sheets**  
  - Type: Google Sheets  
  - Configuration: Likely logs or updates status of the workflow or video generation in spreadsheet.  
  - Inputs: From `Download Video`  
  - Outputs: To `Upload to Blotato`  
  - Edge Cases: Sheet access errors.

---

#### 1.6 Timing and Wait Controls

**Overview:**  
Wait nodes inserted throughout the workflow to manage asynchronous operations and API rate limits.

**Nodes Involved:**  
- 3 Seconds  
- 5 Minutes  
- 10 Seconds  
- 12 Seconds  
- 20 Seconds  
- 60 Seconds

**Node Details:**  

- All wait nodes have no input parameters set, they simply pause the workflow for the specified duration before passing data downstream.  
- Used primarily for polling API job statuses or allowing enough time for media generation.  
- Edge Cases: Excessive waiting can delay processing; insufficient waiting may cause premature polling.

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                         | Input Node(s)             | Output Node(s)                  | Sticky Note |
|--------------------|-------------------------------|---------------------------------------|---------------------------|--------------------------------|-------------|
| Schedule Trigger    | Schedule Trigger              | Initiates scheduled workflow runs     | -                         | Get Story                      |             |
| Get Story          | Google Sheets                 | Retrieves brand story data             | Schedule Trigger          | Set Brands                    |             |
| Set Brands         | Set                          | Formats brand data                     | Get Story                 | Split Out                     |             |
| Split Out          | Split Out                    | Splits array for parallel processing  | Set Brands                | Prompt Generator              |             |
| Prompt Generator   | LangChain Agent              | Generates AI marketing prompts        | Split Out                 | Generate Images               |             |
| GPT 4.1            | LangChain LM Chat OpenRouter | Processes prompts with GPT-4           | Prompt Generator (ai_languageModel) | Prompts (ai_outputParser) |             |
| Prompts            | LangChain Structured Parser  | Parses GPT output into structured data | GPT 4.1                   | Prompt Generator (loop)       |             |
| Generate Images    | HTTP Request                 | Calls API to generate images           | Prompt Generator           | 12 Seconds                   |             |
| 12 Seconds         | Wait                         | Pause for image generation             | Generate Images            | Get Status                   |             |
| Get Status         | HTTP Request                 | Polls image generation status          | 12 Seconds, 3 Seconds      | Images Done?                 |             |
| Images Done?       | If                           | Checks if images are ready              | Get Status                 | Get Images, 3 Seconds         |             |
| Get Images         | HTTP Request                 | Downloads generated images             | Images Done? True          | Generate Videos              |             |
| Generate Videos    | HTTP Request                 | Requests video generation              | Get Images                 | 5 Minutes                   |             |
| 5 Minutes          | Wait                         | Pause for video generation             | Generate Videos            | Get Video Status             |             |
| Get Video Status   | HTTP Request                 | Polls video generation status           | 5 Minutes, 20 Seconds      | Videos Done?                 |             |
| Videos Done?       | If                           | Checks if videos are ready              | Get Video Status           | Get Videos, 20 Seconds        |             |
| Get Videos         | HTTP Request                 | Retrieves generated videos             | Videos Done? True          | Generate Audio               |             |
| Generate Audio     | HTTP Request                 | Synthesizes audio via ElevenLabs       | Get Videos                 | Upload to Drive              |             |
| Upload to Drive    | Google Drive                 | Uploads audio/video files to Drive     | Generate Audio             | Share File                  |             |
| Share File         | Google Drive                 | Shares files and prepares for aggregation | Upload to Drive           | Grab Elements               |             |
| Grab Elements      | Set                          | Collects media elements for aggregation | Share File                 | Aggregate                   |             |
| Aggregate          | Aggregate                    | Combines media elements                | Grab Elements              | Render Video                |             |
| Render Video       | HTTP Request                 | Calls rendering API                    | Aggregate                  | 60 Seconds                  |             |
| 60 Seconds         | Wait                         | Pause after rendering before download | Render Video               | Download Video              |             |
| Download Video     | HTTP Request                 | Downloads final video                   | 60 Seconds, 10 Seconds     | Google Sheets, 10 Seconds    |             |
| Google Sheets      | Google Sheets                | Logs or updates workflow status         | Download Video             | Upload to Blotato            |             |
| Upload to Blotato  | HTTP Request                 | Uploads video to Blotato platform       | Google Sheets              | YouTube, Instagram, TikTok   |             |
| YouTube            | HTTP Request                 | Publishes video to YouTube              | Upload to Blotato          | -                          |             |
| Instagram          | HTTP Request                 | Publishes video to Instagram            | Upload to Blotato          | -                          |             |
| TikTok             | HTTP Request                 | Publishes video to TikTok               | Upload to Blotato          | -                          |             |
| 3 Seconds          | Wait                         | Short pause for polling throttling      | Images Done? False         | Get Status                  |             |
| 10 Seconds         | Wait                         | Pause after download before checking again | Download Video           | Download Video              |             |
| 20 Seconds         | Wait                         | Pause for video status polling          | Videos Done? False         | Get Video Status            |             |
| Sticky Note1 to 10 | Sticky Note                  | Visual notes (empty content)            | -                         | -                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** to start the workflow periodically (e.g., every hour or day).

2. **Add a Google Sheets node** ("Get Story") configured with credentials and set to read brand story data from a specific sheet.

3. **Add a Set node** ("Set Brands") to format or select relevant fields from the Google Sheets output for downstream processing.

4. **Add a Split Out node** to split the array of brand stories into individual items for parallel processing.

5. **Add a LangChain Agent node** ("Prompt Generator") configured to use the GPT-4 model (credential required). Set it to generate marketing content prompts based on the input brand data.

6. **Add a LangChain LM Chat node** ("GPT 4.1") connected to the Prompt Generator’s ai_languageModel output. Configure it with GPT-4 credentials.

7. **Add a LangChain Structured Output Parser node** ("Prompts") connected to the GPT 4.1’s ai_outputParser output, to parse GPT-4 outputs into structured JSON. Connect its output back to the Prompt Generator’s ai_outputParser input to enable prompt-response chaining.

8. **Add an HTTP Request node** ("Generate Images") to call the FAL AI image generation API. Configure authentication and pass prompts.

9. **Add a 12 Seconds Wait node** connected after "Generate Images" to allow asynchronous image generation to proceed.

10. **Add an HTTP Request node** ("Get Status") to poll the image generation status endpoint. Connect it after the wait node.

11. **Add an If node** ("Images Done?") that checks the status response for completion.

12. - If true, connect to HTTP Request node ("Get Images") to retrieve the generated images.  
    - If false, connect to a 3 Seconds Wait node, which loops back to "Get Status" for re-polling.

13. **Add an HTTP Request node** ("Generate Videos") to request video creation from the images and prompts.

14. **Add a 5 Minutes Wait node** after "Generate Videos" to allow video processing.

15. **Add an HTTP Request node** ("Get Video Status") to poll video generation status.

16. **Add an If node** ("Videos Done?") to check if video is ready.

17. - If true, connect to HTTP Request node ("Get Videos") to download videos.  
    - If false, connect to a 20 Seconds Wait node that loops back to "Get Video Status" for re-polling.

18. **Add an HTTP Request node** ("Generate Audio") to generate voiceover audio via ElevenLabs API.

19. **Add a Google Drive node** ("Upload to Drive") to upload audio and video files.

20. **Add a Google Drive node** ("Share File") to set sharing permissions and prepare files.

21. **Add a Set node** ("Grab Elements") to gather uploaded file references for video rendering.

22. **Add an Aggregate node** ("Aggregate") to combine all media elements into one payload.

23. **Add an HTTP Request node** ("Render Video") to call the rendering API with aggregated media.

24. **Add a 60 Seconds Wait node** after "Render Video" before downloading.

25. **Add an HTTP Request node** ("Download Video") to fetch the rendered video. Configure error handling to continue on failure.

26. **Add a 10 Seconds Wait node** after "Download Video" to handle repeated download attempts if needed.

27. **Add a Google Sheets node** to log or update workflow progress/status after download.

28. **Add an HTTP Request node** ("Upload to Blotato") to upload the video to the Blotato platform.

29. **Add HTTP Request nodes** for "YouTube", "Instagram", and "TikTok" to distribute videos to these platforms. Connect them in parallel after "Upload to Blotato".

30. **Configure all HTTP Request nodes** with appropriate API URLs, authentication (OAuth2 or API keys), headers, and JSON bodies matching each service’s API.

31. **Configure Google Sheets and Google Drive nodes** with OAuth2 credentials.

32. **Set up LangChain nodes** with OpenAI GPT-4 credentials.

33. **Confirm all wait nodes** have appropriate timing values to balance polling frequency and API rate limits.

34. **Add error handling where needed**, especially for HTTP Request nodes that interact with external APIs.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                  |
|--------------------------------------------------------------------------------------------------|--------------------------------|
| This workflow integrates GPT-4, FAL AI, and ElevenLabs for content generation and synthesis.      | Workflow branding               |
| The workflow uses LangChain nodes for structured AI prompt management and parsing.                | n8n LangChain integration      |
| Wait nodes serve as polling mechanisms to respect asynchronous generation latencies and API limits. | API interaction design pattern |
| Google Sheets is used both as an input data source and as a logging/status mechanism.              | Workflow data persistence      |
| Distribution includes multi-platform publishing: YouTube, Instagram, TikTok, and Blotato.         | Multi-channel social media posting |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available.