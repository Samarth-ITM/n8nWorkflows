Generate AI Video Avatars from URLs with HeyGen, Gemini & Upload to Social Media

https://n8nworkflows.xyz/workflows/generate-ai-video-avatars-from-urls-with-heygen--gemini---upload-to-social-media-10642


# Generate AI Video Avatars from URLs with HeyGen, Gemini & Upload to Social Media

### 1. Workflow Overview

This workflow automates the creation of AI-generated vertical short videos ("shorts") from any URL input, such as news articles, blog posts, or n8n workflow pages. It leverages AI to fetch and summarize webpage content, craft a concise 30–45 second script, generate captions/descriptions optimized for multiple social platforms, and produce a talking-head avatar video using HeyGen. The final video is then uploaded to multiple social media networks via Upload-Post. The workflow supports both free/trial and paid HeyGen API modes, with options for split-screen or background removal rendering styles.

The workflow is logically organized into these main blocks:

- **1.1 Input Reception & Variable Setup**: Manual trigger and setting core input variables including API keys, target URL, and rendering mode.
- **1.2 AI Content Generation**: Using Google Gemini to fetch webpage content, analyze it, and generate platform-specific scripts and captions.
- **1.3 Background Media Retrieval**: Obtaining either a scrolling video capture or a static image background of the URL.
- **1.4 Video Avatar Creation with HeyGen**: Conditional avatar video generation with HeyGen’s API, supporting free split-screen or paid background removal modes.
- **1.5 Video Status Polling and Retrieval**: Poll HeyGen until video rendering completes, then retrieve the final video URL.
- **1.6 Human Review and Approval**: Wait for manual approval before publishing.
- **1.7 Social Media Upload**: Upload the final video with platform-specific captions to social media via Upload-Post.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Variable Setup

- **Overview:** Begins the workflow manually and initializes all required variables and API keys for downstream nodes.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’ (Manual Trigger)
  - Set Input Vars (Set node)
  - Sticky Note nodes for documentation

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Inputs: None  
    - Outputs: Triggers Set Input Vars  
    - Edge cases: None typical; manual start.

  - **Set Input Vars**  
    - Type: Set  
    - Role: Define input parameters such as:  
      - `workflow_url`: URL of the page to convert into a video  
      - API keys for HeyGen, ScreenshotOne, Upload-Post  
      - HeyGen avatar and voice IDs  
      - `background_removal`: boolean string to select rendering mode  
      - `background_type`: either "video" or "photo"  
    - Inputs: From manual trigger  
    - Outputs: Feeds AI agent and background retrieval nodes  
    - Edge cases: Missing or invalid API keys will cause API failures downstream.

  - **Sticky Note nodes (Sticky Note2, Sticky Note3, Sticky Note13)**  
    - Purpose: Document workflow purpose, setup instructions, and key notes.  
    - No functional role.

#### 2.2 AI Content Generation

- **Overview:** Uses Google Gemini to download and analyze the target URL, then generates a short script and multiple social media captions optimized per platform.
- **Nodes Involved:** 
  - Social Media Agent (LangChain agent node)  
  - Google Gemini Chat Model (LM call)  
  - Google Gemini Chat Model1 (LM call)  
  - Structured Output Parser (to parse JSON output from AI)

- **Node Details:**

  - **Social Media Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates prompt sending and output from Google Gemini.  
    - Configuration:  
      - Custom prompt defines role/persona, analysis instructions, script requirements (30s, Spanish, simple language), caption formats per platform, output JSON format.  
      - Receives webpage content from HTTP Request node.  
    - Inputs: URL content from HTTP Request1  
    - Outputs: JSON with script and captions for TikTok, Instagram, YouTube Shorts, and HeyGen TTS bundle.  
    - Edge cases: If URL is unreachable or ambiguous, fallback script generation is triggered per prompt instructions.

  - **Google Gemini Chat Model** and **Google Gemini Chat Model1**  
    - Type: Language model nodes (Google PaLM API)  
    - Role: Process natural language prompts for summarization and script generation.  
    - Credentials: Google Gemini API key required.  
    - Inputs: Prompt from Social Media Agent and HTTP Request1.  
    - Outputs: Text response passed to Structured Output Parser.  
    - Edge cases: API quota limits, timeouts, or malformed prompts.

  - **Structured Output Parser**  
    - Type: LangChain output parser  
    - Role: Ensures AI output JSON is correctly parsed and validated.  
    - Inputs: AI text response.  
    - Outputs: Structured JSON to Social Media Agent.  
    - Edge cases: Malformed AI output causing parse errors.

  - **HTTP Request1**  
    - Type: HTTP Request Tool  
    - Role: Downloads raw HTML content of the provided URL for analysis.  
    - Inputs: URL from Set Input Vars.  
    - Outputs: HTML content to Social Media Agent.  
    - Edge cases: URL unreachable, timeout, or HTTP errors.

#### 2.3 Background Media Retrieval

- **Overview:** Conditionally obtains either a scrolling video capture or a static photo screenshot of the target webpage, used as the video background.
- **Nodes Involved:**  
  - video or photo (If node)  
  - Get background video (HTTP Request)  
  - Get background photo (HTTP Request)  
  - If1 (If node)

- **Node Details:**

  - **video or photo (If)**  
    - Type: If  
    - Role: Branches logic based on `background_type` variable ("video" vs "photo").  
    - Inputs: From Social Media Agent  
    - Outputs: Routes to either Get background video or Get background photo.  
    - Edge cases: Unexpected background_type values.

  - **Get background video**  
    - Type: HTTP Request  
    - Role: Calls ScreenshotOne API to generate a 30-second vertically scrolling MP4 video of the webpage.  
    - Parameters include viewport size, mobile emulation, ad blocking, scroll parameters, delay, and timeout.  
    - Inputs: URL and API key from Set Input Vars.  
    - Outputs: Video URL for HeyGen upload.  
    - Edge cases: API limits, timeout, video generation failure.

  - **Get background photo**  
    - Type: HTTP Request  
    - Role: Calls ScreenshotOne API to capture a static JPG screenshot of the webpage.  
    - Inputs: URL and API key.  
    - Outputs: Image URL.  
    - Edge cases: API limits, screenshot failure.

  - **If1**  
    - Type: If  
    - Role: Checks `background_removal` boolean to select between paid and free HeyGen video creation nodes.  
    - Inputs: From Get background photo.  
    - Outputs: Routes to appropriate HeyGen video creation node.  
    - Edge cases: Incorrect boolean strings.

#### 2.4 Video Avatar Creation with HeyGen

- **Overview:** Depending on `background_removal` and `background_type`, calls HeyGen API to generate the avatar video with appropriate background and rendering mode.
- **Nodes Involved:**  
  - upload to heygen (HTTP Request If node)  
  - Create video avatar with background removal (HTTP Request)  
  - Create video avatar with split screen (heygen free trial api) (HTTP Request)  
  - Create video avatar with background removal1 (HTTP Request)  
  - Create video avatar with split screen (heygen free trial api)1 (HTTP Request)  
  - Edit Fields (Set node)

- **Node Details:**

  - **upload to heygen**  
    - Type: HTTP Request  
    - Role: Uploads the background media (video or photo) to HeyGen asset server.  
    - Inputs: Binary data (video or photo) from previous step, HeyGen API key.  
    - Outputs: Asset URL for background in video generation.  
    - Edge cases: Upload failures, invalid API key.

  - **Create video avatar with background removal / Create video avatar with background removal1**  
    - Type: HTTP Request  
    - Role: Generate HeyGen avatar video with background removal enabled (`matting: true`) for clean overlay.  
    - Background can be video or static image depending on previous logic.  
    - Inputs: Avatar ID, voice ID, script text, background URL from upload step, HeyGen API key.  
    - Outputs: Video creation job ID.  
    - Edge cases: API quota, invalid avatar or voice IDs, malformed JSON, network errors.

  - **Create video avatar with split screen (heygen free trial api) / Create video avatar with split screen (heygen free trial api)1**  
    - Type: HTTP Request  
    - Role: Generate HeyGen avatar video with split-screen layout (no background removal).  
    - Background video or photo displayed in upper half, avatar in lower half.  
    - Inputs: Same as above.  
    - Outputs: Video creation job ID.  
    - Edge cases: As above.

  - **Edit Fields**  
    - Type: Set  
    - Role: Extracts and sets the `video_id` from HeyGen response for polling.  
    - Outputs: Feeds Wait node.

#### 2.5 Video Status Polling and Retrieval

- **Overview:** Poll HeyGen API to check the rendering status of the video periodically until completion, then retrieve the downloadable video URL.
- **Nodes Involved:**  
  - Wait (Wait node)  
  - Get Avatar Video (HTTP Request)  
  - If Video Done (If node)  
  - Get video url (HTTP Request)  
  - Edit Fields  
  - Wait to approve by user1 (Wait node for manual form)

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 1 minute between status checks.  
    - Inputs: Video ID from Edit Fields.  
    - Outputs: Get Avatar Video request.

  - **Get Avatar Video**  
    - Type: HTTP Request  
    - Role: Queries HeyGen API for video job status using `video_id`.  
    - Inputs: video_id, HeyGen API key.  
    - Outputs: Video status JSON.  
    - Edge cases: API errors, invalid video_id.

  - **If Video Done**  
    - Type: If  
    - Role: Checks if video status is "completed".  
    - If yes, proceeds to Get video url; if no, loops back to Wait.  
    - Edge cases: Unexpected statuses.

  - **Get video url**  
    - Type: HTTP Request  
    - Role: Retrieves final downloadable MP4 URL from HeyGen.  
    - Inputs: video_id, HeyGen API key.  
    - Outputs: Video URL for user review.  
    - Edge cases: API errors.

  - **Wait to approve by user1**  
    - Type: Wait (form-based)  
    - Role: Pauses workflow for manual approval with a form presenting video URL and a dropdown decision (approve/reject).  
    - Inputs: Video URL.  
    - Outputs: Decision output for next step.  
    - Edge cases: User delay or no response.

#### 2.6 Human Review and Approval

- **Overview:** Checks user's approval decision from the form and conditionally proceeds to publish or halt.
- **Nodes Involved:**  
  - If approved publish1 (If node)  
  - Sticky Note11 (documentation)

- **Node Details:**

  - **If approved publish1**  
    - Type: If  
    - Role: Checks if user decision is "approve".  
    - If true, proceeds to Upload video to all social networks.  
    - If false, workflow ends or could be configured to loop back (not shown).  
    - Edge cases: Invalid or missing decision.

#### 2.7 Social Media Upload

- **Overview:** Uploads the finalized video to multiple social media platforms via Upload-Post with platform-specific captions.
- **Nodes Involved:**  
  - Upload video to all social networks (Upload-Post node)  
  - Sticky Note10 (documentation)

- **Node Details:**

  - **Upload video to all social networks**  
    - Type: Upload-Post node (custom integration)  
    - Role: Uploads the video file to selected platforms (Instagram Reels, TikTok, YouTube) using provided titles and captions.  
    - Inputs: Video URL, platform captions from Social Media Agent outputs, Upload-Post credentials.  
    - Outputs: Upload response/status.  
    - Edge cases: API rate limits, credential expiration, upload failures.

---

### 3. Summary Table

| Node Name                                   | Node Type                          | Functional Role                              | Input Node(s)                     | Output Node(s)                             | Sticky Note                                      |
|---------------------------------------------|----------------------------------|----------------------------------------------|----------------------------------|--------------------------------------------|-------------------------------------------------|
| When clicking ‘Execute workflow’             | Manual Trigger                   | Entry point for manual workflow start        | None                             | Set Input Vars                             |                                                 |
| Set Input Vars                               | Set                              | Define variables and API keys                 | When clicking ‘Execute workflow’ | Social Media Agent                         |                                                 |
| Social Media Agent                           | LangChain Agent                  | Generate script and social captions           | Set Input Vars, HTTP Request1    | video or photo                            |                                                 |
| Google Gemini Chat Model                      | LM Chat Model (Google Gemini)    | Language model for content generation         | Social Media Agent               | Social Media Agent                         |                                                 |
| Google Gemini Chat Model1                     | LM Chat Model (Google Gemini)    | Language model for parsing structured output  | Structured Output Parser          | Structured Output Parser                   |                                                 |
| Structured Output Parser                      | Output Parser                   | Parse AI-generated JSON output                 | Google Gemini Chat Model1         | Social Media Agent                         |                                                 |
| HTTP Request1                                | HTTP Request Tool                | Download HTML content from URL                 | Set Input Vars                   | Social Media Agent                         |                                                 |
| video or photo                               | If                              | Branch to get video or photo background       | Social Media Agent               | Get background video / Get background photo |                                                 |
| Get background video                         | HTTP Request                    | Get scrolling background video from ScreenshotOne | video or photo                   | upload to heygen                          |                                                 |
| Get background photo                         | HTTP Request                    | Get static photo background from ScreenshotOne | video or photo                   | If1                                       |                                                 |
| If1                                          | If                              | Branch based on background_removal boolean    | Get background photo             | Create video avatar with background removal1 / Create video avatar with split screen (heygen free trial api)1 |                                                 |
| upload to heygen                             | HTTP Request                    | Upload background media to HeyGen              | Get background video             | If                                         |                                                 |
| If                                           | If                              | Branch based on background_removal boolean    | upload to heygen                 | Create video avatar with background removal / Create video avatar with split screen (heygen free trial api) |                                                 |
| Create video avatar with background removal | HTTP Request                    | Generate HeyGen avatar video with background removal (video background) | If                             | Edit Fields                                |                                                 |
| Create video avatar with split screen (heygen free trial api) | HTTP Request                    | Generate HeyGen split-screen avatar video (video background) | If                             | Edit Fields                                |                                                 |
| Create video avatar with background removal1 | HTTP Request                    | Generate HeyGen avatar video with background removal (static image background) | If1                            | Edit Fields                                |                                                 |
| Create video avatar with split screen (heygen free trial api)1 | HTTP Request                    | Generate HeyGen split-screen avatar video (static image background) | If1                            | Edit Fields                                |                                                 |
| Edit Fields                                  | Set                              | Extract video ID from HeyGen response          | Video creation nodes             | Wait                                       |                                                 |
| Wait                                         | Wait                            | Pause between polling attempts                 | Edit Fields                     | Get Avatar Video                           |                                                 |
| Get Avatar Video                             | HTTP Request                    | Poll HeyGen video status                        | Wait                           | If Video Done                              |                                                 |
| If Video Done                               | If                              | Check if HeyGen video job is completed         | Get Avatar Video                | Get video url / Wait                       |                                                 |
| Get video url                               | HTTP Request                    | Retrieve final video URL from HeyGen            | If Video Done                  | Wait to approve by user1                    |                                                 |
| Wait to approve by user1                     | Wait (Form)                    | Pause for manual approval with form             | Get video url                  | If approved publish1                       |                                                 |
| If approved publish1                         | If                              | Check if user approved video for publishing    | Wait to approve by user1         | Upload video to all social networks         |                                                 |
| Upload video to all social networks          | Upload-Post Upload Node          | Upload video with captions to social media     | If approved publish1             | None                                       |                                                 |
| Sticky Note2                                | Sticky Note                    | Workflow overview and description               | None                           | None                                       |                                                 |
| Sticky Note3                                | Sticky Note                    | Setup instructions and account notes            | None                           | None                                       |                                                 |
| Sticky Note10                               | Sticky Note                    | Post to all social networks label                | None                           | None                                       |                                                 |
| Sticky Note11                               | Sticky Note                    | Human approval step explanation                   | None                           | None                                       |                                                 |
| Sticky Note12                               | Sticky Note                    | HeyGen video creation modes explanation          | None                           | None                                       |                                                 |
| Sticky Note13                               | Sticky Note                    | Note about background removal requiring paid API | None                           | None                                       |                                                 |
| Sticky Note1                                | Sticky Note                    | Video demo reference                              | None                           | None                                       |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - Purpose: Entry point to start the workflow manually.

2. **Create Set Node for Input Variables**  
   - Name: `Set Input Vars`  
   - Add variables with appropriate default or empty values:  
     - `workflow_url` (string): target URL to process  
     - `screenshotone.comAPI_KEY` (string): API key for ScreenshotOne  
     - `heygen_api_key` (string): HeyGen API key  
     - `avatar_id` (string): HeyGen avatar ID  
     - `voice_id` (string): HeyGen voice ID  
     - `background_removal` (string): `"true"` or `"false"` to select HeyGen rendering mode  
     - `background_type` (string): `"video"` or `"photo"` to choose background media type

3. **Create HTTP Request Node to Download Webpage HTML**  
   - Name: `HTTP Request1`  
   - Method: GET  
   - URL: Expression referencing `workflow_url` from `Set Input Vars`  
   - Purpose: Fetch raw page content for AI analysis.

4. **Create LangChain Agent Node for AI Script & Captions Generation**  
   - Name: `Social Media Agent`  
   - Configure prompt with detailed instructions for:  
     - Extracting main idea and details from HTML  
     - Writing a ~30-second Spanish script, captions, and titles for TikTok, Instagram, YouTube Shorts  
     - Output JSON format (strict) with platforms and fields  
   - Connect input from `Set Input Vars` and `HTTP Request1`.

5. **Add Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Credentials: Google Gemini API key  
   - Role: Perform AI processing of prompt from `Social Media Agent`.

6. **Add Structured Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Configure with JSON schema example matching expected AI output.  
   - Connect output of `Google Gemini Chat Model` to this parser.

7. **Add Second Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model1`  
   - Connect output of `Structured Output Parser` to this node with appropriate chaining.

8. **Create If Node to Branch Background Type**  
   - Name: `video or photo`  
   - Condition: Check if `background_type` equals `"video"`  
   - True output: `Get background video` node  
   - False output: `Get background photo` node

9. **Create HTTP Request Node: Get background video**  
   - Name: `Get background video`  
   - URL: ScreenshotOne animate API with parameters for scrolling MP4 video  
   - Use API key and `workflow_url` from `Set Input Vars`.

10. **Create HTTP Request Node: Get background photo**  
    - Name: `Get background photo`  
    - URL: ScreenshotOne take API for JPG screenshot  
    - Use API key and `workflow_url`.

11. **Create If Node `If1` to branch based on background_removal (for photo background case)**  
    - Condition: Check if `background_removal` is `"true"`  
    - True output: `Create video avatar with background removal1`  
    - False output: `Create video avatar with split screen (heygen free trial api)1`

12. **Create HTTP Request Node: Upload to HeyGen**  
    - Name: `upload to heygen`  
    - POST to `https://upload.heygen.com/v1/asset`  
    - Content-Type: binaryData  
    - Use HeyGen API key from `Set Input Vars`  
    - Upload media from `Get background video` or `Get background photo` nodes.

13. **Create If Node `If` to branch based on background_removal (for video background case)**  
    - Condition: Check if `background_removal` is `"true"`  
    - True output: `Create video avatar with background removal`  
    - False output: `Create video avatar with split screen (heygen free trial api)`

14. **Create HTTP Request Nodes for HeyGen video generation:**  
    - `Create video avatar with background removal`  
      - POST to `https://api.heygen.com/v2/video/generate`  
      - JSON body includes avatar_id, voice_id, script text, background video URL, `matting: true`  
    - `Create video avatar with split screen (heygen free trial api)`  
      - Same endpoint but without `matting` and with split-screen settings  
    - Similarly, for photo background:  
      - `Create video avatar with background removal1` (with image background and matting)  
      - `Create video avatar with split screen (heygen free trial api)1` (with photo background no matting)

15. **Create Set Node `Edit Fields`**  
    - Extract `video_id` from HeyGen video generation response JSON for further polling.

16. **Create Wait Node**  
    - Wait 1 minute between status polls.

17. **Create HTTP Request Node `Get Avatar Video`**  
    - GET `https://api.heygen.com/v1/video_status.get` with `video_id` query parameter  
    - Use HeyGen API key

18. **Create If Node `If Video Done`**  
    - Check if video status equals `"completed"`  
    - True: Move to get video url  
    - False: Loop back to Wait

19. **Create HTTP Request Node `Get video url`**  
    - GET final video URL from HeyGen  
    - Input: `video_id`, HeyGen API key

20. **Create Wait Node `Wait to approve by user1`**  
    - Wait for manual form approval  
    - Form fields: dropdown with options "approve" and "reject"  
    - Show video URL in form description

21. **Create If Node `If approved publish1`**  
    - Check if decision equals "approve"  
    - True: proceed to upload to social media

22. **Create Upload-Post Node `Upload video to all social networks`**  
    - Use Upload-Post credentials  
    - Upload video URL with captions for TikTok, Instagram, YouTube Shorts from `Social Media Agent` outputs  
    - Platforms enabled: instagram, tiktok, youtube  
    - Instagram media type: REELS

23. **Connect all nodes according to the logical flow:**  
    - Manual trigger → Set Input Vars → HTTP Request1 → Social Media Agent → video or photo → background retrieval → upload to heygen → If → video creation → Edit Fields → Wait → Get Avatar Video → If Video Done → Get video URL → Wait to approve → If approved → Upload video to all social networks

24. **Add Sticky Notes to document the workflow purpose, setup instructions, HeyGen modes, and approval steps for clarity.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                              | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This AI Avatar Viral News system turns any URL into a short vertical talking-head AI video with no filming or editing required. Supports free/trial and paid HeyGen API rendering modes.                                                                                  | Sticky Note2 content                                                                                         |
| HeyGen rendering modes: Free/Trial uses split-screen; Paid API enables background removal for clean avatar overlays. Pricing and plans at https://www.heygen.com/api-pricing                                                                                              | Sticky Note12 content                                                                                        |
| Background removal requires HeyGen paid API plan and setting `background_removal` variable to `true`.                                                                                                                                                                     | Sticky Note13 content                                                                                        |
| Setup requires API keys for HeyGen, ScreenshotOne (for background media), Upload-Post (for social upload), and Google Gemini API. Optionally, ElevenLabs voice can be configured inside HeyGen.                                                                               | Sticky Note3 content                                                                                         |
| Upload-Post allows scheduling and publishing videos to multiple social media platforms including TikTok, Instagram, YouTube Shorts, Facebook, and X (Twitter).                                                                                                           | Sticky Note10 content                                                                                        |
| Video demo reference available on YouTube: https://youtu.be/fh3t6Kc2oRM                                                                                                                                                                                                   | Sticky Note1 content                                                                                          |

---

This structured document provides a comprehensive, stepwise explanation of the entire workflow, enabling advanced users and AI agents to fully understand, reproduce, or modify the AI Video Avatar Generator pipeline.