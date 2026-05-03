Generate & Publish AI Cinematic Videos to YouTube Shorts using VEO3 & Gemini

https://n8nworkflows.xyz/workflows/generate---publish-ai-cinematic-videos-to-youtube-shorts-using-veo3---gemini-8004


# Generate & Publish AI Cinematic Videos to YouTube Shorts using VEO3 & Gemini

---

### 1. Workflow Overview

This workflow automates the generation and publishing of AI-created cinematic videos tailored for YouTube Shorts and Telegram channels. It is designed to run twice daily, leveraging advanced AI models to conceptualize surreal, hyperrealistic video ideas, generate the videos using an external API, and then publish them with enriched metadata.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Scheduling:** Automatically initiates the workflow twice per day based on cron schedules.
- **1.2 AI Prompt Generation:** Uses AI (Google Gemini via LangChain agent) to generate a detailed video prompt, title, caption, and description tailored for cinematic surreal videos.
- **1.3 Video Generation API:** Sends the AI-generated prompt to the KIE AI VEO API to create a vertical (9:16) cinematic video.
- **1.4 Render Monitoring:** Waits and checks the rendering status of the video until completion.
- **1.5 Video Publishing:** Uploads the finished video to YouTube Shorts and shares it on Telegram with AI-generated metadata.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Scheduling

**Overview:**  
This block triggers the entire workflow automatically twice a day using cron expressions.

**Nodes Involved:**  
- Sticky Note - Trigger  
- Schedule Trigger

**Node Details:**

- **Sticky Note - Trigger**  
  - Type: Sticky Note  
  - Role: Documentation for the trigger setup  
  - Content: Explains the workflow runs twice daily and can be adjusted.  
  - Connections: None  
  - Edge cases: None (informational only)

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically starts the workflow based on specified times  
  - Configuration: Runs at 02:12 and 17:12 daily (cron expressions: "0 12 02 * * *" and "0 12 17 * * *")  
  - Input: None  
  - Output: Triggers the AI Agent node  
  - Edge cases: Cron misconfiguration; workflow not triggered if n8n scheduler fails

---

#### 1.2 AI Prompt Generation

**Overview:**  
Generates a detailed video prompt, short caption with hashtags, YouTube Short title, and description using an AI agent. This content is precisely tailored for surreal, cinematic short videos.

**Nodes Involved:**  
- Sticky Note - AI Prompt  
- AI Agent  
- Google Gemini Chat Model (as AI backend)  
- Structured Output Parser

**Node Details:**

- **Sticky Note - AI Prompt**  
  - Type: Sticky Note  
  - Role: Documentation for AI prompt generation expectations  
  - Content: Specifies the four outputs AI must produce (detailed prompt, caption, title, description)  
  - Connections: None  
  - Edge cases: None (informational)

- **AI Agent**  
  - Type: LangChain Agent node  
  - Role: Generates AI outputs based on a system prompt and user instructions  
  - Configuration:  
    - Prompt instructs to generate:  
      1. Highly detailed video prompt (900–1800 chars)  
      2. Short hypnotic caption with hashtags (max 12 words)  
      3. YouTube Short title (max 100 chars)  
      4. Humorous, keyword-rich description (up to 2000 chars)  
    - System message defines style, scene characteristics, and rules (e.g., no humans, surreal visuals, 8-second length, 4K quality)  
    - Uses Google Gemini Chat Model as AI backend  
    - Output parser enabled to structure AI outputs into JSON fields: prompt, caption, title, description  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: JSON with AI-generated content, passed to Generate Video node  
  - Edge cases: AI model errors, output parsing failures, API rate limits, malformed AI output

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Language Model node  
  - Role: Provides LLM processing for the AI Agent  
  - Configuration: Uses Google Palm API credentials  
  - Inputs: AI Agent as consumer  
  - Outputs: AI-generated text for AI Agent node processing  
  - Edge cases: API authentication errors, quota limits, network timeouts

- **Structured Output Parser**  
  - Type: LangChain structured output parser  
  - Role: Parses AI raw text output into structured JSON matching schema (prompt, caption, title, description)  
  - Configuration: JSON schema example provided for output validation  
  - Inputs: AI Agent's AI output parser input  
  - Outputs: Parsed structured data for downstream nodes  
  - Edge cases: Parsing errors if AI output deviates from schema, invalid JSON, missing fields

---

#### 1.3 Video Generation API

**Overview:**  
Sends the AI-generated video prompt to the KIE AI VEO API to generate a cinematic video file with specific parameters.

**Nodes Involved:**  
- Sticky Note - Video API  
- Generate Video

**Node Details:**

- **Sticky Note - Video API**  
  - Type: Sticky Note  
  - Role: Documentation on video generation API usage  
  - Content: Explains API model used (`veo3_fast`), aspect ratio (9:16), and synchronous waiting for completion  
  - Connections: None  
  - Edge cases: None (informational)

- **Generate Video**  
  - Type: HTTP Request  
  - Role: POST request to KIE AI VEO API to start video generation  
  - Configuration:  
    - URL: https://api.kie.ai/api/v1/veo/generate  
    - Method: POST  
    - Headers: Authorization Bearer token (user must replace YOUR_API_KEY), Content-Type application/json  
    - Body (JSON):  
      - `prompt`: AI Agent's structured prompt, sanitized (removes quotes and line breaks)  
      - `model`: "veo3_fast"  
      - `aspectRatio`: "9:16" (vertical video)  
      - `seeds`: 12345 (fixed seed for reproducibility)  
      - `enableFallback`: false  
  - Inputs: AI Agent output JSON  
  - Outputs: API response containing a `taskId` for tracking render status  
  - Edge cases: API key invalid or missing, network errors, API rate limits, malformed prompt causing generation failure

---

#### 1.4 Render Monitoring

**Overview:**  
Waits for a fixed period to allow the video rendering to complete, then queries the API for render status using the task ID.

**Nodes Involved:**  
- Wait for Render  
- Check Render Status

**Node Details:**

- **Wait for Render**  
  - Type: Wait node  
  - Role: Pauses workflow for 60 seconds to allow video rendering  
  - Configuration: Wait 60 seconds  
  - Inputs: Output from Generate Video  
  - Outputs: Passes to Check Render Status  
  - Edge cases: Fixed wait may be insufficient for longer renders; no dynamic polling implemented

- **Check Render Status**  
  - Type: HTTP Request  
  - Role: Queries KIE AI VEO API for video render status and metadata  
  - Configuration:  
    - URL: https://api.kie.ai/api/v1/veo/record-info  
    - Method: GET  
    - Query Parameter: taskId from previous node response  
    - Headers: Authorization Bearer token (same as Generate Video)  
  - Inputs: Wait for Render output containing taskId  
  - Outputs: Returns video file URL and metadata for publishing  
  - Edge cases: API errors, taskId invalid or expired, video not ready yet (no retry logic shown)

---

#### 1.5 Video Publishing

**Overview:**  
Uploads the finished video to YouTube Shorts with AI-generated metadata, and sends the video to a Telegram channel with the AI caption.

**Nodes Involved:**  
- Sticky Note - Publishing  
- Upload to YouTube  
- Send to Telegram

**Node Details:**

- **Sticky Note - Publishing**  
  - Type: Sticky Note  
  - Role: Documentation describing the publishing process  
  - Content: Explains automatic upload to YouTube Shorts and Telegram with AI-generated metadata  
  - Connections: None  
  - Edge cases: None (informational)

- **Upload to YouTube**  
  - Type: YouTube node (video upload operation)  
  - Role: Uploads the generated video to YouTube Shorts  
  - Configuration:  
    - Title: AI Agent's generated title, truncated to 99 characters  
    - Description: AI Agent's generated description  
    - Tags: Empty by default  
    - Privacy: Public  
    - Category ID: 1 (Film & Animation)  
    - Region Code: DE (Germany)  
    - Other options: Embeddable, notify subscribers, public stats enabled, not made for kids  
  - Credentials: OAuth2 YouTube API (user-specific)  
  - Inputs: Check Render Status output (video file info)  
  - Outputs: Video upload confirmation  
  - Edge cases: OAuth token expiration, YouTube API quota limits, upload failures, video format incompatibility

- **Send to Telegram**  
  - Type: Telegram node (sendVideo operation)  
  - Role: Sends the generated video file to a Telegram channel with AI-generated caption  
  - Configuration:  
    - Chat ID: "-1001234567890" (Telegram channel/group)  
    - Caption: AI Agent's generated caption  
    - Binary Data: true (sends video as file)  
  - Credentials: Telegram Bot API token  
  - Inputs: Check Render Status output (video file)  
  - Outputs: Message sent confirmation  
  - Edge cases: Invalid chat ID, bot permissions issues, video size limits, network failures

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                          | Input Node(s)             | Output Node(s)                       | Sticky Note                                                                                                   |
|-------------------------|--------------------------------|----------------------------------------|---------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note - Trigger    | Sticky Note                    | Documentation for workflow trigger     | None                      | None                               | ## Start Trigger\nThis workflow is scheduled to run automatically twice a day. You can adjust the Schedule Trigger to your preferred timing. |
| Schedule Trigger        | Schedule Trigger               | Starts workflow twice daily             | None                      | AI Agent                          |                                                                                                              |
| Sticky Note - AI Prompt  | Sticky Note                    | Documentation for AI prompt generation | None                      | None                               | ## AI Prompt Generation\nThe AI Agent creates:\n1. A detailed video prompt (900–1800 characters).\n2. A short hypnotic caption with hashtags.\n3. A YouTube Short title (max 100 characters).\n4. A description (up to 2000 characters). |
| AI Agent                | LangChain Agent                | Generates AI content for video          | Schedule Trigger          | Generate Video                    |                                                                                                              |
| Google Gemini Chat Model | LangChain Google Gemini Model | AI language model backend                | AI Agent (languageModel)  | AI Agent                           |                                                                                                              |
| Structured Output Parser | LangChain Output Parser        | Parses AI raw output into structured JSON | AI Agent (outputParser)    | AI Agent                          |                                                                                                              |
| Sticky Note - Video API  | Sticky Note                    | Documentation on Video Generation API   | None                      | None                               | ## Video Generation API\nThe prompt is sent to the KIE AI VEO API which generates the video file.\n- Uses `veo3_fast` model\n- Aspect ratio: 9:16 (vertical video)\n- Waits for API to complete before fetching results |
| Generate Video          | HTTP Request                  | Sends prompt to video generation API    | AI Agent                  | Wait for Render                   |                                                                                                              |
| Wait for Render         | Wait                         | Waits for video rendering to complete   | Generate Video            | Check Render Status               |                                                                                                              |
| Check Render Status     | HTTP Request                  | Checks video render status               | Wait for Render           | Upload to YouTube, Send to Telegram |                                                                                                              |
| Sticky Note - Publishing | Sticky Note                    | Documentation on video publishing        | None                      | None                               | ## Video Publishing\nThe generated video is uploaded automatically:\n- To YouTube Shorts with AI-generated title, description & tags\n- To Telegram channel with caption |
| Upload to YouTube       | YouTube                       | Uploads video to YouTube Shorts          | Check Render Status       | None                             |                                                                                                              |
| Send to Telegram        | Telegram                      | Sends video to Telegram channel          | Check Render Status       | None                             |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set two cron expressions:  
     - "0 12 02 * * *" (2:12 AM daily)  
     - "0 12 17 * * *" (5:12 PM daily)  
   - Position it as the workflow start node.

2. **Add the AI Agent node:**  
   - Type: LangChain Agent  
   - Connect Schedule Trigger output to AI Agent input.  
   - Configure the prompt: instruct it to generate:  
     - A detailed video prompt (900–1800 characters)  
     - A short hypnotic caption with hashtags (max 12 words)  
     - A YouTube Short title (max 100 characters)  
     - A humorous, keyword-rich description (up to 2000 characters)  
   - Set system message to define style and scene rules, including:  
     - 8-second surreal cinematic video concepts  
     - 4K ultra HD, warm tones, impossible visuals, no humans or text  
   - Enable output parser.  
   - Connect Google Gemini Chat Model to AI Agent as language model backend.

3. **Set up the Google Gemini Chat Model node:**  
   - Type: LangChain Google Gemini Model  
   - Provide Google Palm API credentials.

4. **Add Structured Output Parser node:**  
   - Type: LangChain Output Parser Structured  
   - Define JSON schema with fields: prompt, caption, title, description  
   - Connect AI Agent's AI output parser to this node.

5. **Create Generate Video HTTP Request node:**  
   - URL: https://api.kie.ai/api/v1/veo/generate  
   - Method: POST  
   - Headers:  
     - Authorization: Bearer YOUR_API_KEY (replace YOUR_API_KEY with actual key)  
     - Content-Type: application/json  
   - Body (JSON):  
     ```json
     {
       "prompt": "{{ $json.output.prompt.replaceAll('\"', '').replaceAll('\\n', '') }}",
       "model": "veo3_fast",
       "aspectRatio": "9:16",
       "seeds": 12345,
       "enableFallback": false
     }
     ```  
   - Connect AI Agent’s output to this node input.

6. **Add Wait node:**  
   - Configure wait time: 60 seconds  
   - Connect Generate Video output to Wait input.

7. **Add Check Render Status HTTP Request node:**  
   - URL: https://api.kie.ai/api/v1/veo/record-info  
   - Method: GET  
   - Query parameter: taskId = `={{ $json.data.taskId }}` from previous node output  
   - Headers: Authorization Bearer token same as Generate Video  
   - Connect Wait output to this node input.

8. **Create Upload to YouTube node:**  
   - Type: YouTube node (resource: video, operation: upload)  
   - Configure parameters:  
     - Title: `={{ $('AI Agent').item.json.output.title.substring(0,99) }}`  
     - Description: `={{ $('AI Agent').item.json.output.describtion }}` (note spelling from workflow)  
     - Tags: leave empty or add as desired  
     - Privacy status: public  
     - Category ID: 1 (Film & Animation)  
     - Region Code: DE  
     - Options: embeddable, notify subscribers, public stats enabled, not made for kids  
   - Connect Check Render Status output to this node input.  
   - Set up YouTube OAuth2 credentials.

9. **Add Send to Telegram node:**  
   - Type: Telegram node (operation: sendVideo)  
   - Chat ID: "-1001234567890" (replace with your channel/group ID)  
   - Enable binary data sending.  
   - Caption: `={{ $('AI Agent').item.json.output.caption }}`  
   - Connect Check Render Status output to this node input.  
   - Configure Telegram API credentials (bot token).

10. **Add Sticky Notes as documentation:**  
    - Place sticky notes near respective blocks for clarity:  
      - Trigger information near Schedule Trigger  
      - AI prompt details near AI Agent  
      - Video API notes near Generate Video  
      - Publishing information near Upload to YouTube and Send to Telegram

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The AI prompt generation uses Google Gemini (PaLM) API via LangChain for advanced language understanding.            | Google Gemini (PaLM) API credentials setup                                                        |
| Video generation relies on KIE AI’s VEO API with `veo3_fast` model for vertical cinematic videos.                    | https://api.kie.ai/api/v1/veo/generate                                                           |
| The workflow targets YouTube Shorts (vertical 9:16 videos) and Telegram channels for video distribution.             | YouTube Shorts best practices: https://support.google.com/youtube/answer/10803921?hl=en           |
| AI-generated metadata (title, caption, description) is carefully crafted to maximize engagement and SEO.             | Custom prompt guidelines in AI Agent system message                                               |
| The workflow assumes proper API keys and OAuth2 credentials are configured for Google Gemini, KIE AI, YouTube, Telegram. |                                                                                                   |
| Fixed 60-second wait might require adjustment based on actual video render times; consider implementing retry logic. |                                                                                                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all current content policies. No illegal, offensive, or protected content is included. All data handled is legal and public.

---