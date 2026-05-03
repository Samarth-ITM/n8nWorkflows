Transform Quotes to Viral Videos with Gemini, GPT & ElevenLabs for Social Media

https://n8nworkflows.xyz/workflows/transform-quotes-to-viral-videos-with-gemini--gpt---elevenlabs-for-social-media-6737


# Transform Quotes to Viral Videos with Gemini, GPT & ElevenLabs for Social Media

### 1. Workflow Overview

This workflow, titled **"Transform Quotes to Viral Videos with Gemini, GPT & ElevenLabs for Social Media"**, is designed to automate the generation of cinematic short videos from textual quotes or input data, leveraging advanced AI models (Google Gemini and OpenAI GPT), text-to-speech synthesis (ElevenLabs), video rendering services, cloud storage, and social media scheduling APIs. Its target use case is content creators, marketers, and social media managers who want to convert text quotes or scripts into engaging viral short videos and automatically schedule their distribution on platforms such as TikTok, YouTube, and Instagram.

The workflow is structured into several logical blocks reflecting the flow from input retrieval, AI processing, media generation, cloud storage, and social media scheduling:

- **1.1 Input Reception and Preparation:** Triggering the workflow and fetching input data from Google Sheets.
- **1.2 AI Processing:** Using Google Gemini and OpenAI GPT models to generate video script content and structured outputs.
- **1.3 Text-to-Speech & Audio Upload:** Converting generated text to speech with ElevenLabs and uploading audio to Google Cloud Storage (GCS).
- **1.4 Video Generation & Rendering:** Creating videos via an external service (Creatomate), monitoring rendering status, and downloading final videos.
- **1.5 Media Upload and Processing:** Uploading generated videos and audio to cloud services (GCS, Cloudinary, Postiz).
- **1.6 Social Media Scheduling:** Scheduling the finalized videos for posting on TikTok, YouTube, and Instagram platforms.
- **1.7 Quota and Workflow Control:** Managing quotas in Google Sheets and controlling conditional logic and flow switching.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preparation

- **Overview:**  
This block initiates the workflow either manually or on a schedule and fetches input rows from a Google Sheet, limiting and setting them up for AI processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger  
  - Get row(s) in sheet (Google Sheets)  
  - Limit  
  - Set Input Data  

- **Node Details:**  
  - *When clicking ‘Execute workflow’*  
    - Type: Manual Trigger  
    - Role: Manual start point for testing or on-demand execution  
    - No parameters  
    - Output connects to "Get row(s) in sheet"
    - Edge cases: None, but user-trigger dependent

  - *Schedule Trigger*  
    - Type: Schedule Trigger  
    - Role: Automated periodic trigger to start the workflow  
    - No parameters shown (likely cron or interval configured in production)  
    - Output connects to "Get row(s) in sheet"
    - Edge cases: Scheduling errors or missed executions

  - *Get row(s) in sheet*  
    - Type: Google Sheets node  
    - Role: Retrieve data rows containing quotes or inputs for video creation  
    - Configured with target spreadsheet and worksheet  
    - Outputs multiple rows; input for “Limit” node  
    - Edge cases: API errors, empty sheets, invalid ranges

  - *Limit*  
    - Type: Limit Node  
    - Role: Restrict number of processed rows per execution (e.g., batch size)  
    - Configured with max item count to process  
    - Output connects to "Set Input Data"  
    - Edge cases: If limit set too low/high, affects throughput

  - *Set Input Data*  
    - Type: Set Node  
    - Role: Prepare data structure or variables for downstream AI processing  
    - Sets or formats input data fields for prompt generation  
    - Output connects to “Prompt Agent”  
    - Edge cases: Expression errors if data missing

---

#### 1.2 AI Processing

- **Overview:**  
Generates video content scripts using Google Gemini and OpenAI GPT language models and processes structured outputs for further media generation.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Prompt Agent  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Content writer  

- **Node Details:**  
  - *Google Gemini Chat Model*  
    - Type: LangChain Google Gemini Chat Language Model  
    - Role: Primary AI model to generate initial prompt responses or dialogue  
    - Likely configured with credentials and parameters for chat  
    - Output connects to "Prompt Agent" in ai_languageModel input  
    - Edge cases: API key errors, rate limits, model failures

  - *Prompt Agent*  
    - Type: LangChain Agent Node  
    - Role: Orchestrates prompt handling and chaining between language models  
    - Parameters likely include prompt templates and chaining logic  
    - Output connects to "Content writer"  
    - Edge cases: Expression errors, chaining failures

  - *OpenAI Chat Model*  
    - Type: LangChain OpenAI Chat Language Model  
    - Role: Secondary language model to refine or generate content based on Gemini output  
    - Uses structured output parser for formatting  
    - Output connects to "Content writer" as ai_languageModel input  
    - Edge cases: API quota exceeded, model unavailability

  - *Structured Output Parser*  
    - Type: LangChain Output Parser Structured  
    - Role: Parses AI model responses into structured format suitable for content writing  
    - Tightly coupled with "Content writer" node  
    - Edge cases: Parsing failures if AI output format unexpected

  - *Content writer*  
    - Type: LangChain Chain LLM Node  
    - Role: Final step generating the text scripts for video and audio narration  
    - Output connects to "Convert text to speech"  
    - Edge cases: AI generation errors, empty outputs

---

#### 1.3 Text-to-Speech & Audio Upload

- **Overview:**  
Converts generated text scripts into speech audio using ElevenLabs and uploads the audio files to Google Cloud Storage.

- **Nodes Involved:**  
  - Convert text to speech (ElevenLabs)  
  - Upload to GCS Audio  
  - SET Credentials  

- **Node Details:**  
  - *Convert text to speech*  
    - Type: ElevenLabs TTS node  
    - Role: Uses ElevenLabs API to synthesize speech from AI-generated text  
    - Configured with voice settings and API credentials  
    - Output audio data passed to "Upload to GCS Audio"  
    - Edge cases: API failures, unsupported languages or text, rate limits

  - *Upload to GCS Audio*  
    - Type: Google Cloud Storage Node  
    - Role: Uploads generated audio files to a specified GCS bucket  
    - Configured with bucket name, path, and credentials  
    - Retry enabled with 2s delay between attempts  
    - Output connects to "SET Credentials" (likely resets or sets tokens)  
    - Edge cases: Upload failures, permission errors

  - *SET Credentials*  
    - Type: Set Node  
    - Role: Handles or refreshes credentials or tokens for subsequent steps  
    - Output connects to "JWT" node  
    - Edge cases: Missing or invalid credentials

---

#### 1.4 Video Generation & Rendering

- **Overview:**  
Handles video generation via external API (Creatomate), monitors rendering status through polling with waits, and finally downloads the rendered video.

- **Nodes Involved:**  
  - JWT  
  - GET Token  
  - Generate Video1  
  - Wait for Video  
  - Fetch Status  
  - Switch  
  - Convert to File  
  - Upload to GCS Video  
  - Creatomate HTTP Body  
  - Merge - Creatomate  
  - Wait for Rendering  
  - Check status of render  
  - If3  
  - Send to Cloudinary  
  - Make upscaled URL  
  - Download file Cloudinary  
  - Upload video to Postiz  
  - Get Postiz integrations  
  - Switch1  

- **Node Details:**  
  - *JWT*  
    - Type: JWT Node  
    - Role: Generates JWT tokens for authentication with video generation API  
    - Output connects to "GET Token"  
    - Edge cases: Token generation failure

  - *GET Token*  
    - Type: HTTP Request  
    - Role: Requests access tokens for video generation API  
    - Uses JWT in headers or body  
    - Output connects to "Generate Video1"  
    - Edge cases: Authentication errors, network failures

  - *Generate Video1*  
    - Type: HTTP Request  
    - Role: Initiates video generation with API using obtained token and parameters  
    - Output connects to "Wait for Video"  
    - Edge cases: API quota exceeded, invalid parameters

  - *Wait for Video*  
    - Type: Wait Node with webhook  
    - Role: Pauses workflow until notified or timeout before checking video status  
    - Output connects to "Fetch Status"  
    - Edge cases: Timeout or webhook failure

  - *Fetch Status*  
    - Type: HTTP Request  
    - Role: Polls API for current video rendering status  
    - Output connects to "Switch" to decide next step  
    - Edge cases: API unresponsiveness, parsing errors

  - *Switch*  
    - Type: Switch Node  
    - Role: Branches flow based on render status (e.g., ready or still processing)  
    - Two outputs: "Convert to File" (if ready), "Wait for Video" (if not ready)  
    - Edge cases: Invalid status values

  - *Convert to File*  
    - Type: Convert To File Node  
    - Role: Converts or processes generated video file, likely adjusting aspect ratio to 9:16  
    - Output connects to "Upload to GCS Video"  
    - Edge cases: File conversion failures

  - *Upload to GCS Video*  
    - Type: Google Cloud Storage Node  
    - Role: Uploads final video file to GCS bucket  
    - Retry enabled with 2s intervals  
    - Output connects to "Creatomate HTTP Body"  
    - Edge cases: Upload failures

  - *Creatomate HTTP Body*  
    - Type: Code Node  
    - Role: Prepares HTTP request body for Creatomate API call for video rendering  
    - Executes once per workflow run  
    - Output connects to "Merge - Creatomate"  
    - Edge cases: Code errors

  - *Merge - Creatomate*  
    - Type: HTTP Request  
    - Role: Sends finalized request to Creatomate API to merge video components and start rendering  
    - Output connects to "Wait for Rendering"  
    - Edge cases: API failures

  - *Wait for Rendering*  
    - Type: Wait Node with webhook  
    - Role: Pauses workflow until rendering completes  
    - Output connects to "Check status of render"  
    - Edge cases: Timeout, webhook errors

  - *Check status of render*  
    - Type: HTTP Request  
    - Role: Polls Creatomate API for rendering status  
    - Output connects to "If3" node for conditional handling  
    - Edge cases: API failures

  - *If3*  
    - Type: If Node  
    - Role: Decides whether rendering finished or needs to wait further  
    - True branch: "Send to Cloudinary" for uploading final video  
    - False branch: loops back to "Wait for Rendering"  
    - Edge cases: Condition misconfiguration

  - *Send to Cloudinary*  
    - Type: HTTP Request  
    - Role: Uploads final video to Cloudinary CDN/storage  
    - Output connects to "Make upscaled URL"  
    - Edge cases: Upload failure

  - *Make upscaled URL*  
    - Type: Code Node  
    - Role: Generates or modifies Cloudinary URL to use upscaled video version  
    - Output connects to "Download file Cloudinary"  
    - Edge cases: URL manipulation errors

  - *Download file Cloudinary*  
    - Type: HTTP Request  
    - Role: Downloads video from Cloudinary for further processing or upload  
    - Output connects to "Upload video to Postiz"  
    - Edge cases: Download failures

  - *Upload video to Postiz*  
    - Type: HTTP Request  
    - Role: Uploads video to Postiz platform for social media integrations  
    - Output connects to "Get Postiz integrations"  
    - Edge cases: API or upload failures

  - *Get Postiz integrations*  
    - Type: HTTP Request  
    - Role: Fetches available social media integrations from Postiz  
    - Output connects to "Switch1" for scheduling decisions  
    - Edge cases: API failures

  - *Switch1*  
    - Type: Switch Node  
    - Role: Routes workflow to schedule posts on TikTok, YouTube, or Instagram based on integration availability or config  
    - Outputs connect to platform-specific scheduling nodes  
    - Edge cases: Misrouting

---

#### 1.5 Social Media Scheduling

- **Overview:**  
Schedules the generated videos to be published on social media platforms including TikTok, YouTube, and Instagram.

- **Nodes Involved:**  
  - Schedule TikTok  
  - Schedule YouTube  
  - Schedule Instagram  
  - Mark Quota as Done  

- **Node Details:**  
  - *Schedule TikTok*  
    - Type: HTTP Request  
    - Role: Calls API to schedule video posting on TikTok via Postiz or direct integration  
    - Edge cases: API errors, scheduling failures

  - *Schedule YouTube*  
    - Type: HTTP Request  
    - Role: Schedules video upload or publication on YouTube  
    - Edge cases: OAuth token expiration, API quota

  - *Schedule Instagram*  
    - Type: HTTP Request  
    - Role: Schedules video posting on Instagram  
    - Output connects to "Mark Quota as Done"  
    - Edge cases: API rate limits

  - *Mark Quota as Done*  
    - Type: Google Sheets Node  
    - Role: Updates Google Sheet to mark quota consumption or task completion for the processed row  
    - Edge cases: Sheet update failures

---

#### 1.6 Quota and Workflow Control

- **Overview:**  
Manages workflow control, branching logic, and quota state updates to ensure smooth processing and avoid overuse.

- **Nodes Involved:**  
  - Switch  
  - Switch1  
  - If3  
  - Limit  
  - Set Input Data  

- **Node Details:**  
  - *Switch* and *Switch1*  
    - Type: Switch nodes for branching logic based on conditions (e.g., video render status, platform selection)  
    - Critical for routing flow properly  
    - Edge cases: Misconfiguration causing dead ends

  - *If3*  
    - Type: If node, controlling rendering completion logic  
    - Edge cases: Condition failures

  - *Limit* and *Set Input Data*  
    - Control batch sizes and input preparation, critical for quota management  
    - Edge cases: Improper limits affecting processing

---

### 3. Summary Table

| Node Name                | Node Type                                  | Functional Role                                | Input Node(s)                | Output Node(s)                 | Sticky Note                                     |
|--------------------------|--------------------------------------------|------------------------------------------------|-----------------------------|-------------------------------|------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                             | Manual start trigger                           |                             | Get row(s) in sheet           |                                                |
| Schedule Trigger         | Schedule Trigger                           | Scheduled start trigger                        |                             | Get row(s) in sheet           |                                                |
| Get row(s) in sheet      | Google Sheets                             | Retrieve input data rows                       | When clicking ‘Execute workflow’, Schedule Trigger | Limit                     |                                                |
| Limit                   | Limit                                      | Limit input rows processed                      | Get row(s) in sheet          | Set Input Data                |                                                |
| Set Input Data           | Set                                        | Format and prepare input for AI                | Limit                       | Prompt Agent                  |                                                |
| Google Gemini Chat Model | LangChain Google Gemini Chat Model         | AI model for initial prompt generation          |                             | Prompt Agent (ai_languageModel input) |                                                |
| Prompt Agent             | LangChain Agent                            | Orchestrate prompt handling with AI models    | Set Input Data, Google Gemini Chat Model | Content writer               |                                                |
| OpenAI Chat Model        | LangChain OpenAI Chat Model                 | AI model for content refinement                 |                             | Content writer (ai_languageModel input) |                                                |
| Structured Output Parser | LangChain Output Parser Structured          | Parse structured AI responses                   |                             | Content writer (ai_outputParser input) |                                                |
| Content writer           | LangChain Chain LLM                        | Generate final video script text                | Prompt Agent, OpenAI Chat Model, Structured Output Parser | Convert text to speech      |                                                |
| Convert text to speech   | ElevenLabs Text-to-Speech                   | Convert text scripts to speech audio            | Content writer              | Upload to GCS Audio           |                                                |
| Upload to GCS Audio      | Google Cloud Storage                        | Upload audio files to GCS bucket                | Convert text to speech       | SET Credentials              |                                                |
| SET Credentials          | Set                                        | Manage or refresh credentials                    | Upload to GCS Audio          | JWT                         |                                                |
| JWT                      | JWT Node                                   | Create JWT token for API authentication         | SET Credentials             | GET Token                   |                                                |
| GET Token                | HTTP Request                              | Obtain API access token                          | JWT                         | Generate Video1              |                                                |
| Generate Video1          | HTTP Request                              | Request video generation from API                | GET Token                   | Wait for Video              |                                                |
| Wait for Video           | Wait Node (Webhook)                        | Wait for video rendering to complete             | Generate Video1             | Fetch Status                |                                                |
| Fetch Status             | HTTP Request                              | Poll video generation API for status             | Wait for Video              | Switch                      |                                                |
| Switch                   | Switch                                    | Branch depending on video status                  | Fetch Status                | Convert to File, Wait for Video |                                                |
| Convert to File          | Convert To File                           | Convert video to 9:16 aspect ratio                 | Switch                      | Upload to GCS Video         |                                                |
| Upload to GCS Video      | Google Cloud Storage                      | Upload video file to GCS                           | Convert to File             | Creatomate HTTP Body        |                                                |
| Creatomate HTTP Body     | Code                                      | Prepare HTTP body for Creatomate video rendering | Upload to GCS Video         | Merge - Creatomate          |                                                |
| Merge - Creatomate       | HTTP Request                              | Initiate video merging/rendering via Creatomate  | Creatomate HTTP Body        | Wait for Rendering          |                                                |
| Wait for Rendering       | Wait Node (Webhook)                        | Wait for Creatomate rendering to complete         | Merge - Creatomate          | Check status of render      |                                                |
| Check status of render   | HTTP Request                              | Poll Creatomate API for render status             | Wait for Rendering          | If3                        |                                                |
| If3                      | If Node                                   | Conditional check for render completion           | Check status of render      | Send to Cloudinary, Wait for Rendering |                                                |
| Send to Cloudinary       | HTTP Request                              | Upload final video to Cloudinary                   | If3 (true)                  | Make upscaled URL           |                                                |
| Make upscaled URL        | Code                                      | Modify Cloudinary URL for upscaled video           | Send to Cloudinary          | Download file Cloudinary    |                                                |
| Download file Cloudinary | HTTP Request                              | Download video from Cloudinary                      | Make upscaled URL           | Upload video to Postiz      |                                                |
| Upload video to Postiz   | HTTP Request                              | Upload video to Postiz platform                      | Download file Cloudinary    | Get Postiz integrations     |                                                |
| Get Postiz integrations  | HTTP Request                              | Retrieve available social media integrations        | Upload video to Postiz      | Switch1                    |                                                |
| Switch1                  | Switch                                    | Route to scheduling nodes based on integration     | Get Postiz integrations     | Schedule TikTok, Schedule YouTube, Schedule Instagram |                                                |
| Schedule TikTok          | HTTP Request                              | Schedule video post on TikTok                         | Switch1                    |                             |                                                |
| Schedule YouTube         | HTTP Request                              | Schedule video post on YouTube                        | Switch1                    |                             |                                                |
| Schedule Instagram       | HTTP Request                              | Schedule video post on Instagram                      | Switch1                    | Mark Quota as Done          |                                                |
| Mark Quota as Done       | Google Sheets                             | Update Google Sheet to mark quota as used             | Schedule Instagram          |                             |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Workflow:** Name it "AI Cinematic Short Generator".

2. **Set up Triggers:**  
   - Add a *Manual Trigger* node for on-demand execution.  
   - Add a *Schedule Trigger* node configured with desired cron or interval for automatic runs.

3. **Fetch Input Data:**  
   - Add a *Google Sheets* node ("Get row(s) in sheet") configured with the target spreadsheet and worksheet containing quotes or inputs.  
   - Add a *Limit* node to restrict the number of rows processed per run (set max items as needed).  
   - Connect *Schedule Trigger* and *Manual Trigger* to "Get row(s) in sheet", then connect to *Limit*.

4. **Prepare Input Data:**  
   - Add a *Set* node ("Set Input Data") to format or set variables based on rows from *Limit*.  
   - Connect *Limit* to *Set Input Data*.

5. **Configure AI Models:**  
   - Add a *LangChain - Google Gemini Chat Model* node ("Google Gemini Chat Model") with Google Gemini API credentials.  
   - Add a *LangChain - Agent* node ("Prompt Agent") that takes input from "Set Input Data" and "Google Gemini Chat Model".  
   - Add a *LangChain - OpenAI Chat Model* node with OpenAI API credentials.  
   - Add a *LangChain - Structured Output Parser* node to parse AI outputs.  
   - Add a *LangChain - Chain LLM* node ("Content writer") that uses outputs from "Prompt Agent", "OpenAI Chat Model", and "Structured Output Parser".

6. **Connect AI Outputs:**  
   - Connect "Set Input Data" to "Prompt Agent".  
   - Connect "Google Gemini Chat Model" to "Prompt Agent" as AI language model input.  
   - Connect "Prompt Agent" to "Content writer".  
   - Connect "OpenAI Chat Model" and "Structured Output Parser" as AI inputs to "Content writer".

7. **Text-to-Speech Conversion:**  
   - Add an *ElevenLabs* TTS node configured with ElevenLabs API key ("Convert text to speech").  
   - Connect "Content writer" output to "Convert text to speech".

8. **Upload Audio to Cloud Storage:**  
   - Add a *Google Cloud Storage* node ("Upload to GCS Audio") configured with GCS credentials and target bucket. Enable retry with 2s intervals.  
   - Connect "Convert text to speech" to "Upload to GCS Audio".

9. **Manage Credentials:**  
   - Add a *Set* node ("SET Credentials") to prepare or refresh tokens for video generation API.  
   - Connect "Upload to GCS Audio" to "SET Credentials".

10. **Authenticate Video API:**  
    - Add a *JWT* node to generate JWT tokens for video generation API authentication.  
    - Connect "SET Credentials" to "JWT".

11. **Get API Token:**  
    - Add an *HTTP Request* node ("GET Token") configured to request access tokens using JWT.  
    - Connect "JWT" to "GET Token".

12. **Request Video Generation:**  
    - Add an *HTTP Request* node ("Generate Video1") to start video generation.  
    - Connect "GET Token" to "Generate Video1".

13. **Wait for Video Completion:**  
    - Add a *Wait* node ("Wait for Video") with webhook to pause until video rendering is done.  
    - Connect "Generate Video1" to "Wait for Video".

14. **Poll Video Status:**  
    - Add an *HTTP Request* node ("Fetch Status") to poll video rendering status.  
    - Connect "Wait for Video" to "Fetch Status".

15. **Branch on Status:**  
    - Add a *Switch* node with conditions based on "Fetch Status" output (e.g., if video ready).  
    - Connect "Fetch Status" to "Switch".

16. **If Ready, Convert Video:**  
    - Add a *Convert To File* node ("Convert to File") to convert video to 9:16 aspect ratio.  
    - Connect appropriate "Switch" output to "Convert to File".

17. **Upload Video to Cloud Storage:**  
    - Add a *Google Cloud Storage* node ("Upload to GCS Video"), configured with bucket and retry.  
    - Connect "Convert to File" to "Upload to GCS Video".

18. **Prepare Video Rendering Request:**  
    - Add a *Code* node ("Creatomate HTTP Body") to prepare HTTP body for Creatomate API.  
    - Connect "Upload to GCS Video" to "Creatomate HTTP Body".

19. **Start Video Rendering:**  
    - Add an *HTTP Request* node ("Merge - Creatomate") to send rendering request.  
    - Connect "Creatomate HTTP Body" to "Merge - Creatomate".

20. **Wait for Rendering Completion:**  
    - Add a *Wait* node ("Wait for Rendering") with webhook.  
    - Connect "Merge - Creatomate" to "Wait for Rendering".

21. **Check Rendering Status:**  
    - Add an *HTTP Request* node ("Check status of render") polling rendering status.  
    - Connect "Wait for Rendering" to "Check status of render".

22. **Conditional on Render Status:**  
    - Add an *If* node ("If3") to check if rendering is complete.  
    - Connect "Check status of render" to "If3".

23. **If Complete, Upload to Cloudinary:**  
    - Add an *HTTP Request* node ("Send to Cloudinary") to upload final video.  
    - Connect true branch of "If3" to "Send to Cloudinary".

24. **Generate Upscaled URL:**  
    - Add a *Code* node ("Make upscaled URL") to manipulate Cloudinary URL for upscaled video.  
    - Connect "Send to Cloudinary" to "Make upscaled URL".

25. **Download from Cloudinary:**  
    - Add an *HTTP Request* node ("Download file Cloudinary") to download video.  
    - Connect "Make upscaled URL" to "Download file Cloudinary".

26. **Upload Video to Postiz:**  
    - Add an *HTTP Request* node ("Upload video to Postiz") for Postiz platform upload.  
    - Connect "Download file Cloudinary" to "Upload video to Postiz".

27. **Fetch Postiz Integrations:**  
    - Add an *HTTP Request* node ("Get Postiz integrations") to list social media channels.  
    - Connect "Upload video to Postiz" to "Get Postiz integrations".

28. **Route to Scheduling:**  
    - Add a *Switch* node ("Switch1") to route to TikTok, YouTube, or Instagram scheduling based on integrations.  
    - Connect "Get Postiz integrations" to "Switch1".

29. **Schedule Posts:**  
    - Add three *HTTP Request* nodes: "Schedule TikTok", "Schedule YouTube", "Schedule Instagram" each with platform-specific API credentials.  
    - Connect "Switch1" outputs to each scheduling node respectively.

30. **Mark Quota Completion:**  
    - Add a *Google Sheets* node ("Mark Quota as Done") to update sheet marking task done.  
    - Connect "Schedule Instagram" to "Mark Quota as Done".

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| At the "Convert to File" node step, the video should already be generated and ready to be converted to 9:16 aspect ratio. | Node note in workflow                                |
| Uses ElevenLabs for high-quality text-to-speech audio generation.                                               | ElevenLabs TTS integration                           |
| Uses Creatomate API for advanced video rendering and merging.                                                   | Creatomate API documentation                         |
| Workflow supports multiple social media platforms via Postiz integration for scheduling.                         | Postiz API and platform integration                  |
| Retry mechanisms are enabled for Google Cloud Storage uploads with wait intervals to increase robustness.       | GCS Upload nodes configuration                        |
| Includes webhook-based wait nodes to effectively poll for asynchronous video rendering completion.              | Wait for Video and Wait for Rendering nodes          |

---

This documentation fully covers all nodes and logic in the workflow "AI Cinematic Short Generator," enabling reproduction, modification, and troubleshooting by advanced users or AI agents.