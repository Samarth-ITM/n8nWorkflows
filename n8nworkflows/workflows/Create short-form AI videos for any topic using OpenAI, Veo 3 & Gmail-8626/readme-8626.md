Create short-form AI videos for any topic using OpenAI, Veo 3 & Gmail

https://n8nworkflows.xyz/workflows/create-short-form-ai-videos-for-any-topic-using-openai--veo-3---gmail-8626


# Create short-form AI videos for any topic using OpenAI, Veo 3 & Gmail

### 1. Workflow Overview

This n8n workflow automates the creation of short-form AI-generated financial news videos, including script generation, video creation, description production for social media, and distribution via email. It targets content creators, marketers, and financial news outlets aiming to produce daily engaging video snippets for social media platforms efficiently.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception & Trigger:** Automatically triggers the workflow daily.
- **1.2 Content Generation:** Generates an 8-second financial news script, a video prompt for Veo 3 AI video generation, and a social media description using OpenAI's GPT-4.1-mini model.
- **1.3 Video Production & Monitoring:** Sends the video generation prompt to Veo 3, waits for the video to be generated, polls the status, and loops until the video is ready.
- **1.4 Video Download & Distribution:** Downloads the generated video and emails it with the social media description attached.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Trigger

- **Overview:** The workflow starts automatically every day via a scheduled trigger node.
- **Nodes Involved:** `Daily Trigger`
- **Node Details:**

  - **Daily Trigger**
    - Type: Schedule Trigger
    - Role: Initiates the workflow once every day without manual intervention.
    - Configuration: Uses a default interval schedule triggering daily at midnight (default empty interval).
    - Inputs: None (start node).
    - Outputs: Triggers `Generate News Script` node.
    - Edge Cases: Possible delay if server time is off; ensure correct timezone configured in n8n instance.
    - No credential required.
    - Sticky Note: "Step 1: Trigger Workflow - Starts the workflow automatically every day."

#### 2.2 Content Generation

- **Overview:** Generates the financial news script, converts it into a video prompt for Veo 3, and produces a social media description suitable for multiple platforms.
- **Nodes Involved:** `Generate News Script`, `Generate Veo3 Prompt`, `Social Media Description`
- **Node Details:**

  - **Generate News Script**
    - Type: OpenAI (Langchain) Node
    - Role: Creates a concise 8-second spoken-style financial news script focusing on recent global market events.
    - Configuration: Uses GPT-4.1-mini model with a prompt instructing to generate 3-4 short sentences, urgent and informative tone.
    - Inputs: Triggered by `Daily Trigger`.
    - Outputs: Passes script content to `Generate Veo3 Prompt` and `Social Media Description`.
    - Credentials: OpenAI API key.
    - Edge Cases: API rate limits, prompt failure, or empty response.
  
  - **Generate Veo3 Prompt**
    - Type: OpenAI (Langchain) Node
    - Role: Transforms the financial script into a detailed video prompt describing the visuals, mood, and setting for Veo 3 video generation.
    - Configuration: Uses GPT-4.1-mini model; prompt excludes direct script text, focusing on cinematic video scene description.
    - Inputs: Receives the script content from `Generate News Script`.
    - Outputs: Provides video prompt to `Create Video`.
    - Credentials: OpenAI API key.
    - Edge Cases: Misinterpretation of script content, API response errors.
  
  - **Social Media Description**
    - Type: OpenAI (Langchain) Node
    - Role: Generates an engaging, SEO-friendly social media post description for LinkedIn, Instagram, and YouTube.
    - Configuration: GPT-4.1-mini, prompt instructs to summarize script content with hashtags and professional tone within 50-150 words.
    - Inputs: Receives script content from `Generate News Script`.
    - Outputs: Sends description to `Send a message` node.
    - Credentials: OpenAI API key.
    - Edge Cases: Text generation failure, inappropriate content generation (mitigated by prompt design).

- Sticky Note: "Step 2: Generate Content - Creates 8-sec financial news script, video prompt for Veo3, and social media description."

#### 2.3 Video Production & Monitoring

- **Overview:** Sends the video prompt to Veo 3 API for generation, waits 30 seconds, checks generation status via Google's generative language API, and loops until video is ready.
- **Nodes Involved:** `Create Video`, `Wait for Video`, `Status`, `If`
- **Node Details:**

  - **Create Video**
    - Type: HTTP Request
    - Role: Calls Veo 3 API endpoint to start video generation with the prompt.
    - Configuration: POST request to Veo 3 model endpoint with JSON body containing the prompt and aspect ratio 16:9; API key passed as query parameter.
    - Inputs: Receives video prompt from `Generate Veo3 Prompt`.
    - Outputs: Passes response to `Wait for Video`.
    - Edge Cases: API timeout, authentication errors, malformed prompt causing generation failure.
  
  - **Wait for Video**
    - Type: Wait Node
    - Role: Pauses workflow for 30 seconds before checking video generation status.
    - Configuration: Fixed wait duration of 30 seconds.
    - Inputs: Triggered after `Create Video` or `If` node when video is not ready.
    - Outputs: Calls `Status` node.
    - Edge Cases: Waiting too short or too long may affect workflow efficiency.
  
  - **Status**
    - Type: HTTP Request
    - Role: Queries Google generative language API for the current status of video generation.
    - Configuration: GET request to status endpoint, with model name dynamically inserted; API key passed as query parameter.
    - Inputs: Triggered after wait period.
    - Outputs: Passes status response to `If`.
    - Edge Cases: API quota limits, network errors, incorrect endpoint usage.
  
  - **If**
    - Type: If Condition
    - Role: Checks if the video generation is done (`done` flag in JSON response).
    - Configuration: Condition tests if `$json.done` equals `true`.
    - Inputs: Receives status from `Status` node.
    - Outputs: If true, proceeds to `Download Video`; if false, loops back to `Wait for Video` to poll again.
    - Edge Cases: Failure if `done` field missing or improperly formatted.

- Sticky Note: "Step 3: Generate Video - Generates the video, waits for it to finish, checks status, and loops if not ready."

#### 2.4 Video Download & Distribution

- **Overview:** Downloads the fully generated video and sends it via Gmail with the social media description included in the email body.
- **Nodes Involved:** `Download Video`, `Send a message`
- **Node Details:**

  - **Download Video**
    - Type: HTTP Request
    - Role: Downloads the generated video file from the URI provided in the generation response.
    - Configuration: GET request to the video URI extracted from the previous response; API key passed as query parameter.
    - Inputs: Triggered after `If` node confirms video is ready.
    - Outputs: Passes binary video data to `Send a message`.
    - Edge Cases: Download failures, broken or expired video URLs.
  
  - **Send a message**
    - Type: Gmail Node
    - Role: Sends an email containing the social media description as message content and attaches the video file.
    - Configuration:
      - Recipient email set by user (`YOUR_EMAIL_HERE` placeholder).
      - Subject: "The video is ready to share on social media."
      - Message body includes the social media description.
      - Attaches the downloaded video binary.
    - Inputs: Receives social media description from `Social Media Description` (parallel path) and video binary from `Download Video`.
    - Credentials: Gmail OAuth2.
    - Edge Cases: Email sending failures, attachment size limits, OAuth token expiration.

- Sticky Note: "Step 4: Download and Share Video - Downloads the generated video and sends an email with description and attachment."

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                                 | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                  |
|----------------------|----------------------------------|------------------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Daily Trigger        | Schedule Trigger                 | Starts workflow daily                           | None                   | Generate News Script     | Step 1: Trigger Workflow - Starts the workflow automatically every day.                      |
| Generate News Script | OpenAI (Langchain)               | Generates 8-sec financial news script          | Daily Trigger          | Generate Veo3 Prompt, Social Media Description | Step 2: Generate Content - Creates 8-sec script, video prompt, and social media description. |
| Generate Veo3 Prompt | OpenAI (Langchain)               | Creates video generation prompt for Veo 3      | Generate News Script   | Create Video            | Step 2: Generate Content - Creates 8-sec script, video prompt, and social media description. |
| Social Media Description | OpenAI (Langchain)            | Generates SEO-friendly social media description| Generate News Script   | Send a message          | Step 2: Generate Content - Creates 8-sec script, video prompt, and social media description. |
| Create Video         | HTTP Request                    | Sends video prompt to Veo 3 API                 | Generate Veo3 Prompt   | Wait for Video          | Step 3: Generate Video - Generates video, waits, checks status, loops if not ready.          |
| Wait for Video       | Wait                           | Pauses workflow 30 sec before status check     | Create Video, If       | Status                  | Step 3: Generate Video - Generates video, waits, checks status, loops if not ready.          |
| Status               | HTTP Request                   | Checks video generation status                   | Wait for Video         | If                      | Step 3: Generate Video - Generates video, waits, checks status, loops if not ready.          |
| If                   | If Condition                   | Determines if video generation is complete      | Status                 | Download Video (if true), Wait for Video (if false) | Step 3: Generate Video - Generates video, waits, checks status, loops if not ready.          |
| Download Video       | HTTP Request                   | Downloads the generated video file               | If                     | Send a message          | Step 4: Download and Share Video - Downloads video and sends email with attachment.          |
| Send a message       | Gmail                          | Sends email with social media description & video attachment | Download Video, Social Media Description | None                    | Step 4: Download and Share Video - Downloads video and sends email with attachment.          |
| Status               | HTTP Request                   | Queries video generation status                   | Wait for Video         | If                      |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add a **Schedule Trigger** node.
   - Set the schedule to trigger daily (default daily interval).
   - Name it `Daily Trigger`.

2. **Create Content Generation Nodes:**
   - Add an **OpenAI (Langchain)** node named `Generate News Script`.
     - Model: `gpt-4.1-mini`.
     - Prompt: Instruct to generate a concise 8-second spoken-style financial news script on trending stock market news, 3-4 short sentences, urgent and informative tone.
     - Connect input from `Daily Trigger`.

   - Add an **OpenAI (Langchain)** node named `Generate Veo3 Prompt`.
     - Model: `gpt-4.1-mini`.
     - Prompt: Convert the news script to a cinematic video prompt describing visuals for Veo 3.
     - Connect input from `Generate News Script`.

   - Add an **OpenAI (Langchain)** node named `Social Media Description`.
     - Model: `gpt-4.1-mini`.
     - Prompt: Create an engaging, SEO-friendly social media description summarizing the script, including hashtags, 50-150 words.
     - Connect input from `Generate News Script`.

3. **Create Video Request & Polling Nodes:**
   - Add an **HTTP Request** node named `Create Video`.
     - Method: POST.
     - URL: `https://generativelanguage.googleapis.com/v1beta/models/veo-3.0-fast-generate-preview:predictLongRunning`.
     - Body (JSON): Send `{ "instances": [ { "prompt": <video prompt> }], "parameters": { "aspectRatio": "16:9" } }`.
     - Query Parameter: `key` for API key.
     - Connect input from `Generate Veo3 Prompt`.

   - Add a **Wait** node named `Wait for Video`.
     - Duration: 30 seconds.
     - Connect input from `Create Video`.

   - Add an **HTTP Request** node named `Status`.
     - Method: GET.
     - URL: `https://generativelanguage.googleapis.com/v1beta/{{ $json.name }}` (dynamic).
     - Query Parameter: `key` for API key.
     - Connect input from `Wait for Video`.

   - Add an **If** node.
     - Condition: Check if `$json.done` equals `true`.
     - Connect input from `Status`.
     - If true: Connect to `Download Video`.
     - If false: Loop back to `Wait for Video`.

4. **Download Video and Send Email:**
   - Add an **HTTP Request** node named `Download Video`.
     - Method: GET.
     - URL: Use video URI from previous response `{{$json.response.generateVideoResponse.generatedSamples[0].video.uri}}`.
     - Query Parameter: `key` for API key.
     - Connect input from `If` (true branch).

   - Add a **Gmail** node named `Send a message`.
     - Credentials: Gmail OAuth2 credentials.
     - To: Your email address.
     - Subject: "The video is ready to share on social media."
     - Message: Use social media description content (from `Social Media Description` node).
     - Attachments: Add binary video data from `Download Video`.
     - Connect input from both `Download Video` and `Social Media Description` (merge data as needed).

5. **Credentials Setup:**
   - Configure OpenAI API credentials with valid API key.
   - Configure Gmail OAuth2 credentials with authorized Gmail account.
   - Set Google Generative Language API key in HTTP Request nodes as query parameter `key`.

6. **Final Connections & Testing:**
   - Ensure the data flows correctly between nodes, especially that the script content propagates to dependent nodes.
   - Test the workflow manually for a single run before scheduling.
   - Monitor for API limits or errors, adjust wait times if video generation takes longer.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses the GPT-4.1-mini model for reliable, concise text generation tailored for financial news and video prompt creation. | OpenAI GPT-4.1-mini model specifications and best practices.                                        |
| Veo 3 is used as the AI video generation engine, requiring specific prompt formatting to guide visuals. | Veo 3 API documentation for video prompt structure and generation parameters.                       |
| Gmail OAuth2 credentials must be properly authorized with the "Send Email" scope to avoid failures. | Gmail API OAuth2 setup instructions.                                                               |
| The workflow includes a polling loop with a 30-second wait to handle asynchronous video generation. | Adjust polling interval based on average generation time for efficiency.                            |
| API keys and sensitive credentials should be stored securely in n8n credentials manager.             | n8n documentation: Credential management best practices.                                           |
| For detailed prompt writing, ensure to keep messages concise and tone-appropriate to avoid off-topic generation. | Prompt engineering guidelines for OpenAI models.                                                   |
| The workflow can be extended to support multiple languages or topics by adjusting prompt contents.  | Multi-language content creation considerations.                                                    |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and publicly available.