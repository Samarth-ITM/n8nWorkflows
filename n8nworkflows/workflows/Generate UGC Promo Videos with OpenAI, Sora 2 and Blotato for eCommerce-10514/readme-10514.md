Generate UGC Promo Videos with OpenAI, Sora 2 and Blotato for eCommerce

https://n8nworkflows.xyz/workflows/generate-ugc-promo-videos-with-openai--sora-2-and-blotato-for-ecommerce-10514


# Generate UGC Promo Videos with OpenAI, Sora 2 and Blotato for eCommerce

### 1. Workflow Overview

This workflow automates the generation of User-Generated Content (UGC) promotional videos for eCommerce products by integrating Telegram, OpenAI, Sora 2 AI video generation, and Blotato social publishing platforms. It accepts product images and promotional text via Telegram, uses AI to analyze and script a short promo video, generates the video via Sora 2’s image-to-video API, and publishes the final video across multiple social media platforms through Blotato.

Logical blocks:

- **1.1 Telegram Input Reception:** Handles incoming Telegram messages containing product images and captions.
- **1.2 Image Processing and AI Analysis:** Downloads the image, uploads it publicly, and uses OpenAI’s vision AI to analyze product features.
- **1.3 Script Generation:** Uses OpenAI to generate a detailed 12-second UGC video script based on the analysis.
- **1.4 Video Generation with Sora 2:** Submits the script and image to Sora 2 API, polls for completion, downloads the video.
- **1.5 Telegram Delivery:** Sends the finished video back to the Telegram user.
- **1.6 Caption Generation:** Creates a catchy TikTok caption using OpenAI.
- **1.7 Multi-Platform Publishing via Blotato:** Uploads and publishes the video to TikTok, YouTube, LinkedIn, Facebook, Instagram, and Twitter (X).
- **1.8 Error Handling and Notifications:** Manages timeouts or failures by notifying users via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

- **Overview:** This block listens for incoming Telegram messages containing product images and promotional text, extracting necessary data.
- **Nodes Involved:** Telegram Trigger, Workflow Configuration, Extract Photo and text, Get Photo File from Telegram
- **Node Details:**

  - **Telegram Trigger**
    - *Type:* Trigger node for Telegram updates.
    - *Configuration:* Listens for "message" updates from a Telegram bot, authenticated with Telegram API credentials.
    - *Input/Output:* Starts the workflow on receiving a message.
    - *Edge cases:* Messages without images or captions may cause missing data downstream.
  
  - **Workflow Configuration**
    - *Type:* Set node to initialize workflow parameters (API keys, model, aspect ratio, duration).
    - *Configuration:* Stores static config values for reuse: maxPollingAttempts=20, falApiKey placeholder, model "sora-2", aspect_ratio "9:16", duration "12".
    - *Role:* Provides centralized configuration.
  
  - **Extract Photo and text**
    - *Type:* Set node extracting from the Telegram message: highest resolution photo file ID and promotion text (caption or message text).
    - *Key expressions:* Extracts photo file ID from the last photo in the array; promotionText from caption or text.
    - *Edge cases:* Missing photo or caption/text can cause nulls.
  
  - **Get Photo File from Telegram**
    - *Type:* Telegram node to download the photo.
    - *Configuration:* Uses the extracted photo file ID, authenticated with Telegram credentials.
    - *Role:* Retrieves the actual image file for processing.
    - *Edge cases:* File download failure or invalid file ID.

#### 1.2 Image Processing and AI Analysis

- **Overview:** Uploads the image to a public hosting service and uses OpenAI's vision API to analyze product features, audience, and style.
- **Nodes Involved:** Build Public Image URL, Analyze Product Image (Vision API)
- **Node Details:**

  - **Build Public Image URL**
    - *Type:* HTTP Request node.
    - *Configuration:* POSTs the image file to tmpfiles.org for public URL generation.
    - *Role:* Makes image accessible publicly for Sora 2 API.
    - *Edge cases:* Upload failure, invalid file formats.
  
  - **Analyze Product Image (Vision API)**
    - *Type:* OpenAI Vision API node (GPT-4o-mini model).
    - *Configuration:* Inputs base64 image, prompts for product name, category, features, audience, visual style, and promotion objective.
    - *Role:* Extracts structured product insights for scriptwriting.
    - *Edge cases:* API rate limits, vision model errors, incomplete analysis.

#### 1.3 Script Generation

- **Overview:** Generates a structured 12-second UGC video script with frames, dialogue, camera directions, captions, and hashtags using OpenAI.
- **Nodes Involved:** Generate UGC Script (OpenAI), Parse Script Response, Merge
- **Node Details:**

  - **Generate UGC Script (OpenAI)**
    - *Type:* OpenAI chat completion node.
    - *Configuration:* Uses GPT-4o-mini with temperature 0.8; system prompt instructs to create frame-by-frame video script JSON.
    - *Input:* Product analysis text and promotion objective.
    - *Output:* JSON with frames array and hashtags.
    - *Edge cases:* Model output parsing errors.
  
  - **Parse Script Response**
    - *Type:* Code node.
    - *Function:* Extracts script JSON, frames, hashtags, and chatId from OpenAI response.
    - *Edge cases:* Missing or malformed JSON response.
  
  - **Merge**
    - *Type:* Merge node.
    - *Role:* Joins the script data with the uploaded image URL for downstream processing.

#### 1.4 Video Generation with Sora 2

- **Overview:** Sends the image and script to Sora 2 API to generate the video, polls for completion, downloads the final video.
- **Nodes Involved:** Submit to Sora 2 API, Extract Video Job ID, Wait 15 Seconds, Check Video Status, Parse Status Response, Check If Complete, Download Video File
- **Node Details:**

  - **Submit to Sora 2 API**
    - *Type:* HTTP Request.
    - *Configuration:* POST to Sora 2 image-to-video endpoint with prompt (script), image URL, aspect ratio, duration; authorization header with falApiKey.
    - *Output:* Returns request_id for video job.
    - *Edge cases:* API authentication errors, invalid payload.
  
  - **Extract Video Job ID**
    - *Type:* Code node.
    - *Function:* Extracts request_id and chatId; initializes pollingAttempt=0.
    - *Edge cases:* Missing request_id in response.
  
  - **Wait 15 Seconds**
    - *Type:* Wait node.
    - *Configuration:* Waits for 15 seconds between status checks.
  
  - **Check Video Status**
    - *Type:* HTTP Request.
    - *Configuration:* GET request to Sora 2 status endpoint with request_id, authorized via falApiKey.
    - *Output:* Status and video URL if ready.
  
  - **Parse Status Response**
    - *Type:* Code node.
    - *Function:* Parses status, increments pollingAttempt.
    - *Edge cases:* Missing previous poll data or malformed API response.
  
  - **Check If Complete**
    - *Type:* If node.
    - *Conditions:* Checks if status is "COMPLETED" or "succeeded" or pollingAttempt >= maxPollingAttempts.
    - *Branches:* 
      - True: Proceed to download video.
      - False: Wait and poll again.
      - Timeout: Send error message.
  
  - **Download Video File**
    - *Type:* HTTP Request.
    - *Configuration:* Downloads video metadata and URL from Sora 2 API using request_id.
  
#### 1.5 Telegram Delivery

- **Overview:** Sends the generated video back to the user on Telegram.
- **Nodes Involved:** Send Video to Telegram
- **Node Details:**

  - **Send Video to Telegram**
    - *Type:* Telegram node.
    - *Configuration:* Sends the video file URL to the user's chatId with caption including video URL.
    - *Edge cases:* Telegram upload failure, invalid chatId.

#### 1.6 Caption Generation

- **Overview:** Generates an engaging TikTok caption using OpenAI, incorporating trending hashtags from the script.
- **Nodes Involved:** Generate Caption with GPT-4
- **Node Details:**

  - **Generate Caption with GPT-4**
    - *Type:* OpenAI chat completion node.
    - *Configuration:* Uses GPT-4o-mini; prompt requests catchy hook, 5-8 relevant hashtags, concise text optimized for TikTok.
    - *Input:* Telegram message caption and hashtags from the generated script.
    - *Output:* Caption text only.
    - *Edge cases:* Model output formatting issues.

#### 1.7 Multi-Platform Publishing via Blotato

- **Overview:** Uploads the video to Blotato and publishes it to various social platforms (TikTok, YouTube, LinkedIn, Facebook, Instagram, Twitter).
- **Nodes Involved:** Upload Video to BLOTATO, Tiktok, Youtube, Linkedin, Facebook, Instagram, Twitter (X), Merge1, Send a text message
- **Node Details:**

  - **Upload Video to BLOTATO**
    - *Type:* Blotato node.
    - *Configuration:* Uploads video URL to Blotato media resource.
  
  - **Tiktok, Youtube, Linkedin, Facebook, Instagram, Twitter (X)**
    - *Type:* Blotato nodes for publishing.
    - *Configuration:* Each node configured for respective platform with account IDs, post text (caption), media URLs.
    - *Privacy:* YouTube post set to private with no subscriber notification.
    - *Edge cases:* API authentication failures, publishing errors.
  
  - **Merge1**
    - *Type:* Merge node.
    - *Role:* Combines outputs of all platform publishing nodes.
  
  - **Send a text message**
    - *Type:* Telegram node.
    - *Function:* Notifies user that the video is published.

#### 1.8 Error Handling and Notifications

- **Overview:** Handles timeout or failure in video generation by informing the Telegram user.
- **Nodes Involved:** Send Error Message
- **Node Details:**

  - **Send Error Message**
    - *Type:* Telegram node.
    - *Configuration:* Sends a predefined error message to the user's chat if video generation fails or times out.
    - *Edge cases:* Telegram API failure.

---

### 3. Summary Table

| Node Name                   | Node Type                            | Functional Role                      | Input Node(s)              | Output Node(s)                        | Sticky Note                                                                                                  |
|-----------------------------|------------------------------------|------------------------------------|----------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                   | Receives Telegram messages          | -                          | Workflow Configuration               | # 📱 STEP 1: TELEGRAM BOT SETUP (includes setup instructions and credential info)                            |
| Workflow Configuration      | Set                               | Stores API keys and settings        | Telegram Trigger            | Extract Photo and text               | # ⚙️ STEP 2: API KEYS CONFIGURATION (details FAL.ai, OpenAI keys, video params)                              |
| Extract Photo and text      | Set                               | Extracts photo file ID and text     | Workflow Configuration      | Get Photo File from Telegram         | # 🤖 STEP 3: AI ANALYSIS & SCRIPT GENERATION (explains photo and text extraction)                            |
| Get Photo File from Telegram | Telegram                          | Downloads image file from Telegram  | Extract Photo and text      | Analyze Product Image, Build Public Image URL |                                                                                                              |
| Build Public Image URL      | HTTP Request                      | Uploads image for public access     | Get Photo File from Telegram| Merge                              |                                                                                                              |
| Analyze Product Image (Vision API) | OpenAI Vision API           | AI analyzes product image           | Get Photo File from Telegram| Generate UGC Script (OpenAI)         |                                                                                                              |
| Generate UGC Script (OpenAI)| OpenAI Chat Completion            | Generates detailed video script     | Analyze Product Image       | Parse Script Response                |                                                                                                              |
| Parse Script Response       | Code                             | Extracts and formats script data    | Generate UGC Script         | Merge                              |                                                                                                              |
| Merge                      | Merge                            | Combines script and image URL       | Build Public Image URL, Parse Script Response| Submit to Sora 2 API      |                                                                                                              |
| Submit to Sora 2 API        | HTTP Request                    | Sends image + script to Sora 2      | Merge                      | Extract Video Job ID                 | # 🎥 STEP 4: VIDEO GENERATION WITH SORA 2 (explains video generation process and polling)                   |
| Extract Video Job ID        | Code                             | Extracts job ID and initializes polling | Submit to Sora 2 API        | Wait 15 Seconds                    |                                                                                                              |
| Wait 15 Seconds            | Wait                             | Waits between polling attempts      | Extract Video Job ID        | Check Video Status                  |                                                                                                              |
| Check Video Status          | HTTP Request                    | Checks video job status             | Wait 15 Seconds             | Parse Status Response               |                                                                                                              |
| Parse Status Response       | Code                             | Parses status and increments attempts | Check Video Status          | Check If Complete                   |                                                                                                              |
| Check If Complete           | If                               | Determines if video is ready or timeout | Parse Status Response       | Download Video File, Wait 15 Seconds + Send Error Message |                                                                                                              |
| Download Video File         | HTTP Request                    | Downloads completed video file      | Check If Complete (true)    | Send Video to Telegram              |                                                                                                              |
| Send Video to Telegram      | Telegram                         | Sends video file to user on Telegram | Download Video File         | Generate Caption with GPT-4          |                                                                                                              |
| Generate Caption with GPT-4 | OpenAI Chat Completion          | Creates TikTok caption for video    | Send Video to Telegram      | Upload Video to BLOTATO             |                                                                                                              |
| Upload Video to BLOTATO     | Blotato                         | Uploads video to Blotato platform   | Generate Caption with GPT-4 | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube | # 📤 STEP 5: PUBLISHING & TRACKING (detailed install & usage steps for Blotato community node)             |
| Tiktok                     | Blotato                         | Publishes video to TikTok           | Upload Video to BLOTATO     | Merge1                             |                                                                                                              |
| Linkedin                   | Blotato                         | Publishes video to LinkedIn         | Upload Video to BLOTATO     | Merge1                             |                                                                                                              |
| Facebook                   | Blotato                         | Publishes video to Facebook         | Upload Video to BLOTATO     | Merge1                             |                                                                                                              |
| Instagram                  | Blotato                         | Publishes video to Instagram        | Upload Video to BLOTATO     | Merge1                             |                                                                                                              |
| Twitter (X)                | Blotato                         | Publishes video to Twitter (X)      | Upload Video to BLOTATO     | Merge1                             |                                                                                                              |
| Youtube                    | Blotato                         | Publishes video to YouTube (private) | Upload Video to BLOTATO     | Merge1                             |                                                                                                              |
| Merge1                     | Merge                           | Combines multi-platform publish outputs | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube | Send a text message               |                                                                                                              |
| Send a text message        | Telegram                        | Notifies user of successful publishing | Merge1                     | -                                  |                                                                                                              |
| Send Error Message         | Telegram                        | Notifies user on video generation failure | Check If Complete (false branch) | -                                  |                                                                                                              |
| Workflow Overview          | Sticky Note                    | Provides user overview and video tutorial link | -                          | -                                  | Contains video tutorial link and author contact information                                                 |
| Step 1 - Telegram Setup    | Sticky Note                    | Guides Telegram bot setup and testing | -                          | -                                  | Detailed Telegram bot setup guide                                                                            |
| Step 2 - Configuration    | Sticky Note                    | Guides API key and workflow configuration | -                          | -                                  | Detailed API key and parameter setup instructions                                                            |
| Step 3 - AI Analysis       | Sticky Note                    | Explains AI analysis and script generation | -                          | -                                  | Describes AI logic and nodes                                                                                   |
| Step 4 - Video Generation  | Sticky Note                    | Describes video generation and polling process | -                          | -                                  | Explains Sora 2 API integration and polling mechanism                                                         |
| Step 5 - Publishing        | Sticky Note                    | Details Blotato node installation and usage | -                          | -                                  | Includes link to Blotato and publishing steps                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials:**
   - Use Telegram @BotFather to create a bot.
   - Obtain API token.
   - In n8n, create Telegram API credentials with this token.

2. **Add Telegram Trigger node:**
   - Configure to listen for "message" updates.
   - Use created Telegram credentials.

3. **Add Set node "Workflow Configuration":**
   - Define parameters:
     - maxPollingAttempts: 20
     - falApiKey: Your FAL.ai API key (string)
     - Model: "sora-2"
     - aspect_ratio: "9:16"
     - duration: "12"
   - Include other fields.

4. **Add Set node "Extract Photo and text":**
   - Extract from Telegram message:
     - photoUrl: last photo’s file_id if present.
     - promotionText: caption or text.

5. **Add Telegram node "Get Photo File from Telegram":**
   - Resource: file
   - Use photoUrl from previous node.
   - Use Telegram credentials.

6. **Add HTTP Request node "Build Public Image URL":**
   - POST to https://tmpfiles.org/api/v1/upload
   - Body: multipart-form-data with file binary data from previous node.
   - Response format: JSON.

7. **Add OpenAI node "Analyze Product Image (Vision API)":**
   - Resource: image; Input type: base64.
   - Model: gpt-4o-mini.
   - Prompt: ask to analyze product name, category, features, audience, colors, style, packaging.
   - Include promotionText in prompt.
   - Use OpenAI API credentials.

8. **Add OpenAI node "Generate UGC Script (OpenAI)":**
   - Model: gpt-4o-mini.
   - Temperature: 0.8.
   - System prompt: instruct to generate 12s video script JSON with frames, dialogue, camera, caption, hashtags.
   - User prompt: includes product analysis and promotion objective.
   - Output: JSON.

9. **Add Code node "Parse Script Response":**
   - Extract script JSON, frames array, hashtags, and chatId from inputs.

10. **Add Merge node:**
    - Merge outputs of Build Public Image URL and Parse Script Response.

11. **Add HTTP Request node "Submit to Sora 2 API":**
    - POST to https://queue.fal.run/fal-ai/sora-2/image-to-video.
    - Body JSON includes:
      - prompt: script text (cleaned, no newlines).
      - resolution: "auto"
      - aspect_ratio from config.
      - duration from config.
      - image_url from uploaded image URL.
    - Headers: Authorization with "Key " + falApiKey, Content-Type application/json.

12. **Add Code node "Extract Video Job ID":**
    - Extract request_id and chatId; initialize pollingAttempt=0.

13. **Add Wait node "Wait 15 Seconds":**
    - Wait 15 seconds between polls.

14. **Add HTTP Request node "Check Video Status":**
    - GET https://queue.fal.run/fal-ai/sora-2/requests/{requestId}/status.
    - Authorization header with falApiKey.

15. **Add Code node "Parse Status Response":**
    - Extracts status, video URL (if any), increments pollingAttempt.

16. **Add If node "Check If Complete":**
    - Conditions:
      - status == "COMPLETED" or "succeeded"
      - OR pollingAttempt >= maxPollingAttempts.
    - True branch → Download video.
    - False branch → Wait 15 seconds and check again.
    - Timeout branch → send error message.

17. **Add HTTP Request node "Download Video File":**
    - GET https://queue.fal.run/fal-ai/sora-2/requests/{requestId}.
    - Authorization header.

18. **Add Telegram node "Send Video to Telegram":**
    - Operation: sendVideo.
    - File: video URL from downloaded data.
    - ChatId: from Telegram Trigger.
    - Caption: include video URL.
    - Use Telegram credentials.

19. **Add OpenAI node "Generate Caption with GPT-4":**
    - Model: gpt-4o-mini.
    - Prompt: generate catchy TikTok caption using Telegram message caption and trending hashtags extracted from script.
    - Output: text only.

20. **Add Blotato node "Upload Video to BLOTATO":**
    - mediaUrl: video URL.
    - Use Blotato API credentials.

21. **Add Blotato publishing nodes for TikTok, YouTube (private), LinkedIn, Facebook, Instagram, Twitter:**
    - Configure each with respective platform, account IDs.
    - postContentText: caption from OpenAI caption node.
    - postContentMediaUrls: video URL from upload node.
    - YouTube: privacy set to private, no subscriber notifications.

22. **Add Merge node "Merge1":**
    - Combine outputs of all platform publishing nodes.

23. **Add Telegram node "Send a text message":**
    - Text: "Published"
    - ChatId: from Telegram Trigger.
    - Use Telegram credentials.

24. **Add Telegram node "Send Error Message":**
    - Text: "Video generation timed out or failed. Please try again with a different product image or description."
    - ChatId: from Telegram Trigger.
    - Used on timeout or failure branch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Workflow includes detailed stepwise sticky notes guiding Telegram bot setup, API key configuration, AI analysis, video generation, and publishing steps.                                                                                                                  | Within workflow, sticky notes nodes titled "Step 1 - Telegram Setup", "Step 2 - Configuration", etc.                      |
| Video tutorial on workflow usage: [YouTube Video](https://youtu.be/SZMWXW8Vk8E)                                                                                                                                                                                          | Workflow Overview sticky note                                                                                             |
| Contact and support links: LinkedIn [https://www.linkedin.com/in/dr-firas/](https://www.linkedin.com/in/dr-firas/), YouTube [https://www.youtube.com/@DRFIRASS](https://www.youtube.com/@DRFIRASS), n8n training [https://hotm.art/formation-n8n](https://hotm.art/formation-n8n) | Workflow Overview sticky note                                                                                             |
| Blotato community node installation and API key setup instructions: https://blotato.com/?ref=firas                                                                                                                                                                        | Step 5 - Publishing sticky note                                                                                           |
| Requires n8n community node installation: `@blotato/n8n-nodes-blotato` for multi-platform publishing integration.                                                                                                                                                        | Step 5 - Publishing sticky note                                                                                           |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.