Generate & Publish Professional Video Ads with Veo 3, Gemini & Creatomate

https://n8nworkflows.xyz/workflows/generate---publish-professional-video-ads-with-veo-3--gemini---creatomate-8263


# Generate & Publish Professional Video Ads with Veo 3, Gemini & Creatomate

### 1. Workflow Overview

This workflow automates the generation, processing, and multi-platform publishing of professional video advertisements using Veo 3, Google Gemini AI models, Creatomate video rendering, and various social media platforms. Its target use case is marketing teams or content creators who want to automate the creative ideation, video creation, analysis, and distribution pipeline for video ads.

The workflow is logically divided into the following key blocks:

- **1.1 Input Initialization:** Manual trigger and initial data setup.
- **1.2 AI-Driven Ad Planning and Script Generation:** Uses OpenAI and Google Gemini language models to plan ad structure and generate content parts.
- **1.3 Video Generation and Conversion:** Requests video rendering, waits for completion, converts videos to target formats/aspect ratios.
- **1.4 Video Upload and Analysis:** Uploads videos to Gemini for AI-based video analysis.
- **1.5 Video Merge and Final Rendering:** Combines video parts and triggers final rendering through Creatomate.
- **1.6 Video Download and Social Publishing:** Downloads final videos and schedules posts on social media platforms like TikTok, YouTube, Instagram, and Facebook.
- **1.7 Social Media Content Generation:** Generates supporting social media content using AI models.
- **1.8 Credential and Token Management:** Handles authentication tokens for API calls.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** Starts the workflow manually and sets initial variables/credentials.
- **Nodes Involved:** When clicking ‘Execute workflow’, Form, Set Credintenails
- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Entry point to manually start the workflow.
    - Configuration: Default manual trigger, no parameters.
    - Inputs: None
    - Outputs: Form node
    - Failure Types: None
    - Version: 1

  - **Form**
    - Type: Set
    - Role: Presumably sets or prepares initial form data or variables (empty parameters).
    - Configuration: Empty; likely a placeholder or structured data initializer.
    - Inputs: When clicking ‘Execute workflow’
    - Outputs: Set Credintenails
    - Failure Types: None
    - Version: 3.4

  - **Set Credintenails**
    - Type: Set
    - Role: Sets API credentials or related parameters for downstream nodes.
    - Configuration: Not explicitly shown, but likely contains credentials or environment variables.
    - Inputs: Form
    - Outputs: Planning the ad
    - Failure Types: None
    - Version: 3.4

---

#### 2.2 AI-Driven Ad Planning and Script Generation

- **Overview:** Uses chained AI language model calls to plan the advertisement and generate script parts.
- **Nodes Involved:** Planning the ad, Merge, Analyze part 1, Part_1, Part_2, OpenAI Chat Model, Google Gemini Chat Model variants, Structured Output Parser nodes
- **Node Details:**

  - **Planning the ad**
    - Type: LangChain Chain LLM
    - Role: Generates the overall ad plan using AI.
    - Configuration: Uses AI language models (OpenAI, Gemini) as input.
    - Inputs: Set Credintenails (for context), OpenAI Chat Model (ai_languageModel), Structured Output Parser (ai_outputParser)
    - Outputs: Merge, Write content for social media, Part_1
    - Failure Types: AI call failures, parsing errors
    - Version: 1.7

  - **Merge**
    - Type: Merge
    - Role: Combines data from planning and video analysis for further processing.
    - Configuration: Standard merge combining multiple inputs.
    - Inputs: Analyze Video with Gemini, Planning the ad
    - Outputs: Analyze part 1
    - Failure Types: Data mismatch, missing inputs
    - Version: 3.2

  - **Analyze part 1**
    - Type: LangChain Chain LLM
    - Role: Further analyzes part of the ad content.
    - Configuration: Uses Google Gemini Chat Model3 as AI input; output parsed by Structured Output Parser1.
    - Inputs: Merge, Google Gemini Chat Model3, Structured Output Parser1
    - Outputs: Part_2
    - Failure Types: AI errors, parsing issues
    - Version: 1.7

  - **Part_1 and Part_2**
    - Type: LangChain Chain LLM
    - Role: Generate segments or parts of the ad script or creative content.
    - Configuration: Each receives AI model input and outputs to next step.
    - Inputs: Planning the ad (for Part_1), Analyze part 1 (for Part_2)
    - Outputs: JWT (Part_1), Generate Video1 (Part_2)
    - Failure Types: AI call failures
    - Version: 1.7

  - **OpenAI Chat Model (multiple instances)**
    - Type: LangChain LLM OpenAI
    - Role: Provides language model capabilities to generate text or plan.
    - Configuration: OpenAI credentials, prompt templates (not shown).
    - Inputs/Outputs: Connected to respective Chain LLM nodes.
    - Failure Types: API auth errors, rate limits
    - Version: 1.2

  - **Google Gemini Chat Model (multiple instances)**
    - Type: LangChain LLM Google Gemini
    - Role: Alternative or complementary AI model for text generation and analysis.
    - Configuration: Google Gemini API credentials.
    - Inputs/Outputs: Connected to Chain LLM or Output Parser nodes.
    - Failure Types: API errors, timeouts
    - Version: 1

  - **Structured Output Parser (multiple instances)**
    - Type: LangChain Output Parser
    - Role: Parses structured AI outputs into usable data.
    - Configuration: Custom parsing templates or schema.
    - Inputs/Outputs: Connected between AI models and Chain LLM nodes.
    - Failure Types: Parsing failures, malformed AI output
    - Version: 1.3

---

#### 2.3 Video Generation and Conversion

- **Overview:** Handles video generation API calls, waits for processing, and converts videos to the required aspect ratio (9:16).
- **Nodes Involved:** JWT, GET TOKEN, Generate Video, Wait, Fetch Status, Switch, Convert to File, Post video Cloudinary Part1/Part 2
- **Node Details:**

  - **JWT**
    - Type: JWT Node
    - Role: Generates JWT token for authentication.
    - Configuration: Likely configured with secret keys.
    - Inputs: Part_1
    - Outputs: GET TOKEN
    - Failure Types: Token generation errors
    - Version: 1

  - **GET TOKEN**
    - Type: HTTP Request
    - Role: Obtains access token using JWT.
    - Configuration: HTTP POST to auth endpoint.
    - Inputs: JWT
    - Outputs: Generate Video
    - Failure Types: Auth failures, network errors
    - Version: 4.2

  - **Generate Video / Generate Video1**
    - Type: HTTP Request
    - Role: Calls API to generate video based on script/content.
    - Configuration: Posts video generation request with content.
    - Inputs: GET TOKEN (Generate Video), Part_2 (Generate Video1)
    - Outputs: Wait, Wait1
    - Failure Types: API errors, request timeouts
    - Version: 4.2

  - **Wait / Wait1**
    - Type: Wait Node
    - Role: Pauses workflow while video generation completes.
    - Configuration: Uses webhook ID for async wait.
    - Inputs: Generate Video, Generate Video1
    - Outputs: Fetch Status, Fetch Status1
    - Failure Types: Timeout, missed webhook triggers
    - Version: 1.1

  - **Fetch Status / Fetch Status1**
    - Type: HTTP Request
    - Role: Polls to check video generation status.
    - Configuration: GET request to status endpoint.
    - Inputs: Wait, Wait1
    - Outputs: Switch, Switch1
    - Failure Types: Network errors, invalid status responses
    - Version: 4.2

  - **Switch / Switch1**
    - Type: Switch Node
    - Role: Branches logic based on status or response.
    - Configuration: Conditions based on status values.
    - Inputs: Fetch Status, Fetch Status1
    - Outputs: Convert to File / Part_1 / Wait (Switch), Convert to File1 / Part_2 / Wait1 (Switch1)
    - Failure Types: Misconfigured conditions
    - Version: 3.2

  - **Convert to File / Convert to File1**
    - Type: Convert to File
    - Role: Converts generated video data to 9:16 aspect ratio file.
    - Configuration: Conversion settings for video format/resolution.
    - Inputs: Switch, Switch1
    - Outputs: Post video Cloudinary Part1, Post video Cloudinary Part 2
    - Failure Types: Conversion errors, unsupported formats
    - Version: 1.1

  - **Post video Cloudinary Part1 / Part 2**
    - Type: HTTP Request
    - Role: Uploads video parts to Cloudinary CDN for storage.
    - Configuration: POST requests with video binary or URL.
    - Inputs: Convert to File, Convert to File1
    - Outputs: Begin Gemini Upload Session, Shotstack HTTP Body
    - Failure Types: Upload failures, auth errors
    - Version: 4.2

---

#### 2.4 Video Upload and Analysis

- **Overview:** Uploads videos to Gemini AI for detailed video content analysis.
- **Nodes Involved:** Begin Gemini Upload Session, Downloading Part 1 (Binary)1, Upload Video to Gemini, Waiting..., Analyze Video with Gemini, Merge
- **Node Details:**

  - **Begin Gemini Upload Session**
    - Type: HTTP Request
    - Role: Initiates an upload session with Gemini API.
    - Configuration: POST request to start session.
    - Inputs: Post video Cloudinary Part1
    - Outputs: Downloading Part 1 (Binary)1
    - Failure Types: API errors, session failures
    - Version: 4.2

  - **Downloading Part 1 (Binary)1**
    - Type: HTTP Request
    - Role: Downloads binary video data for upload.
    - Configuration: GET request for file download.
    - Inputs: Begin Gemini Upload Session
    - Outputs: Upload Video to Gemini
    - Failure Types: Download errors, network issues
    - Version: 4.2

  - **Upload Video to Gemini**
    - Type: HTTP Request
    - Role: Uploads video binary to Gemini.
    - Configuration: POST with binary data.
    - Inputs: Downloading Part 1 (Binary)1
    - Outputs: Waiting...
    - Failure Types: Upload failures
    - Version: 4.2

  - **Waiting...**
    - Type: Wait Node
    - Role: Pauses for video processing by Gemini.
    - Configuration: Uses webhook ID.
    - Inputs: Upload Video to Gemini
    - Outputs: Analyze Video with Gemini
    - Failure Types: Timeout, webhook errors
    - Version: 1.1

  - **Analyze Video with Gemini**
    - Type: HTTP Request
    - Role: Requests analysis results from Gemini.
    - Configuration: GET request with retry (wait 15s between tries).
    - Inputs: Waiting...
    - Outputs: Merge
    - Failure Types: API errors, retry exhaustion
    - Version: 4.2

  - **Merge**
    - (Described above in AI analysis section; merges analysis results)

---

#### 2.5 Video Merge and Final Rendering

- **Overview:** Combines video parts and sends the merged video to Creatomate for final rendering.
- **Nodes Involved:** Shotstack HTTP Body, Merge - Creatomate, Rendering...., done?, Switch2, Download final video
- **Node Details:**

  - **Shotstack HTTP Body**
    - Type: Code
    - Role: Prepares HTTP body for Creatomate merge request.
    - Configuration: Custom JavaScript code to build request payload.
    - Inputs: Post video Cloudinary Part 2
    - Outputs: Merge - Creatomate
    - Failure Types: Code errors, malformed data
    - Version: 2

  - **Merge - Creatomate**
    - Type: HTTP Request
    - Role: Sends video merge/render request to Creatomate API.
    - Configuration: POST request with prepared body.
    - Inputs: Shotstack HTTP Body
    - Outputs: Rendering....
    - Failure Types: API errors, auth failures
    - Version: 4.2

  - **Rendering....**
    - Type: Wait
    - Role: Waits asynchronously for rendering completion.
    - Configuration: Webhook-based wait node.
    - Inputs: Merge - Creatomate
    - Outputs: done?
    - Failure Types: Timeout, webhook failures
    - Version: 1.1

  - **done?**
    - Type: HTTP Request
    - Role: Polls Creatomate or rendering service to check if final rendering is done.
    - Configuration: GET request.
    - Inputs: Rendering....
    - Outputs: Switch2
    - Failure Types: API errors, response parsing
    - Version: 4.2

  - **Switch2**
    - Type: Switch
    - Role: Branches based on rendering status.
    - Configuration: Routes to Download final video, Shotstack HTTP Body (retry), or Rendering.... (wait).
    - Inputs: done?
    - Outputs: Download final video, Shotstack HTTP Body, Rendering....
    - Failure Types: Misrouting, condition misconfiguration
    - Version: 3.2

  - **Download final video**
    - Type: HTTP Request
    - Role: Downloads the fully rendered video.
    - Configuration: GET request to rendering server.
    - Inputs: Switch2
    - Outputs: Upload video to Postiz
    - Failure Types: Download errors
    - Version: 4.2

---

#### 2.6 Video Upload and Social Publishing

- **Overview:** Uploads the final video to Postiz and schedules posts on various social media platforms.
- **Nodes Involved:** Upload video to Postiz, Get Postiz integrations, Merge1, Switch3, Schedule TikTok, Schedule YouTube, Schedule Instagram, Schedule Facebook
- **Node Details:**

  - **Upload video to Postiz**
    - Type: HTTP Request
    - Role: Uploads final video to Postiz platform.
    - Configuration: POST request with video data.
    - Inputs: Download final video
    - Outputs: Get Postiz integrations
    - Failure Types: Upload errors, auth issues
    - Version: 4.2

  - **Get Postiz integrations**
    - Type: HTTP Request
    - Role: Fetches configured social media integrations from Postiz.
    - Configuration: GET request.
    - Inputs: Upload video to Postiz
    - Outputs: Merge1
    - Failure Types: API errors
    - Version: 4.2

  - **Merge1**
    - Type: Merge
    - Role: Combines integration data with social content.
    - Inputs: Get Postiz integrations, Write content for social media
    - Outputs: Switch3
    - Failure Types: Input mismatch
    - Version: 3.2

  - **Switch3**
    - Type: Switch
    - Role: Routes to appropriate social media scheduling node based on integration type.
    - Inputs: Merge1
    - Outputs: Schedule TikTok, Schedule YouTube, Schedule Instagram, Schedule Facebook
    - Failure Types: Condition misconfiguration
    - Version: 3.2

  - **Schedule TikTok / YouTube / Instagram / Facebook**
    - Type: HTTP Request
    - Role: Calls APIs to schedule video posts on respective platforms.
    - Configuration: POST with scheduling parameters.
    - Inputs: Switch3
    - Outputs: None
    - Failure Types: API errors, auth failures, rate limits
    - Version: 4.2

---

#### 2.7 Social Media Content Generation

- **Overview:** Generates social media post content using AI models and parses structured output.
- **Nodes Involved:** Write content for social media, OpenAI Chat Model3, Structured Output Parser2, OpenAI Chat Model4
- **Node Details:**

  - **Write content for social media**
    - Type: LangChain Chain LLM
    - Role: Generates captions, hashtags, or posts text for social media.
    - Configuration: Uses AI input from OpenAI Chat Model3 and output parser.
    - Inputs: Planning the ad, OpenAI Chat Model3, Structured Output Parser2
    - Outputs: Merge1
    - Failure Types: AI errors, parsing failures
    - Version: 1.7

  - **OpenAI Chat Model3 / OpenAI Chat Model4**
    - Type: LangChain LLM OpenAI
    - Role: Provide text generation capabilities.
    - Configuration: Connected to Write content for social media and Structured Output Parser2.
    - Inputs: Write content for social media, Structured Output Parser2
    - Outputs: Write content for social media, Structured Output Parser2
    - Failure Types: API auth, rate limits
    - Version: 1.2

  - **Structured Output Parser2**
    - Type: LangChain Output Parser
    - Role: Parses AI-generated social media content.
    - Inputs: OpenAI Chat Model4
    - Outputs: Write content for social media
    - Failure Types: Parsing errors
    - Version: 1.3

---

#### 2.8 Credential and Token Management

- **Overview:** Manages API authentication tokens for subsequent API calls (video generation, upload, etc.).
- **Nodes Involved:** JWT, GET TOKEN
- **Node Details:**

  - Refer to nodes described in Video Generation and Conversion section.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                            | Input Node(s)                      | Output Node(s)                    | Sticky Note                                        |
|-------------------------------|--------------------------------|--------------------------------------------|----------------------------------|----------------------------------|---------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Entry point for manual execution             | None                             | Form                             |                                                   |
| Form                          | Set                            | Initializes form data                        | When clicking ‘Execute workflow’ | Set Credintenails                |                                                   |
| Set Credintenails             | Set                            | Sets API credentials                         | Form                             | Planning the ad                  |                                                   |
| Planning the ad               | Chain LLM                      | AI-driven ad planning                        | Set Credintenails, OpenAI Chat Model, Structured Output Parser | Merge, Write content for social media, Part_1 |                                                   |
| OpenAI Chat Model             | LLM OpenAI                    | Provides AI text generation                  | Planning the ad                  | Planning the ad                  |                                                   |
| Structured Output Parser      | Output Parser                 | Parses AI outputs                            | OpenAI Chat Model                | Planning the ad                  |                                                   |
| Merge                        | Merge                         | Combines data from AI and video analysis    | Analyze Video with Gemini, Planning the ad | Analyze part 1                 |                                                   |
| Analyze part 1               | Chain LLM                     | Further AI analysis of ad content            | Merge, Google Gemini Chat Model3, Structured Output Parser1 | Part_2                         |                                                   |
| Google Gemini Chat Model3     | LLM Google Gemini             | AI model for analysis                        | Analyze part 1                  | Analyze part 1                  |                                                   |
| Structured Output Parser1     | Output Parser                 | Parses Gemini AI output                      | Google Gemini Chat Model3         | Analyze part 1                 |                                                   |
| Part_1                       | Chain LLM                     | Generates first video script part            | Planning the ad                  | JWT                            |                                                   |
| JWT                          | JWT                          | Generates authentication token               | Part_1                         | GET TOKEN                      |                                                   |
| GET TOKEN                    | HTTP Request                 | Obtains access token                          | JWT                            | Generate Video                 |                                                   |
| Generate Video               | HTTP Request                 | Requests video generation                     | GET TOKEN                      | Wait                          |                                                   |
| Wait                         | Wait                         | Waits for video generation webhook           | Generate Video                 | Fetch Status                  |                                                   |
| Fetch Status                 | HTTP Request                 | Polls video generation status                 | Wait                          | Switch                       |                                                   |
| Switch                      | Switch                       | Branches on video status                       | Fetch Status                  | Convert to File, Part_1, Wait |                                                   |
| Convert to File              | Convert to File              | Converts video to 9:16 aspect ratio           | Switch                       | Post video Cloudinary Part1    | At this step, the video should be generated and ready to convert to 9:16 aspect ratio |
| Post video Cloudinary Part1  | HTTP Request                 | Uploads video part 1 to Cloudinary             | Convert to File               | Begin Gemini Upload Session    |                                                   |
| Begin Gemini Upload Session  | HTTP Request                 | Starts Gemini video upload session             | Post video Cloudinary Part1   | Downloading Part 1 (Binary)1   |                                                   |
| Downloading Part 1 (Binary)1 | HTTP Request                 | Downloads binary video for upload              | Begin Gemini Upload Session   | Upload Video to Gemini         |                                                   |
| Upload Video to Gemini       | HTTP Request                 | Uploads video to Gemini                         | Downloading Part 1 (Binary)1  | Waiting...                    |                                                   |
| Waiting...                  | Wait                         | Waits for Gemini video processing              | Upload Video to Gemini        | Analyze Video with Gemini      |                                                   |
| Analyze Video with Gemini    | HTTP Request                 | Requests Gemini video analysis                  | Waiting...                   | Merge                        |                                                   |
| Analyze part 1              | Chain LLM                     | AI analysis continuation                       | Merge                        | Part_2                       |                                                   |
| Part_2                      | Chain LLM                     | Generates second video script part             | Analyze part 1               | Generate Video1               |                                                   |
| Generate Video1             | HTTP Request                 | Requests second video generation                | Part_2                      | Wait1                        |                                                   |
| Wait1                       | Wait                         | Waits for second video generation              | Generate Video1              | Fetch Status1                |                                                   |
| Fetch Status1               | HTTP Request                 | Polls second video generation status           | Wait1                        | Switch1                     |                                                   |
| Switch1                    | Switch                       | Branches on second video status                 | Fetch Status1                | Convert to File1, Part_2, Wait1 |                                                   |
| Convert to File1           | Convert to File              | Converts second video to 9:16 aspect ratio      | Switch1                     | Post video Cloudinary Part 2 | At this step, the video should be generated and ready to convert to 9:16 aspect ratio |
| Post video Cloudinary Part 2 | HTTP Request                 | Uploads video part 2 to Cloudinary             | Convert to File1             | Shotstack HTTP Body          |                                                   |
| Shotstack HTTP Body         | Code                         | Prepares Creatomate HTTP request body           | Post video Cloudinary Part 2 | Merge - Creatomate           |                                                   |
| Merge - Creatomate          | HTTP Request                 | Sends merge/render request to Creatomate API     | Shotstack HTTP Body          | Rendering....                |                                                   |
| Rendering....               | Wait                         | Waits for final rendering process                | Merge - Creatomate           | done?                       |                                                   |
| done?                      | HTTP Request                 | Polls final rendering status                      | Rendering....               | Switch2                     |                                                   |
| Switch2                    | Switch                       | Branches based on rendering status                | done?                       | Download final video, Shotstack HTTP Body, Rendering.... |                                                   |
| Download final video        | HTTP Request                 | Downloads fully rendered video                     | Switch2                     | Upload video to Postiz       |                                                   |
| Upload video to Postiz      | HTTP Request                 | Uploads final video to Postiz platform             | Download final video         | Get Postiz integrations      |                                                   |
| Get Postiz integrations     | HTTP Request                 | Fetches Postiz social media integrations           | Upload video to Postiz       | Merge1                      |                                                   |
| Merge1                     | Merge                         | Combines social content and integration data       | Get Postiz integrations, Write content for social media | Switch3                    |                                                   |
| Write content for social media | Chain LLM                   | Generates social media post content                | Planning the ad, OpenAI Chat Model3, Structured Output Parser2 | Merge1                      |                                                   |
| OpenAI Chat Model3          | LLM OpenAI                   | AI text generation for social content              | Write content for social media | Write content for social media |                                                   |
| Structured Output Parser2   | Output Parser                | Parses AI social content output                      | OpenAI Chat Model4           | Write content for social media |                                                   |
| OpenAI Chat Model4          | LLM OpenAI                   | Supports social media content parsing               | Structured Output Parser2    | Write content for social media |                                                   |
| Switch3                    | Switch                       | Routes scheduling to correct social media platform  | Merge1                      | Schedule TikTok, Schedule YouTube, Schedule Instagram, Schedule Facebook |                                                   |
| Schedule TikTok             | HTTP Request                 | Schedules video post on TikTok                       | Switch3                     | None                        |                                                   |
| Schedule YouTube            | HTTP Request                 | Schedules video post on YouTube                      | Switch3                     | None                        |                                                   |
| Schedule Instagram          | HTTP Request                 | Schedules video post on Instagram                    | Switch3                     | None                        |                                                   |
| Schedule Facebook           | HTTP Request                 | Schedules video post on Facebook                     | Switch3                     | None                        |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: "When clicking ‘Execute workflow’"
   - Type: Manual Trigger
   - No special parameters

2. **Add a Set Node**
   - Name: "Form"
   - Purpose: Initialize variables or inputs (empty parameters)

3. **Add a Set Node**
   - Name: "Set Credintenails"
   - Purpose: Set API credentials and environment variables for OpenAI, Google Gemini, video APIs.

4. **Connect:** Manual Trigger → Form → Set Credintenails

5. **Add AI Language Model Nodes for Planning**
   - Add OpenAI Chat Model (name: "OpenAI Chat Model")
   - Add Structured Output Parser (name: "Structured Output Parser")
   - Add Chain LLM Node (name: "Planning the ad")
     - Connect OpenAI Chat Model as language model input
     - Connect Structured Output Parser as output parser input
     - Connect Set Credintenails as main input
   - Connect Set Credintenails → Planning the ad

6. **Add Merge Node**
   - Name: "Merge"
   - Will merge AI planning output and video analysis results later.

7. **Add Chain LLM for Analyze part 1**
   - Add Google Gemini Chat Model3
   - Add Structured Output Parser1
   - Add chain LLM node "Analyze part 1"
   - Connect Gemini Chat Model3 and Structured Output Parser1 as AI inputs
   - Connect Merge node as main input
   - Connect Analyze part 1 → Part_2 (next chain LLM)

8. **Add Chain LLM Nodes for Part_1 and Part_2**
   - Part_1 connects to JWT node
   - Part_2 connects to Generate Video1

9. **Add JWT Node**
   - Configure with secret key for video API
   - Connect Part_1 → JWT → GET TOKEN

10. **Add HTTP Request Node GET TOKEN**
    - POST to obtain access token
    - Connect JWT → GET TOKEN → Generate Video

11. **Add Generate Video HTTP Request Node**
    - POST to video generation API using token
    - Connect GET TOKEN → Generate Video → Wait

12. **Add Wait Node**
    - Use webhook ID to wait for async video generation
    - Connect Generate Video → Wait → Fetch Status

13. **Add Fetch Status HTTP Request Node**
    - Poll video generation status endpoint
    - Connect Wait → Fetch Status → Switch

14. **Add Switch Node**
    - Configure branches for different statuses (e.g., completed, pending)
    - Connect Fetch Status → Switch
    - Branch to Convert to File, Part_1, or Wait accordingly

15. **Add Convert to File Node**
    - Convert video to 9:16 aspect ratio
    - Connect Switch → Convert to File → Post video Cloudinary Part1

16. **Add Post video Cloudinary Part1 HTTP Request**
    - Upload video part 1 to Cloudinary
    - Connect Convert to File → Post video Cloudinary Part1 → Begin Gemini Upload Session

17. **Add Nodes for Gemini Upload and Analysis**
    - Begin Gemini Upload Session (HTTP POST)
    - Download binary video (HTTP GET)
    - Upload video to Gemini (HTTP POST)
    - Waiting... (Wait with webhook)
    - Analyze Video with Gemini (HTTP GET with retry)
    - Connect nodes sequentially as per flow

18. **Add Merge Node to combine analysis**
    - Connect Analyze Video with Gemini and Planning the ad to Merge

19. **Add Analyze part 1 Chain LLM and connect to Part_2**

20. **Add Generate Video1 HTTP Request**
    - Connect Part_2 → Generate Video1 → Wait1 → Fetch Status1 → Switch1

21. **Repeat Steps 14-16 for second video part with Switch1, Convert to File1, Post video Cloudinary Part 2**

22. **Add Shotstack HTTP Body Code Node**
    - Prepare final render request JSON
    - Connect Post video Cloudinary Part 2 → Shotstack HTTP Body → Merge - Creatomate

23. **Add Merge - Creatomate HTTP Request**
    - POST to Creatomate rendering API
    - Connect Shotstack HTTP Body → Merge - Creatomate → Rendering....

24. **Add Rendering.... Wait Node**
    - Wait for async rendering completion webhook
    - Connect Merge - Creatomate → Rendering.... → done?

25. **Add done? HTTP Request**
    - Poll rendering status endpoint
    - Connect Rendering.... → done? → Switch2

26. **Add Switch2 Node**
    - Branch logic: If done download final video, else retry or wait
    - Connect done? → Switch2 → Download final video (if done), Shotstack HTTP Body (retry), or Rendering.... (wait)

27. **Add Download final video HTTP Request**
    - Connect Switch2 → Download final video → Upload video to Postiz

28. **Add Upload video to Postiz HTTP Request**
    - Connect Download final video → Upload video to Postiz → Get Postiz integrations

29. **Add Get Postiz integrations HTTP Request**
    - Connect Upload video to Postiz → Get Postiz integrations → Merge1

30. **Add Write content for social media Chain LLM**
    - Connect Planning the ad, OpenAI Chat Model3, Structured Output Parser2 to this node
    - Connect Write content for social media → Merge1

31. **Add Merge1 Node**
    - Combine social content and integrations
    - Connect Get Postiz integrations, Write content for social media → Merge1 → Switch3

32. **Add Switch3 Node**
    - Branch scheduling to social media platforms
    - Connect Merge1 → Switch3 → Schedule TikTok, Schedule YouTube, Schedule Instagram, Schedule Facebook

33. **Add Schedule TikTok, YouTube, Instagram, and Facebook HTTP Request Nodes**
    - Configure each for respective platform API calls with scheduling parameters

34. **Add all AI models (OpenAI and Google Gemini) with proper credential configuration**

35. **Configure all HTTP Request nodes with appropriate URLs, headers, and authentication**

36. **Set webhook IDs and ensure webhooks are registered for Wait nodes**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| At the "Convert to File" and "Convert to File1" nodes, the video must be generated and ready to convert to 9:16 aspect ratio. | Video conversion step                              |
| The workflow uses Google Gemini and OpenAI language models interchangeably for different AI tasks.               | AI integration                                    |
| The workflow relies on asynchronous webhook-based waiting for video rendering and processing completion.         | Webhook IDs in Wait nodes                          |
| Creatomate API is used for final video rendering and merging of video parts.                                     | Video rendering service                            |
| Postiz platform handles the final video upload and social media scheduling integrations.                         | Social media publishing                            |
| Google Gemini video analysis nodes include retry logic with a 15-second wait between tries.                      | Robust video analysis                              |
| The workflow contains multiple switch nodes to branch logic based on API status responses or AI outputs.         | Conditional branching                              |
| This workflow is designed for professional digital marketing teams automating video ad creation and publishing. | Target use case                                   |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.