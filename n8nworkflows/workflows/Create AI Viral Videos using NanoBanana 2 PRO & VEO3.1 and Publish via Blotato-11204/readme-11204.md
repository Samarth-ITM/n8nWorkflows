Create AI Viral Videos using NanoBanana 2 PRO & VEO3.1 and Publish via Blotato

https://n8nworkflows.xyz/workflows/create-ai-viral-videos-using-nanobanana-2-pro---veo3-1-and-publish-via-blotato-11204


# Create AI Viral Videos using NanoBanana 2 PRO & VEO3.1 and Publish via Blotato

### 1. Workflow Overview

This workflow automates the creation and multi-platform publishing of AI-generated viral videos using NanoBanana 2 PRO and VEO3.1, triggered by a Telegram message containing a video idea and reference image. It proceeds through three main logical blocks:

- **1.1 Input Reception and Preparation:** Receives the video idea and reference image from Telegram, retrieves and analyzes the image, and sets up master prompts and tokens.
- **1.2 AI Content Generation and Media Creation:** Generates structured video and image prompts using OpenAI models and LangChain agents, creates an edited image with NanoBanana 2 PRO, generates a video via VEO3.1 based on the prompts and image, and prepares the final video for delivery.
- **1.3 Multi-Platform Publishing and Notification:** Uploads the produced video to Blotato and publishes it to multiple social media platforms (YouTube, TikTok, Instagram, LinkedIn, Facebook, Twitter/X), followed by sending confirmation messages via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

**Overview:**  
This block listens for user input on Telegram, retrieves the submitted image, analyzes it with AI to extract descriptive metadata, and prepares master prompts and tokens to be used downstream.

**Nodes Involved:**  
- Telegram Trigger: Receive Video Idea  
- Set: Bot Token (Placeholder)  
- Telegram API: Get File URL  
- OpenAI Vision: Analyze Reference Image  
- Set Master Prompt  

**Node Details:**

- **Telegram Trigger: Receive Video Idea**  
  - Type: Telegram Webhook Trigger  
  - Role: Receives incoming messages (updates) from Telegram users, including video ideas with images and captions.  
  - Config: Triggers on message updates only. Uses Telegram API credentials.  
  - Inputs: External Telegram message event.  
  - Outputs: Message content including caption and photo array.  
  - Edge Cases: Input message without photo or caption may cause downstream failures. Telegram API rate limits or connectivity issues possible.

- **Set: Bot Token (Placeholder)**  
  - Type: Set node  
  - Role: Stores placeholders for Telegram Bot Token, FAL API key, and caption extracted from Telegram message.  
  - Config: Assigns `YOUR_BOT_TOKEN`, `fal_api_key`, and `CAPTION` variables. `CAPTION` dynamically set from Telegram message caption.  
  - Inputs: Telegram message JSON.  
  - Outputs: JSON with tokens and caption for further use.  
  - Edge Cases: Tokens must be replaced with actual credentials; missing tokens cause API request failures.

- **Telegram API: Get File URL**  
  - Type: HTTP Request  
  - Role: Retrieves the direct file path URL for the last photo in the Telegram message.  
  - Config: Uses Telegram Bot Token to call `getFile` endpoint with the file_id of the last photo in array.  
  - Inputs: JSON with `YOUR_BOT_TOKEN` and Telegram message photo array.  
  - Outputs: JSON containing the `file_path` used to access the image.  
  - Edge Cases: Invalid token or file_id causes API error; missing photo array causes failure.

- **OpenAI Vision: Analyze Reference Image**  
  - Type: LangChain OpenAI Image Analysis node  
  - Role: Analyzes the retrieved reference image and outputs a YAML description with details about product or character visible in the image.  
  - Config: Uses GPT-4o model, outputs YAML only, no explanation text.  
  - Inputs: Image URL constructed from Telegram file_path and Bot Token.  
  - Outputs: YAML string describing color schemes, brand, outfit style, and visual description.  
  - Edge Cases: Image URL invalid or inaccessible; OCR/vision model misinterpretation; rate limits or timeouts.

- **Set Master Prompt**  
  - Type: Set node  
  - Role: Defines a detailed JSON schema template called `json_master` for structured video ad prompt generation.  
  - Config: Contains rich schema with fields for description, style, camera, lighting, environment, motion, VFX, audio, ending, text, format, and keywords.  
  - Inputs: Triggered after image download.  
  - Outputs: JSON variable `json_master` passed to AI prompt generator.  
  - Edge Cases: Misconfiguration or invalid JSON could cause parsing failures downstream.

---

#### 2.2 AI Content Generation and Media Creation

**Overview:**  
This block generates natural language prompts for image and video creation using AI agents; creates the edited image with NanoBanana 2 PRO; generates the video using VEO3.1; and prepares the outputs for publishing.

**Nodes Involved:**  
- Generate Image Prompt  
- NanoBanana: Create Image  
- Wait for Image Edit  
- Download Edited Image  
- AI Agent: Generate Video Script  
- Parse GPT Response  
- Optimize Prompt for Veo  
- Prepare Veo Request Body  
- Veo Generation  
- Wait  
- Download Video  
- LLM: OpenAI Chat  
- LLM: Structured Output Parser  

**Node Details:**

- **Generate Image Prompt**  
  - Type: LangChain Agent node  
  - Role: Generates a concise, realistic UGC-style image prompt based on the user description (caption) and the analyzed reference image YAML.  
  - Config: System prompt enforces JSON-only output with `image_prompt` key, tone casual and lifelike, max 120 words, includes camera cues and text accuracy.  
  - Inputs: Caption from Telegram and YAML from image analysis.  
  - Outputs: JSON with `image_prompt` string.  
  - Edge Cases: AI model may misinterpret or output invalid JSON; missing inputs cause failure.

- **NanoBanana: Create Image**  
  - Type: HTTP Request  
  - Role: Sends the generated image prompt and reference image URL to NanoBanana 2 PRO API to create an edited image (1K resolution, 9:16 aspect ratio).  
  - Config: POST request with JSON body including escaped image prompt and image URLs; authorization header uses FAL API key.  
  - Inputs: JSON from Generate Image Prompt.  
  - Outputs: JSON with response URL to edited image.  
  - Edge Cases: API downtime, invalid API key, malformed request, rate limit.

- **Wait for Image Edit**  
  - Type: Wait node  
  - Role: Pauses workflow 2 seconds to allow image processing to complete on NanoBanana server.  
  - Config: Fixed 2 seconds delay.  
  - Inputs: After NanoBanana image request.  
  - Outputs: Passes control downstream.  
  - Edge Cases: Delay may be insufficient if image processing takes longer; no error detection.

- **Download Edited Image**  
  - Type: HTTP Request  
  - Role: Downloads the edited image from the NanoBanana API using the response URL.  
  - Config: GET request to the URL provided by NanoBanana.  
  - Inputs: Response URL from NanoBanana.  
  - Outputs: JSON containing image URLs for video generation.  
  - Edge Cases: Network issues, invalid URL, timeout.

- **AI Agent: Generate Video Script**  
  - Type: LangChain Agent node  
  - Role: Generates a structured JSON video prompt including `prompt`, `caption`, `title`, and `hashtags` based on the user description and image analysis content, following a detailed master schema.  
  - Config: Uses GPT-4.1-mini model, with strict output constraints (valid JSON only, escaped strings).  
  - Inputs: Master prompt JSON, user caption, and image analysis data.  
  - Outputs: JSON with video script content.  
  - Edge Cases: Model output not JSON, invalid formatting, API errors.

- **Parse GPT Response**  
  - Type: Code node  
  - Role: Parses and normalizes the AI-generated video script JSON to extract title, prompt, caption, and hashtags (array and string form), handling legacy and new formats.  
  - Config: JavaScript code with robust parsing and fallback logic.  
  - Inputs: AI Agent output JSON.  
  - Outputs: Clean JSON fields for downstream use.  
  - Edge Cases: Unexpected AI response format, JSON parse errors.

- **Optimize Prompt for Veo**  
  - Type: Set node  
  - Role: Appends fixed cinematic style instructions and technical details (duration, aspect ratio, fps) to the video prompt for VEO3.1 compatibility.  
  - Config: Adds "consistent character throughout, photorealistic quality, professional cinematography, 8 seconds duration, 9:16 aspect ratio, 24fps" to prompt string.  
  - Inputs: Parsed prompt.  
  - Outputs: `veo_prompt` field alongside other data.  
  - Edge Cases: Empty or invalid prompt causes issues downstream.

- **Prepare Veo Request Body**  
  - Type: Code node  
  - Role: Constructs the JSON request body for VEO API, including the optimized prompt and the edited image URL from NanoBanana output.  
  - Config: Validates presence and format of prompt and image URL before building request body with duration and aspect ratio.  
  - Inputs: `veo_prompt` and edited image URL.  
  - Outputs: JSON with `veo_request_body`.  
  - Edge Cases: Missing prompt or image URL throws error, halting workflow.

- **Veo Generation**  
  - Type: HTTP Request  
  - Role: Sends the video prompt and image URLs to VEO3.1 API to generate a vertical AI video (up to 8 seconds).  
  - Config: POST JSON request with authorization header (FAL API key), 10-minute timeout.  
  - Inputs: JSON `veo_request_body`.  
  - Outputs: JSON response with video URL and metadata.  
  - Edge Cases: API timeout, authorization error, invalid request.

- **Wait**  
  - Type: Wait node  
  - Role: Pauses 2 seconds post video generation request for processing time.  
  - Config: Fixed delay.  
  - Inputs: After video generation request.  
  - Outputs: Passes control downstream.  
  - Edge Cases: Fixed delay may not cover longer processing.

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads the generated video file from the VEO response URL for local or downstream use.  
  - Config: GET request, expects file response format.  
  - Inputs: Video URL from VEO generation.  
  - Outputs: Binary video data and metadata.  
  - Edge Cases: Network issues, invalid video URL.

- **LLM: OpenAI Chat**  
  - Type: LangChain OpenAI Chat node  
  - Role: Supports generation of intermediate text outputs (likely connected to image prompt generation).  
  - Config: Uses GPT-4.1-mini model.  
  - Inputs: Receives system message and user prompt from upstream.  
  - Outputs: Text response parsed by next node.  
  - Edge Cases: API limits, improper prompt.

- **LLM: Structured Output Parser**  
  - Type: Output parser node for LangChain  
  - Role: Parses raw LLM output into structured JSON using example schema.  
  - Config: Example schema for image prompt JSON with `image_prompt` key.  
  - Inputs: Raw LLM chat output.  
  - Outputs: Parsed JSON object.  
  - Edge Cases: Parsing errors if output not in expected format.

---

#### 2.3 Multi-Platform Publishing and Notification

**Overview:**  
This block uploads the final AI-generated video to Blotato and publishes it automatically across multiple social platforms, followed by confirmation messages to the user on Telegram.

**Nodes Involved:**  
- Send Video to Telegram  
- Upload Video to BLOTATO  
- Tiktok  
- Linkedin  
- Facebook  
- Instagram  
- Twitter (X)  
- Youtube  
- Merge1  
- Send a text message  
- Sticky Notes (informative)

**Node Details:**

- **Send Video to Telegram**  
  - Type: Telegram node  
  - Role: Sends the generated video back to the Telegram chat with a caption including the video URL.  
  - Config: Uses Telegram credentials, sends video as binary data, dynamic chat ID from trigger.  
  - Inputs: Binary video data from Download Video node.  
  - Outputs: Telegram message confirmation JSON.  
  - Edge Cases: Telegram API limits, video size constraints.

- **Upload Video to BLOTATO**  
  - Type: Blotato node (media resource)  
  - Role: Uploads the downloaded video to Blotato platform for social publishing.  
  - Config: Uses Blotato API credentials, mediaUrl set from downloaded video URL.  
  - Inputs: Video URL from Download Video.  
  - Outputs: Media upload response with URLs.  
  - Edge Cases: API key invalid, upload failure, network issues.

- **Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube**  
  - Type: Blotato nodes, each for a social platform  
  - Role: Publishes the uploaded video media with text captions to respective platforms.  
  - Config: Each node configured with platform-specific account ID, post text from parsed GPT caption, media URLs from Blotato upload, and platform-specific options (e.g., YouTube privacy set to private).  
  - Inputs: Media URLs and captions from Upload Video to BLOTATO and Parse GPT Response nodes.  
  - Outputs: Posting confirmation JSON.  
  - Edge Cases: Platform API limits, authentication errors, content policy restrictions.

- **Merge1**  
  - Type: Merge node with chooseBranch mode  
  - Role: Collects outputs from all social media publishing nodes to synchronize downstream confirmation.  
  - Config: Awaits all 6 inputs before continuing.  
  - Inputs: Outputs from all social platform nodes.  
  - Outputs: Merged data for notification.  
  - Edge Cases: One platform failing blocks confirmation.

- **Send a text message**  
  - Type: Telegram node  
  - Role: Sends a simple "Published" confirmation message to the original Telegram chat.  
  - Config: Dynamic chat ID, static text message.  
  - Inputs: Merged node output indicating all posts done.  
  - Outputs: Confirmation message JSON.  
  - Edge Cases: Telegram API failure.

- **Sticky Notes**  
  - Provide contextual labels for each major step:  
    - Step 1: Image creation with NanoBanana 2 PRO  
    - Step 2: Video generation with VEO3.1  
    - Step 3: Multi-platform publishing with Blotato  
  - One sticky note includes documentation links and branding info.

---

### 3. Summary Table

| Node Name                       | Node Type                               | Functional Role                            | Input Node(s)                             | Output Node(s)                                 | Sticky Note                                                                                                                      |
|--------------------------------|---------------------------------------|-------------------------------------------|------------------------------------------|------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger: Receive Video Idea | Telegram Trigger                      | Input reception from Telegram              | —                                        | Set: Bot Token (Placeholder)                    | # 📑 STEP 1 — Create Image with NanoBanana 2 PRO                                                                               |
| Set: Bot Token (Placeholder)    | Set                                   | Set API tokens and caption variable        | Telegram Trigger: Receive Video Idea     | Telegram API: Get File URL                       |                                                                                                                                |
| Telegram API: Get File URL      | HTTP Request                         | Get Telegram image file URL                 | Set: Bot Token (Placeholder)              | OpenAI Vision: Analyze Reference Image          |                                                                                                                                |
| OpenAI Vision: Analyze Reference Image | LangChain OpenAI Vision              | Analyze image content to YAML description  | Telegram API: Get File URL                 | Generate Image Prompt                            |                                                                                                                                |
| Generate Image Prompt           | LangChain Agent                      | Generate realistic UGC image prompt        | OpenAI Vision: Analyze Reference Image   | NanoBanana: Create Image                         |                                                                                                                                |
| NanoBanana: Create Image        | HTTP Request                         | Create edited image with NanoBanana 2 PRO  | Generate Image Prompt                      | Wait for Image Edit                              |                                                                                                                                |
| Wait for Image Edit             | Wait                                 | Pause for image processing                  | NanoBanana: Create Image                   | Download Edited Image                            |                                                                                                                                |
| Download Edited Image           | HTTP Request                         | Download edited image from NanoBanana      | Wait for Image Edit                        | Set Master Prompt                                |                                                                                                                                |
| Set Master Prompt               | Set                                   | Define master prompt JSON schema            | Download Edited Image                      | AI Agent: Generate Video Script                  |                                                                                                                                |
| OpenAI Chat Model               | LangChain OpenAI Chat                | Language model for video script generation | Set Master Prompt                         | Think                                            |                                                                                                                                |
| Think                          | LangChain Tool (Think)               | AI thinking tool for intermediate processing | OpenAI Chat Model                         | AI Agent: Generate Video Script                  |                                                                                                                                |
| Structured Output Parser        | LangChain Output Parser              | Parse structured AI output                   | Think                                     | AI Agent: Generate Video Script                  |                                                                                                                                |
| AI Agent: Generate Video Script | LangChain Agent                      | Generate structured video prompt JSON       | Structured Output Parser, OpenAI Chat Model | Parse GPT Response                             |                                                                                                                                |
| Parse GPT Response             | Code                                  | Parse and normalize video script JSON       | AI Agent: Generate Video Script           | Optimize Prompt for Veo                           |                                                                                                                                |
| Optimize Prompt for Veo         | Set                                   | Enhance prompt with cinematic instructions  | Parse GPT Response                        | Prepare Veo Request Body                          |                                                                                                                                |
| Prepare Veo Request Body        | Code                                  | Build VEO3.1 API request body                | Optimize Prompt for Veo                   | Veo Generation                                   |                                                                                                                                |
| Veo Generation                 | HTTP Request                         | Generate video from prompt and image         | Prepare Veo Request Body                  | Wait                                            |                                                                                                                                |
| Wait                           | Wait                                 | Pause for video generation processing         | Veo Generation                           | Download Video                                   |                                                                                                                                |
| Download Video                 | HTTP Request                         | Download generated video file                 | Wait                                     | Send Video to Telegram                            |                                                                                                                                |
| Send Video to Telegram          | Telegram                             | Send generated video back to Telegram user  | Download Video                           | Upload Video to BLOTATO                           |                                                                                                                                |
| Upload Video to BLOTATO         | Blotato                             | Upload video to Blotato platform              | Send Video to Telegram                    | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube | # 📑 STEP 3 — Publish with Blotato                                                                                             |
| Tiktok                        | Blotato                             | Publish video on TikTok                        | Upload Video to BLOTATO                   | Merge1                                           |                                                                                                                                |
| Linkedin                      | Blotato                             | Publish video on LinkedIn                      | Upload Video to BLOTATO                   | Merge1                                           |                                                                                                                                |
| Facebook                      | Blotato                             | Publish video on Facebook                      | Upload Video to BLOTATO                   | Merge1                                           |                                                                                                                                |
| Instagram                     | Blotato                             | Publish video on Instagram                     | Upload Video to BLOTATO                   | Merge1                                           |                                                                                                                                |
| Twitter (X)                  | Blotato                             | Publish video on Twitter (X)                   | Upload Video to BLOTATO                   | Merge1                                           |                                                                                                                                |
| Youtube                      | Blotato                             | Publish video on YouTube                        | Upload Video to BLOTATO                   | Merge1                                           |                                                                                                                                |
| Merge1                        | Merge                              | Synchronize multi-platform publishing output | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube | Send a text message                             |                                                                                                                                |
| Send a text message           | Telegram                           | Notify Telegram user of publication completion | Merge1                                  | —                                                |                                                                                                                                |
| Sticky Note                  | Sticky Note                       | Workflow step label and documentation links   | —                                        | —                                                | # 🚀 AI Viral Video Workflow — NanoBanana 2 PRO × VEO3.1 × Blotato (By Dr. Firas)\n\n[![AI Voice Agent Preview](https://www.dr-firas.com/nanobanana2.png)](https://youtu.be/nlwpbXQqNQ4)\n\n## 📘 Documentation  \nAccess detailed setup instructions, API config, platform connection guides, and workflow customization tips:\n\n📎 [Open the full documentation on Notion](https://automatisation.notion.site/NonoBanan-PRO-2-2b53d6550fd981a5acbecf7cf50aeb3c?source=copy_link) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger: Receive Video Idea**  
   - Node Type: Telegram Trigger  
   - Config: Listen for `message` updates only. Use Telegram API credentials.  
   - Purpose: Receive user message with photo and caption.

2. **Create Set node: Set Bot Token (Placeholder)**  
   - Node Type: Set  
   - Assign variables:  
     - `YOUR_BOT_TOKEN`: your Telegram bot token (string).  
     - `fal_api_key`: your FAL API key (string).  
     - `CAPTION`: expression `={{ $('Telegram Trigger: Receive Video Idea').item.json.message.caption }}`.

3. **Create HTTP Request: Telegram API: Get File URL**  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.telegram.org/bot{{ $json.YOUR_BOT_TOKEN }}/getFile?file_id={{ $('Telegram Trigger: Receive Video Idea').item.json.message.photo[ $('Telegram Trigger: Receive Video Idea').item.json.message.photo.length - 1].file_id }}`  
   - Use credentials from Step 2.  
   - Purpose: Retrieve file path for last photo.

4. **Create OpenAI Vision: Analyze Reference Image**  
   - Node Type: LangChain OpenAI  
   - Model: `chatgpt-4o-latest`  
   - Input: Image URL constructed as `https://api.telegram.org/file/bot{{ $json.YOUR_BOT_TOKEN }}/{{ $json.result.file_path }}`  
   - Prompt: YAML-only analysis as per detailed specifications (product and character analysis).  
   - Use OpenAI credentials.

5. **Create LangChain Agent: Generate Image Prompt**  
   - Node Type: LangChain Agent  
   - System prompt: UGC Image Prompt Builder with strict JSON output for `image_prompt`.  
   - Inputs: User caption and image analysis YAML.  
   - Output parser: LangChain Structured Output Parser with schema `{ "image_prompt": "string" }`.

6. **Create HTTP Request: NanoBanana: Create Image**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/nano-banana-pro/edit`  
   - Headers: `Content-Type: application/json`, `Authorization: Key {{ $json.fal_api_key }}`  
   - Body: JSON including escaped `image_prompt`, Telegram image URL, resolution 1K, aspect ratio 9:16, output_format png.

7. **Create Wait node: Wait for Image Edit**  
   - Node Type: Wait  
   - Duration: 2 seconds  
   - Purpose: Allow NanoBanana image processing time.

8. **Create HTTP Request: Download Edited Image**  
   - Node Type: HTTP Request  
   - URL: Use `response_url` from NanoBanana create image response.  
   - Method: GET

9. **Create Set node: Set Master Prompt**  
   - Node Type: Set  
   - Assign variable `json_master` with detailed video prompt schema JSON for video generation.

10. **Create LangChain OpenAI Chat node: OpenAI Chat Model**  
    - Model: `gpt-4.1-mini`  
    - Purpose: Support AI video script generation.

11. **Create LangChain Tool Think node: Think**  
    - Purpose: Intermediate AI tool step.

12. **Create LangChain Output Parser: Structured Output Parser**  
    - Schema example JSON for video prompt `{ prompt, caption, title, hashtags }`.

13. **Create LangChain Agent: AI Agent: Generate Video Script**  
    - Uses previous nodes for input and output parsing.  
    - System prompt enforces structured video prompt generation with strict JSON output.

14. **Create Code node: Parse GPT Response**  
    - JavaScript to parse AI response to normalized fields: `title`, `prompt`, `caption`, `hashtags` (array and string).

15. **Create Set node: Optimize Prompt for Veo**  
    - Append cinematic style, duration, aspect ratio, fps to prompt string as `veo_prompt`.

16. **Create Code node: Prepare Veo Request Body**  
    - Validate `veo_prompt` and edited image URL, create JSON body for VEO API with prompt, image URLs, duration (8s), aspect ratio (9:16).

17. **Create HTTP Request: Veo Generation**  
    - Method: POST  
    - URL: `https://fal.run/fal-ai/veo3.1/reference-to-video`  
    - Headers: Authorization with FAL API key, Content-Type application/json  
    - Body: `veo_request_body` JSON  
    - Timeout: 600 seconds

18. **Create Wait node: Wait**  
    - Duration: 2 seconds  
    - Purpose: Wait for video generation processing.

19. **Create HTTP Request: Download Video**  
    - Method: GET  
    - URL: Video URL from VEO response  
    - Response format: file (binary)

20. **Create Telegram node: Send Video to Telegram**  
    - Operation: sendVideo  
    - Chat ID: from original Telegram trigger message  
    - Caption: "Your video is ready! 🎥 {{video url}}"  
    - Send video as binary data.

21. **Create Blotato node: Upload Video to BLOTATO**  
    - Media URL: from downloaded video URL  
    - Use Blotato API credentials.

22. **Create Blotato nodes for each social platform**  
    - Platforms: TikTok, LinkedIn, Facebook, Instagram, Twitter (X), YouTube  
    - Configure each with account IDs, post text (caption from Parse GPT Response), media URLs from upload node  
    - YouTube post privacy set to private, no subscriber notifications.

23. **Create Merge node: Merge1**  
    - Mode: chooseBranch  
    - Number inputs: 6 (one per social platform)  
    - Purpose: Synchronize publishing completion.

24. **Create Telegram node: Send a text message**  
    - Text: "Published"  
    - Chat ID: from Telegram trigger message  
    - Triggered after Merge1 node.

25. **Add Sticky Notes**  
    - Add descriptive sticky notes for Steps 1, 2, 3 with color coding and documentation link as in original workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| AI Viral Video Workflow — NanoBanana 2 PRO × VEO3.1 × Blotato by Dr. Firas | Branding and workflow origin |
| Preview video and documentation: [YouTube Preview](https://youtu.be/nlwpbXQqNQ4) | Video preview of workflow in action |
| Full setup and documentation: [Notion Documentation](https://automatisation.notion.site/NonoBanan-PRO-2-2b53d6550fd981a5acbecf7cf50aeb3c?source=copy_link) | Detailed instructions, API setup, and customization tips |
| Blotato account required with Pro plan and API key | https://blotato.com/?ref=firas |
| Ensure “Verified Community Nodes” are enabled in n8n admin settings for Blotato nodes | n8n platform configuration |
| Use valid OpenAI API and Telegram Bot credentials | API credentials management |

---

**Disclaimer:**  
The provided text originates exclusively from an n8n automated workflow. It complies fully with all applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.