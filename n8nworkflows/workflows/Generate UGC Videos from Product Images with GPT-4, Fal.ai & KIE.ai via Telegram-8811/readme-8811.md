Generate UGC Videos from Product Images with GPT-4, Fal.ai & KIE.ai via Telegram

https://n8nworkflows.xyz/workflows/generate-ugc-videos-from-product-images-with-gpt-4--fal-ai---kie-ai-via-telegram-8811


# Generate UGC Videos from Product Images with GPT-4, Fal.ai & KIE.ai via Telegram

### 1. Workflow Overview

This workflow automates the creation of user-generated content (UGC) style videos from product images sent via Telegram. It leverages AI technologies (GPT-4 Vision, LangChain agents, Fal.ai, and KIE.ai) to analyze images, generate realistic image prompts, create images, develop video scene prompts, generate videos, and finally send the composed video back via Telegram.

**Use Cases:**
- Marketing teams creating authentic UGC videos for product promotion
- Social media managers automating personalized video content creation
- Influencers or brands needing fast content generation from product images

**Logical Blocks:**

- **1.1 Input Reception and Image Retrieval:** Receives images via Telegram and retrieves the actual image file path.
- **1.2 Image Analysis:** Uses GPT-4 Vision to classify and describe the image content (character, product).
- **1.3 Image Prompt Generation and Creation:** Generates detailed image prompts and creates new images via Fal.ai.
- **1.4 Video Prompt Generation:** Uses GPT-4-based agents to produce structured video scene prompts.
- **1.5 Video Creation and Processing:** Calls KIE.ai to generate video scenes and merges them into a final video.
- **1.6 Delivery:** Sends generated images and the final video back to the Telegram user.
- **1.7 Support and Setup Notes:** Contains sticky notes for setup instructions, creator info, and workflow explanations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Image Retrieval

**Overview:**  
This block triggers on incoming Telegram messages with images, extracts the image file ID, retrieves the downloadable file path, and prepares it for further processing.

**Nodes Involved:**  
- Telegram Trigger  
- Botid 1 (Set)  
- Image Pqath (HTTP Request)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages containing images.  
  - Configuration: Monitors "message" updates only, uses Telegram API credentials "Telegram UGC AD VIP."  
  - Input: Telegram webhook triggers on new message.  
  - Output: JSON with message data including photo file IDs.  
  - Edge Cases: No image in message, Telegram API downtime, invalid webhook setup.

- **Botid 1**  
  - Type: Set  
  - Role: Stores Telegram Bot Token as a variable "bot id" for API calls.  
  - Configuration: Static string assignment for bot token placeholder "YOUR_TELEGRAM_BOT_TOKEN."  
  - Input: Triggered by Telegram Trigger output.  
  - Output: Adds "bot id" to JSON data.  
  - Edge Cases: Missing or invalid bot token will cause API failures downstream.

- **Image Pqath**  
  - Type: HTTP Request  
  - Role: Calls Telegram API to get the downloadable file path for the sent photo.  
  - Configuration: GET request to `https://api.telegram.org/bot{{bot id}}/getFile?file_id={{photo file_id}}`.  
  - Input: Needs file_id from Telegram Trigger and bot token from Botid 1.  
  - Output: JSON containing file path of the photo.  
  - Edge Cases: Invalid file ID or expired file, API rate limits.

---

#### 1.2 Image Analysis

**Overview:**  
Analyzes the referenced image using GPT-4 Vision to classify the image as character, product, or both, and extracts detailed descriptive content.

**Nodes Involved:**  
- Analyze image (OpenAI GPT-4 Vision)

**Node Details:**

- **Analyze image**  
  - Type: OpenAI GPT-4 Vision (LangChain node)  
  - Role: Receives image URL, prompts GPT-4 Vision to describe contents (character/product) with details.  
  - Configuration:  
    - Model: chatgpt-4o-latest  
    - Operation: "analyze" on image URLs constructed from Telegram file path.  
    - Prompt: Instruction to only provide descriptive details without other text.  
  - Input: Image URL from Image Pqath node output.  
  - Output: Text description of image content.  
  - Edge Cases: Image URL invalid, OpenAI API rate limits or auth failure, unexpected image content.

---

#### 1.3 Image Prompt Generation and Creation

**Overview:**  
Generates a realistic image prompt based on user instructions and analyzed image description, then creates new images via Fal.ai API using the prompt and several reference images.

**Nodes Involved:**  
- Image Prompt (LangChain agent)  
- Create Image 1 (HTTP Request to Fal.ai)  
- Wait (Wait node)  
- Get the Image (HTTP Request)  
- Send a photo message (Telegram)

**Node Details:**

- **Image Prompt**  
  - Type: LangChain Agent (AI agent)  
  - Role: Creates a detailed prompt for image generation based on user caption and analyzed description.  
  - Configuration: System message guides prompt style to produce naturalistic, casual, UGC-style image prompts.  
  - Input: User caption from Telegram Trigger and description from Analyze image.  
  - Output: Text image prompt.  
  - Edge Cases: Ambiguous user instructions, AI hallucination, formatting errors.

- **Create Image 1**  
  - Type: HTTP Request  
  - Role: Sends prompt and image URLs to Fal.ai API to generate edited images.  
  - Configuration: POST to Fal.ai nano-banana endpoint with JSON body including prompt and multiple reference image URLs (including Telegram image). Uses Fal API key header.  
  - Input: Prompt from Image Prompt.  
  - Output: JSON with request ID for generated image.  
  - Edge Cases: API key invalid, request failure, malformed JSON.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow 30 seconds to allow Fal.ai image processing.  
  - Input: Triggered after Create Image 1.  
  - Output: Passes through to next node.  
  - Edge Cases: Insufficient wait time causing premature API fetch.

- **Get the Image**  
  - Type: HTTP Request  
  - Role: Polls Fal.ai API using request ID to retrieve generated image URLs.  
  - Configuration: GET request to Fal.ai nano-banana requests endpoint with request_id. Uses Fal API key.  
  - Input: Request ID from Create Image 1 response.  
  - Output: JSON with generated image URLs.  
  - Edge Cases: Request timeout, failure to generate image, API errors.

- **Send a photo message**  
  - Type: Telegram  
  - Role: Sends the first generated image back to the Telegram user as a photo message.  
  - Configuration: Uses Telegram API credentials, sends image URL to chat ID from Telegram Trigger.  
  - Edge Cases: Telegram API rate limits, invalid chat ID.

---

#### 1.4 Video Prompt Generation

**Overview:**  
Generates structured video scene prompts (3 scenes) in JSON format for UGC-style video creation, based on user caption and analyzed image description.

**Nodes Involved:**  
- Video Prompt (LangChain agent)  
- Structured Output Parser (JSON schema validation)  
- Split Out (Split array of scenes)

**Node Details:**

- **Video Prompt**  
  - Type: LangChain Agent  
  - Role: Produces three video scenes (Hook, Product, Call to Action) in strict JSON format with dialogue, camera, and style guidelines.  
  - Configuration: System message enforces UGC style, casual tone, scene structure, and formatting rules.  
  - Input: User caption, analyzed image description.  
  - Output: JSON object with scenes.  
  - Edge Cases: AI output not matching JSON schema, misformatted output.

- **Structured Output Parser**  
  - Type: Output Parser  
  - Role: Validates and auto-fixes AI-generated JSON output against defined schema for three scenes.  
  - Input: Output from Video Prompt.  
  - Output: Parsed structured JSON for further processing.  
  - Edge Cases: Parsing failures, invalid schema compliance.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits "output.scenes" object into individual scene items for parallel processing.  
  - Input: Parsed JSON from Structured Output Parser.  
  - Output: Separate items for each scene.  
  - Edge Cases: Missing scene keys, empty scenes.

---

#### 1.5 Video Creation and Processing

**Overview:**  
Generates individual video scenes via KIE.ai, waits for completion, retrieves status, aggregates scene URLs, merges videos via Fal.ai, waits, and fetches the final combined video.

**Nodes Involved:**  
- Make video 1 (HTTP Request to KIE.ai)  
- Wait1 (Wait node)  
- Get Record Info (HTTP Request)  
- Aggregate  
- Combine Video (HTTP Request to Fal.ai)  
- Wait2 (Wait node)  
- Get the final Video (HTTP Request)

**Node Details:**

- **Make video 1**  
  - Type: HTTP Request  
  - Role: Sends each video scene prompt to KIE.ai Veo API to generate a video segment.  
  - Configuration: POST with JSON body including prompt scenes, model "veo3_fast," aspect ratio 9:16, and image URL from generated image. Authorization Bearer token required.  
  - Input: Individual scene prompts from Split Out.  
  - Output: JSON with taskId for video generation task.  
  - Edge Cases: API key invalid, malformed prompt, API downtime.

- **Wait1**  
  - Type: Wait  
  - Role: Waits 100 seconds for KIE.ai video generation to complete.  
  - Edge Cases: Too short wait causes premature status check.

- **Get Record Info**  
  - Type: HTTP Request  
  - Role: Polls KIE.ai API for the status of video generation task using taskId.  
  - Configuration: GET with query parameter taskId, with authorization header. Retries on failure.  
  - Input: taskId from Make video 1.  
  - Output: Video result URLs if ready.  
  - Edge Cases: API rate limits, task not ready, invalid taskId.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Collects all video scene URLs from multiple task responses into a single list for merging.  
  - Input: Outputs from Get Record Info.  
  - Output: Array of video URLs.  
  - Edge Cases: Missing URLs, empty aggregation.

- **Combine Video**  
  - Type: HTTP Request  
  - Role: Sends video URLs to Fal.ai API to merge into a single video file.  
  - Configuration: POST with JSON body containing array of video URLs. Uses Fal API key.  
  - Edge Cases: API failure, invalid video URLs.

- **Wait2**  
  - Type: Wait  
  - Role: Waits 100 seconds for video merging process.  
  - Edge Cases: Insufficient wait time.

- **Get the final Video**  
  - Type: HTTP Request  
  - Role: Retrieves the merged video file URL from Fal.ai API using request_id.  
  - Output: Final video URL.  
  - Edge Cases: API error, request timeout.

---

#### 1.6 Delivery

**Overview:**  
Sends the final merged video back to the Telegram user.

**Nodes Involved:**  
- Send a video (Telegram)

**Node Details:**

- **Send a video**  
  - Type: Telegram  
  - Role: Sends the final video URL as a video message to the chat from which the original image was sent.  
  - Configuration: Uses Telegram API credentials, chat ID from Telegram Trigger.  
  - Edge Cases: Invalid video URL, Telegram API limits, invalid chat ID.

---

#### 1.7 Support and Setup Notes

**Overview:**  
Contains sticky notes for documentation, setup instructions, and author information.

**Nodes Involved:**  
- Sticky Note (Content Idea)  
- Sticky Note1 (Image Creation)  
- Sticky Note2 (Video Generation Prompt)  
- Sticky Note3 (Combine The video)  
- Sticky Note7 (Author Info)  
- Setup Instructions (Setup guide)

**Node Details:**

- Sticky notes provide context for workflow sections and setup instructions, including credential setup and API keys required.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                          | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                                           |
|------------------------|----------------------------------|----------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Sticky Note            | Sticky Note                      | Workflow section label                  |                             |                             | Content Idea                                                                                                          |
| Telegram Trigger       | Telegram Trigger                 | Trigger on Telegram image message      |                             | Botid 1                     |                                                                                                                       |
| Botid 1                | Set                             | Store Telegram bot token                | Telegram Trigger            | Image Pqath                 |                                                                                                                       |
| Image Pqath            | HTTP Request                    | Get Telegram image file path            | Botid 1                    | Analyze image               |                                                                                                                       |
| Analyze image          | OpenAI GPT-4 Vision             | Analyze image content                    | Image Pqath                | Image Prompt                |                                                                                                                       |
| Sticky Note1           | Sticky Note                      | Workflow section label                  |                             |                             | Image Creation                                                                                                        |
| Image Prompt           | LangChain Agent                 | Generate image prompt                    | Analyze image              | Create Image 1              |                                                                                                                       |
| Create Image 1         | HTTP Request                    | Generate images via Fal.ai API           | Image Prompt               | Wait                       |                                                                                                                       |
| Wait                   | Wait                           | Pause 30 seconds for image processing   | Create Image 1             | Get the Image               |                                                                                                                       |
| Get the Image          | HTTP Request                    | Retrieve generated image URLs            | Wait                       | Video Prompt, Send a photo message |                                                                                                                       |
| Send a photo message   | Telegram                       | Send generated image via Telegram       | Get the Image              |                             |                                                                                                                       |
| Sticky Note2           | Sticky Note                      | Workflow section label                  |                             |                             | Video Generation Prompt                                                                                               |
| Video Prompt           | LangChain Agent                 | Generate video scene prompts             | Get the Image              | Structured Output Parser    |                                                                                                                       |
| Structured Output Parser| Output Parser                   | Validate and parse video prompts JSON   | Video Prompt               | Video Prompt                |                                                                                                                       |
| Split Out              | Split Out                      | Split scenes for individual processing  | Video Prompt               | Make video 1                |                                                                                                                       |
| Make video 1           | HTTP Request                    | Request video generation from KIE.ai    | Split Out                  | Wait1                      |                                                                                                                       |
| Wait1                  | Wait                           | Pause 100 seconds for video generation  | Make video 1               | Get Record Info             |                                                                                                                       |
| Get Record Info        | HTTP Request                    | Poll video generation status             | Wait1                      | Aggregate                  |                                                                                                                       |
| Aggregate              | Aggregate                      | Collect video URLs from scenes           | Get Record Info            | Combine Video               |                                                                                                                       |
| Combine Video          | HTTP Request                    | Merge video scenes via Fal.ai API        | Aggregate                  | Wait2                      |                                                                                                                       |
| Wait2                  | Wait                           | Pause 100 seconds for video merging      | Combine Video              | Get the final Video         |                                                                                                                       |
| Get the final Video    | HTTP Request                    | Retrieve merged final video URL          | Wait2                      | Send a video                |                                                                                                                       |
| Send a video           | Telegram                       | Send final video via Telegram            | Get the final Video        |                             |                                                                                                                       |
| Sticky Note3           | Sticky Note                      | Workflow section label                  |                             |                             | Combine The video                                                                                                     |
| Sticky Note7           | Sticky Note                      | Author and contact information          |                             |                             | ## Muhammad Farooq Iqbal - Automation Expert & n8n Creator ... [LinkedIn](https://linkedin.com/in/muhammadfarooqiqbal) |
| Setup Instructions     | Sticky Note                      | Setup guide and credential instructions |                             |                             | **Setup Instructions for UGC Content Creator:** ...                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates  
   - Credentials: Add Telegram API credential with bot token  
   - Setup webhook to receive Telegram messages

2. **Create Set node ("Botid 1"):**  
   - Type: Set  
   - Assign variable "bot id" with your Telegram bot token string

3. **Create HTTP Request node ("Image Pqath"):**  
   - Type: HTTP Request (GET)  
   - URL: `https://api.telegram.org/bot{{ $json["bot id"] }}/getFile?file_id={{ $json.message.photo[0].file_id }}`  
   - No authentication needed (public Telegram API)

4. **Create OpenAI GPT-4 Vision node ("Analyze image"):**  
   - Type: OpenAI GPT-4 Vision node (LangChain)  
   - Model: chatgpt-4o-latest  
   - Operation: analyze with image URL `https://api.telegram.org/file/bot{{ $json["bot id"] }}/{{ $json.result.file_path }}`  
   - Prompt: Describe if image is character/product/both, output description only  
   - Credentials: OpenAI API key

5. **Create LangChain Agent node ("Image Prompt"):**  
   - Role: Generate image prompt from analyzed description and user caption  
   - System message set with detailed UGC style instructions  
   - Input: User caption and description from Analyze image  
   - Model: GPT-4 variant

6. **Create HTTP Request node ("Create Image 1"):**  
   - Type: HTTP POST  
   - URL: `https://queue.fal.run/fal-ai/nano-banana/edit`  
   - Body: JSON with prompt and array of image URLs (including Telegram image and reference images)  
   - Authentication: HTTP Header with Fal.ai API key

7. **Create Wait node ("Wait"):**  
   - Duration: 30 seconds (to allow image generation)

8. **Create HTTP Request node ("Get the Image"):**  
   - Type: HTTP GET  
   - URL: `https://queue.fal.run/fal-ai/nano-banana/requests/{{ $json.request_id }}`  
   - Authentication: Fal.ai API key

9. **Create Telegram node ("Send a photo message"):**  
   - Operation: sendPhoto  
   - File: first image URL from Get the Image output  
   - Chat ID: from Telegram Trigger message chat.id  
   - Credentials: Telegram API key

10. **Create LangChain Agent node ("Video Prompt"):**  
    - Role: Generate 3-scene JSON video prompt based on user caption and image analysis  
    - System message enforces UGC style and JSON format  
    - Output parser enabled for structured JSON

11. **Create Structured Output Parser node:**  
    - Schema: JSON object with "scenes" containing "scene 1", "scene 2", "scene 3" strings  
    - AutoFix enabled

12. **Create Split Out node:**  
    - Field to split: "output.scenes"  
    - Splits JSON scenes into separate items

13. **Create HTTP Request node ("Make video 1"):**  
    - Type: HTTP POST  
    - URL: `https://api.kie.ai/api/v1/veo/generate`  
    - Body: JSON with prompt (scene), model "veo3_fast", aspectRatio "9:16", image URL from Get the Image  
    - Headers: Authorization Bearer with KIE.ai API key

14. **Create Wait node ("Wait1"):**  
    - Duration: 100 seconds (wait for video generation)

15. **Create HTTP Request node ("Get Record Info"):**  
    - Type: HTTP GET  
    - URL: `https://api.kie.ai/api/v1/veo/record-info` with query param `taskId` from Make video 1 response  
    - Headers: Authorization Bearer with KIE.ai API key  
    - Enable retry on fail

16. **Create Aggregate node:**  
    - Aggregate field: `data.response.resultUrls[0]` to collect video URLs into an array

17. **Create HTTP Request node ("Combine Video"):**  
    - Type: HTTP POST  
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`  
    - Body: JSON with "video_urls" array from Aggregate output  
    - Authentication: Fal.ai API key via HTTP Header

18. **Create Wait node ("Wait2"):**  
    - Duration: 100 seconds (wait for video merging)

19. **Create HTTP Request node ("Get the final Video"):**  
    - Type: HTTP GET  
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`  
    - Authentication: Fal.ai API key

20. **Create Telegram node ("Send a video"):**  
    - Operation: sendVideo  
    - File: final video URL from Get the final Video node  
    - Chat ID: from Telegram Trigger message chat.id  
    - Credentials: Telegram API key

21. **Add sticky notes:**  
    - For content idea, image creation, video prompt, combine video, setup instructions, and author info as per the original.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| **Setup Instructions for UGC Content Creator:** Details on configuring Telegram API, OpenAI API, Fal.ai API, and KIE.ai API credentials. Instructions on creating Telegram bot, setting up webhook, API keys needed, testing, and customization options.                                                                                                                        | Found in "Setup Instructions" sticky note                                                                            |
| **Creator Contact:** Muhammad Farooq Iqbal - Automation Expert & n8n Creator. Email: mfarooqiqbal143@gmail.com, Phone: +923036991118, LinkedIn: https://linkedin.com/in/muhammadfarooqiqbal, Portfolio: https://mfarooqone.github.io/n8n/                                                                                                                                      | Found in "Sticky Note7"                                                                                              |
| The workflow leverages GPT-4 Vision and LangChain agents for intelligent image analysis and prompt creation, combining multiple AI services to automate realistic UGC video production.                                                                                                                                                                                         | General workflow description                                                                                         |
| Fal.ai API docs: https://fal.ai/ (for image generation and video merging endpoints)  
KIE.ai API docs: https://kie.ai/ (for Veo video generation)  
Telegram Bot API docs: https://core.telegram.org/bots/api                                                                                                                                                                                                                                                       | External API documentation references                                                                                 |

---

This comprehensive reference document enables in-depth understanding, reproduction, and troubleshooting of the presented n8n workflow for generating UGC videos from product images via Telegram.