Generating AI Videos with VEO3 and Distributing with Blotato across Multiple Platforms

https://n8nworkflows.xyz/workflows/generating-ai-videos-with-veo3-and-distributing-with-blotato-across-multiple-platforms-6669


# Generating AI Videos with VEO3 and Distributing with Blotato across Multiple Platforms

---

### 1. Workflow Overview

This workflow automates the entire process of creating and distributing AI-generated viral videos using VEO3 for video generation and Blotato for multi-platform social media distribution. It starts by receiving a video idea via Telegram, then generates a structured video ad script with OpenAI GPT-4, builds the video through the VEO3 API, processes and stores metadata in Google Sheets, and finally auto-posts the video with captions across nine social media platforms via Blotato.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Script Generation:** Handles receiving video ideas from Telegram, generates a master prompt using a predefined schema, and creates a structured video script using an AI agent with OpenAI GPT-4.

- **1.2 Video Creation and Caption Processing:** Sends the AI-generated prompt to VEO3 to generate the video, waits for rendering, downloads the video, rewrites captions with GPT-4o, and saves metadata including video URL and captions to Google Sheets.

- **1.3 Multi-Platform Video Upload and Distribution:** Assigns social media account IDs, uploads the video to Blotato, and posts it automatically to nine social media platforms (Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Bluesky, Pinterest) via Blotato’s API. Includes notifications via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Script Generation

**Overview:**  
This block captures the video idea input from a Telegram user, retrieves video parameters from Google Sheets, builds a master prompt schema, and calls an AI Agent to produce a structured video ad script in JSON format.

**Nodes Involved:**  
- Telegram Trigger: Receive Video Idea  
- Read Video Parameters from Google Sheet  
- Set Master Prompt  
- OpenAI Chat Model  
- Think (LangChain tool)  
- Structured Output Parser  
- AI Agent: Generate Video Script

**Node Details:**

- **Telegram Trigger: Receive Video Idea**  
  - Type: Telegram Trigger  
  - Role: Entry point receiving video idea text messages from Telegram users.  
  - Config: Listens to "message" updates. Uses Telegram API credentials.  
  - Inputs: External Telegram messages.  
  - Outputs: Passes message content downstream.  
  - Edge cases: Telegram API downtime, malformed messages, chat ID missing.

- **Read Video Parameters from Google Sheet**  
  - Type: Google Sheets  
  - Role: Fetches associated video parameters (e.g., model type) from a configured Google Sheet document.  
  - Config: Uses OAuth2 credentials; document ID and sheet name configured by user.  
  - Inputs: Trigger data from Telegram node.  
  - Outputs: Video parameters JSON for prompt generation.  
  - Edge cases: Sheet access denied, empty or malformed data.

- **Set Master Prompt**  
  - Type: Set (Data Transformation)  
  - Role: Defines a detailed master prompt JSON schema for video generation, including cinematic style, camera, lighting, environment, VFX, audio, and ending descriptions.  
  - Config: Assigns a stringified JSON schema to a variable `json_master`.  
  - Inputs: Video parameters from Google Sheets.  
  - Outputs: Master prompt schema for AI agent.  
  - Edge cases: Incorrect JSON format or missing fields.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides a GPT-4.1-mini model instance for AI Agent usage.  
  - Config: Uses OpenAI API credentials.  
  - Inputs: Connected internally within AI Agent node.  
  - Outputs: Language model response for script generation.  
  - Edge cases: API rate limits, quota exhaustion, network errors.

- **Think (LangChain Tool)**  
  - Type: LangChain Think Tool  
  - Role: Auxiliary tool for internal reasoning or processing within the AI Agent.  
  - Config: Empty parameters, default behavior.  
  - Inputs/Outputs: Connected to AI Agent node as a tool.  
  - Edge cases: Internal tool errors.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI Agent output to enforce JSON schema with two keys: `title` and `final_prompt`.  
  - Config: Uses strict JSON schema example for validation.  
  - Inputs: AI Agent raw output.  
  - Outputs: Parsed, validated JSON object.  
  - Edge cases: Parsing failure due to malformed AI output.

- **AI Agent: Generate Video Script**  
  - Type: LangChain AI Agent  
  - Role: Central node that uses the above resources to generate a structured video ad script from the Telegram input.  
  - Config: System prompt instructing generation of a JSON object with `title` and `final_prompt`, strictly following the master prompt schema. Output must be a stringified JSON in a single line.  
  - Inputs: Telegram message text, master prompt schema, OpenAI Chat Model, Think tool, Structured Output Parser.  
  - Outputs: JSON with video title and final prompt string for VEO3.  
  - Edge cases: AI hallucination, invalid JSON output, API failures.

---

#### 2.2 Video Creation and Caption Processing

**Overview:**  
This block sends the AI-generated prompt to VEO3 for video generation, waits for rendering completion, downloads the rendered video, rewrites the video caption with GPT-4o, and appends all relevant metadata to Google Sheets.

**Nodes Involved:**  
- Generate Video with VEO3  
- Wait for VEO3 Rendering  
- Download Video from VEO3  
- Rewrite Caption with GPT-4o  
- Save Caption Video to Google Sheets

**Node Details:**

- **Generate Video with VEO3**  
  - Type: HTTP Request  
  - Role: Sends a POST request to VEO3 API to generate a video based on the final prompt.  
  - Config:  
    - URL: `https://api.kie.ai/api/v1/veo/generate`  
    - Body: JSON with keys `prompt` (from AI Agent), `model` (from Google Sheets), and fixed aspect ratio "16:9".  
    - Authentication: HTTP Header Auth with Kie AI credentials.  
  - Inputs: AI Agent output and video parameters.  
  - Outputs: JSON with a `taskId` for rendering status.  
  - Edge cases: API errors, invalid prompt, auth failures, network timeouts.

- **Wait for VEO3 Rendering**  
  - Type: Wait  
  - Role: Pauses execution for 10 minutes to allow VEO3 to complete video rendering.  
  - Config: Wait duration set in minutes.  
  - Inputs: Receives `taskId` from previous node.  
  - Outputs: Continues after delay.  
  - Edge cases: Fixed wait may be insufficient or excessively long.

- **Download Video from VEO3**  
  - Type: HTTP Request  
  - Role: Queries VEO3 API for the video download URL using the `taskId`.  
  - Config:  
    - URL: `https://api.kie.ai/api/v1/veo/record-info`  
    - Query parameter: taskId from Generate Video node.  
    - Authentication: HTTP Header Auth.  
  - Inputs: Task ID from Generate Video node.  
  - Outputs: JSON with video URLs array.  
  - Edge cases: API errors, task not completed, invalid taskId.

- **Rewrite Caption with GPT-4o**  
  - Type: OpenAI (LangChain OpenAI)  
  - Role: Rewrites the video caption for TikTok using GPT-4o based on the original idea and title.  
  - Config:  
    - Model: gpt-4o  
    - Prompt: Fixed prompt that instructs strict caption rewriting under 200 characters, no explanations, final output only.  
    - Credentials: OpenAI API.  
  - Inputs: Telegram video idea, AI Agent title, video metadata.  
  - Outputs: Caption text string.  
  - Edge cases: Caption longer than 200 chars, API failures.

- **Save Caption Video to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends a new row with the video title, final prompt, rewritten caption, original subject, and video URL to a Google Sheet.  
  - Config:  
    - Operation: Append  
    - Columns mapped explicitly.  
    - Uses OAuth2 credentials, document and sheet configured by user.  
  - Inputs: Caption from GPT-4o, video URL from VEO3 download, title and prompt metadata.  
  - Outputs: Confirmation of append.  
  - Edge cases: Permissions, rate limits, empty data.

---

#### 2.3 Multi-Platform Video Upload and Distribution

**Overview:**  
This block assigns social media account IDs, uploads the generated video to Blotato, and auto-posts it with captions on nine platforms (Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Bluesky, Pinterest). It also sends Telegram notifications of the video URL and final preview.

**Nodes Involved:**  
- Assign Social Media IDs  
- Upload Video to Blotato  
- INSTAGRAM (HTTP Request)  
- YOUTUBE (HTTP Request)  
- TIKTOK (HTTP Request)  
- FACEBOOK (HTTP Request)  
- THREADS (HTTP Request)  
- TWETTER (HTTP Request)  
- LINKEDIN (HTTP Request)  
- BLUESKY (HTTP Request)  
- PINTEREST (HTTP Request)  
- Send Video URL via Telegram  
- Send Final Video Preview

**Node Details:**

- **Assign Social Media IDs**  
  - Type: Set  
  - Role: Hardcodes or sets the social media account IDs and board/page IDs for all platforms used.  
  - Config: Assigns JSON with keys for each platform’s account/page IDs (all set to "1111" as placeholders).  
  - Inputs: Triggered after Save Caption node.  
  - Outputs: Social media IDs passed downstream.  
  - Edge cases: Missing or incorrect IDs will cause posting failures.

- **Upload Video to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads the video URL to Blotato media endpoint, which returns a media URL usable for posting.  
  - Config:  
    - URL: `https://backend.blotato.com/v2/media`  
    - Method: POST  
    - Body: JSON with `url` from Google Sheets video URL.  
    - Header: `blotato-api-key` (user must replace "YOUR_API").  
  - Inputs: Social media IDs, video URL.  
  - Outputs: Media URL for posting.  
  - Edge cases: API key invalid, upload failure.

- **Platform Posting Nodes (INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWETTER, LINKEDIN, BLUESKY, PINTEREST)**  
  - Type: HTTP Request (one per platform)  
  - Role: Posts the video as a social media post on the respective platform via Blotato’s API.  
  - Config:  
    - URL: `https://backend.blotato.com/v2/posts`  
    - Method: POST  
    - Body: JSON with `accountId` from Assign Social Media IDs, `target` specifying platform and sometimes additional fields (e.g., YouTube title, TikTok privacy), and `content` containing caption text (mostly from Google Sheets caption) and media URLs from Blotato upload.  
    - Header: `blotato-api-key` with user API key.  
  - Inputs: Media URL from Upload Video to Blotato, social media IDs, captions.  
  - Outputs: API response from social media posting.  
  - Edge cases: Invalid API key, invalid account IDs, quota limits, network errors, platform-specific restrictions.

- **Send Video URL via Telegram**  
  - Type: Telegram  
  - Role: Sends a Telegram message to the original chat with the video URL for direct access.  
  - Config: Uses chat ID from Telegram trigger, message text includes the video URL.  
  - Inputs: Video URL from Google Sheets.  
  - Outputs: Confirmation of message sent.  
  - Edge cases: Telegram API downtime, invalid chat ID.

- **Send Final Video Preview**  
  - Type: Telegram  
  - Role: Sends the actual video file or URL preview to the user on Telegram.  
  - Config: Uses chat ID from current JSON result, sends video with URL from Google Sheets.  
  - Inputs: Video URL from Google Sheets.  
  - Outputs: Confirmation of video sent.  
  - Edge cases: Large file size, Telegram API limits.

---

### 3. Summary Table

| Node Name                        | Node Type                        | Functional Role                                   | Input Node(s)                                | Output Node(s)                                                                                                                         | Sticky Note                                                                                   |
|---------------------------------|---------------------------------|-------------------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| Telegram Trigger: Receive Video Idea | Telegram Trigger                | Receive video idea from Telegram user            |                                              | Read Video Parameters from Google Sheet                                                                                               |                                                                                               |
| Read Video Parameters from Google Sheet | Google Sheets                   | Fetch video generation parameters                 | Telegram Trigger: Receive Video Idea          | Set Master Prompt                                                                                                                     |                                                                                               |
| Set Master Prompt                | Set                             | Define detailed master prompt JSON schema         | Read Video Parameters from Google Sheet       | AI Agent: Generate Video Script                                                                                                      |                                                                                               |
| OpenAI Chat Model                | LangChain OpenAI Chat Model     | Provide GPT-4.1-mini model instance               |                                              | AI Agent: Generate Video Script                                                                                                      |                                                                                               |
| Think                           | LangChain Think Tool            | Auxiliary AI processing tool                       |                                              | AI Agent: Generate Video Script                                                                                                      |                                                                                               |
| Structured Output Parser         | LangChain Structured Output Parser | Parse AI Agent output to enforce JSON schema      | AI Agent: Generate Video Script                | AI Agent: Generate Video Script                                                                                                      |                                                                                               |
| AI Agent: Generate Video Script | LangChain AI Agent              | Generate structured video ad script                | Telegram Trigger: Receive Video Idea, Set Master Prompt, OpenAI Chat Model, Think, Structured Output Parser | Generate Video with VEO3                                                                                                                |                                                                                               |
| Generate Video with VEO3         | HTTP Request                   | Send prompt to VEO3 API for video generation       | AI Agent: Generate Video Script                | Wait for VEO3 Rendering                                                                                                               |                                                                                               |
| Wait for VEO3 Rendering          | Wait                          | Wait fixed time for video rendering completion    | Generate Video with VEO3                        | Download Video from VEO3                                                                                                              |                                                                                               |
| Download Video from VEO3         | HTTP Request                   | Retrieve video download URL from VEO3 API         | Wait for VEO3 Rendering                         | Rewrite Caption with GPT-4o                                                                                                          |                                                                                               |
| Rewrite Caption with GPT-4o      | OpenAI (LangChain OpenAI)      | Rewrite video caption strictly under 200 characters | Download Video from VEO3                        | Save Caption Video to Google Sheets                                                                                                 |                                                                                               |
| Save Caption Video to Google Sheets | Google Sheets                   | Append video metadata and captions to Google Sheet | Rewrite Caption with GPT-4o                     | Send Video URL via Telegram                                                                                                          |                                                                                               |
| Send Video URL via Telegram      | Telegram                      | Send video URL notification to user               | Save Caption Video to Google Sheets             | Send Final Video Preview                                                                                                             |                                                                                               |
| Send Final Video Preview         | Telegram                      | Send video preview (video file or URL)             | Send Video URL via Telegram                      | Assign Social Media IDs                                                                                                              |                                                                                               |
| Assign Social Media IDs          | Set                           | Define social media account IDs for platforms      | Send Final Video Preview                         | Upload Video to Blotato                                                                                                              |                                                                                               |
| Upload Video to Blotato          | HTTP Request                  | Upload video URL to Blotato media endpoint         | Assign Social Media IDs                          | INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWETTER, LINKEDIN, BLUESKY, PINTEREST                                                 |                                                                                               |
| INSTAGRAM                      | HTTP Request                  | Post video to Instagram via Blotato                  | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| YOUTUBE                       | HTTP Request                  | Post video to YouTube via Blotato                     | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| TIKTOK                        | HTTP Request                  | Post video to TikTok via Blotato                      | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| FACEBOOK                      | HTTP Request                  | Post video to Facebook via Blotato                    | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| THREADS                       | HTTP Request                  | Post video to Threads via Blotato                     | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| TWETTER                       | HTTP Request                  | Post video to Twitter via Blotato                     | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| LINKEDIN                      | HTTP Request                  | Post video to LinkedIn via Blotato                    | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| BLUESKY                       | HTTP Request                  | Post video to Bluesky via Blotato                     | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| PINTEREST                     | HTTP Request                  | Post video to Pinterest via Blotato                   | Upload Video to Blotato                          |                                                                                                                                        |                                                                                               |
| Sticky Note                    | Sticky Note                   | Step 1 - Generate the Perfect Video Ad Script       |                                              |                                                                                                                                        | # 🟫 STEP 1 — Generate the Perfect Video Ad Script                                            |
| Sticky Note1                   | Sticky Note                   | Step 2 - Generate the Full Video with VEO3          |                                              |                                                                                                                                        | # 🟦 STEP 2 — Generate the Full Video with VEO3                                               |
| Sticky Note3                   | Sticky Note                   | Step 3 - Auto-Publish to 9 Social Media Platforms    |                                              |                                                                                                                                        | # 🟥 STEP 3 — Auto-Publish to 9 Social Media Platforms                                        |
| Sticky Note2                   | Sticky Note                   | Workflow documentation link                           |                                              |                                                                                                                                        | # Create and Auto-Post Viral AI Videos with VEO3 and Blotato to 9 Platforms (By Dr. Firas) - **Documentation** : [NOTION](https://automatisation.notion.site/Create-and-Auto-Post-Viral-AI-Videos-with-VEO3-and-Blotato-to-9-Platforms-23f3d6550fd98037ab1ee9f803400666?source=copy_link) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger: Receive Video Idea**  
   - Node type: Telegram Trigger  
   - Configure to listen for "message" updates.  
   - Use your Telegram API credentials.  
   - This node starts the workflow with user input.

2. **Add Google Sheets Node: Read Video Parameters from Google Sheet**  
   - Node type: Google Sheets  
   - Operation: Read rows or configured to retrieve parameters by ID.  
   - Configure with your Google Sheets OAuth2 credentials, document ID, and sheet name where video parameters are stored (e.g., model type).  
   - Connect Telegram Trigger output to this node input.

3. **Add Set Node: Set Master Prompt**  
   - Node type: Set  
   - Assign a stringified JSON to a variable (e.g., `json_master`) defining the detailed video prompt schema: cinematic style, camera, lighting, environment, VFX, audio, ending, format, keywords, etc.  
   - Connect Google Sheets output to this node.

4. **Add LangChain OpenAI Chat Model Node: OpenAI Chat Model**  
   - Node type: LangChain OpenAI Chat Model  
   - Select GPT-4.1-mini model.  
   - Use your OpenAI API credentials.

5. **Add LangChain Think Tool Node: Think**  
   - Node type: LangChain Tool (Think)  
   - No special configuration needed.

6. **Add LangChain Structured Output Parser Node: Structured Output Parser**  
   - Node type: LangChain Output Parser Structured  
   - Configure JSON schema example with two string keys: `title` and `final_prompt`.

7. **Add LangChain AI Agent Node: AI Agent: Generate Video Script**  
   - Node type: LangChain Agent  
   - Configure with:  
     - Input text: Telegram message text.  
     - System prompt: Instruct to generate a strict JSON with `title` and `final_prompt` following the master schema.  
     - Assign the OpenAI Chat Model, Think tool, and Structured Output Parser nodes as resources.  
   - Connect Set Master Prompt output and Telegram Trigger message to this node.

8. **Add HTTP Request Node: Generate Video with VEO3**  
   - Node type: HTTP Request  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/veo/generate`  
   - Body: JSON with keys:  
     - `prompt`: from AI Agent output `final_prompt` (parsed JSON string).  
     - `model`: from Google Sheets video parameters.  
     - `aspectRatio`: fixed "16:9".  
   - Authentication: HTTP Header with your Kie AI API credentials.  
   - Connect AI Agent output to this node.

9. **Add Wait Node: Wait for VEO3 Rendering**  
   - Node type: Wait  
   - Set time to 10 minutes (adjustable).  
   - Connect Generate Video with VEO3 output.

10. **Add HTTP Request Node: Download Video from VEO3**  
    - Node type: HTTP Request  
    - Method: GET  
    - URL: `https://api.kie.ai/api/v1/veo/record-info`  
    - Query Parameter: `taskId` from Generate Video with VEO3 node output.  
    - Authentication: HTTP Header with Kie AI API credentials.  
    - Connect Wait node output here.

11. **Add OpenAI Node: Rewrite Caption with GPT-4o**  
    - Node type: LangChain OpenAI  
    - Model: GPT-4o  
    - Prompt: Fixed instructions for rewriting TikTok captions, limiting output to max 200 characters, no explanations.  
    - Input: Original Telegram idea and AI Agent generated title.  
    - Use OpenAI API credentials.  
    - Connect Download Video output here.

12. **Add Google Sheets Node: Save Caption Video to Google Sheets**  
    - Node type: Google Sheets  
    - Operation: Append row  
    - Columns mapped: Title, Prompt, Caption, Subject (original idea), URL Video (downloaded video URL).  
    - Use Google Sheets OAuth2 credentials, target document and sheet.  
    - Connect Rewrite Caption output here.

13. **Add Telegram Node: Send Video URL via Telegram**  
    - Node type: Telegram  
    - Operation: Send message  
    - Chat ID: from original Telegram Trigger chat ID  
    - Text: "Url VIDEO : {{ video URL }}" from Google Sheets data.  
    - Use Telegram API credentials.  
    - Connect Save Caption node output.

14. **Add Telegram Node: Send Final Video Preview**  
    - Node type: Telegram  
    - Operation: Send video  
    - Chat ID: from Telegram message context  
    - File: Video URL from Google Sheets  
    - Use Telegram API credentials.  
    - Connect Send Video URL node output.

15. **Add Set Node: Assign Social Media IDs**  
    - Node type: Set  
    - Define hardcoded or dynamic account IDs for Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Bluesky, Pinterest, including board/page IDs where applicable.  
    - Connect Send Final Video Preview output.

16. **Add HTTP Request Node: Upload Video to Blotato**  
    - Node type: HTTP Request  
    - Method: POST  
    - URL: `https://backend.blotato.com/v2/media`  
    - Body: JSON with `url` from Google Sheets video URL.  
    - Header: `blotato-api-key` (replace with your API key).  
    - Connect Assign Social Media IDs output.

17. **Add Nine HTTP Request Nodes for Each Platform (Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Bluesky, Pinterest)**  
    - Node type: HTTP Request (one per platform)  
    - Method: POST  
    - URL: `https://backend.blotato.com/v2/posts`  
    - Body: JSON with:  
      - `accountId`: From Assign Social Media IDs node.  
      - `target`: Platform-specific parameters (e.g., targetType, privacy, pageId, boardId).  
      - `content`: Caption from Google Sheets, mediaUrls from Blotato upload response.  
    - Header: `blotato-api-key` with your API key.  
    - Connect Upload Video to Blotato output to all nine nodes.

18. **Activate and Test Workflow**  
    - Ensure all credentials are correctly set.  
    - Replace placeholder IDs and API keys with real values.  
    - Test with sample Telegram input to verify video creation and multi-platform posting.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The workflow is by Dr. Firas and documented in detail on Notion with a comprehensive step-by-step guide and explanation. | [NOTION Documentation](https://automatisation.notion.site/Create-and-Auto-Post-Viral-AI-Videos-with-VEO3-and-Blotato-to-9-Platforms-23f3d6550fd98037ab1ee9f803400666?source=copy_link) |
| The workflow uses the VEO3 API by Kie AI for video generation — ensure you have valid API credentials and usage quotas. | https://api.kie.ai/ |
| Blotato API requires an API key for media upload and posting — replace "YOUR_API" placeholders accordingly. | https://blotato.com/ |
| Telegram API requires configuration of webhook and bot token for message reception and sending. | https://core.telegram.org/bots/api |
| Google Sheets OAuth2 credentials must have read and write permissions for the target spreadsheet. | https://developers.google.com/sheets/api/guides/authorizing |
| Captions are rewritten with GPT-4o with a strict 200 character limit to optimize for TikTok and other short-form platforms. |  |
| The 10-minute wait for VEO3 rendering is fixed and may be adjusted based on actual rendering times observed. |  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.

---