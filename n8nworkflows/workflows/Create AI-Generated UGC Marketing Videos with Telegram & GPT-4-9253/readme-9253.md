Create AI-Generated UGC Marketing Videos with Telegram & GPT-4

https://n8nworkflows.xyz/workflows/create-ai-generated-ugc-marketing-videos-with-telegram---gpt-4-9253


# Create AI-Generated UGC Marketing Videos with Telegram & GPT-4

### 1. Workflow Overview

This workflow automates the creation of AI-generated User-Generated Content (UGC) marketing videos using Telegram as the input interface and GPT-4 based AI agents for content generation. It is designed to receive photos with captions via Telegram, analyze the images, generate AI prompts for both images and video clips, create the media assets using AI services, combine multiple video clips into a single video, and send the final UGC-style video back to the Telegram user.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Image Retrieval:** Receives user input via Telegram (photo + caption), fetches the image file from Telegram servers.
- **1.2 Image Analysis:** Uses OpenAI to analyze the image for product and/or character details.
- **1.3 AI Prompt Generation:** Two AI agents generate structured prompts: one for image generation and one for video clip generation.
- **1.4 Image Generation and Retrieval:** Generates a stylized image based on AI prompts, then polls for completion.
- **1.5 Video Clip Generation:** Splits the video prompt into scenes, generates each video clip via an external API, and waits for completion.
- **1.6 Video Clips Aggregation and Merging:** Aggregates the generated clips and merges them into a single video file using an FFmpeg API.
- **1.7 Final Video Retrieval and Delivery:** Retrieves the final merged video and sends it back to the Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Image Retrieval

**Overview:**  
Captures incoming messages from Telegram, extracts the photo file, and retrieves the full image file path from Telegram servers.

**Nodes Involved:**  
- Telegram Trigger  
- Bot ID  
- Get Img Path

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point listening for incoming Telegram messages containing photos and captions.  
  - Config: Listens for "message" update type.  
  - Inputs: External Telegram messages.  
  - Outputs: Passes on message JSON including the photo object.  
  - Edge Cases: No photo or unsupported message format may cause downstream failures.

- **Bot ID**  
  - Type: Set Node  
  - Role: Stores Telegram bot ID for API calls.  
  - Config: Sets a string field "bot id" (empty by default, to be configured).  
  - Inputs: From Telegram Trigger.  
  - Outputs: Supplies bot ID for HTTP requests.  
  - Edge Cases: Missing or incorrect bot ID will cause Telegram API calls to fail.

- **Get Img Path**  
  - Type: HTTP Request  
  - Role: Retrieves the file path of the highest resolution photo from Telegram API.  
  - Config: HTTP GET to `https://api.telegram.org/bot{{bot_id}}/getFile?file_id={{file_id}}`  
  - Inputs: Uses bot ID and photo file_id from Telegram Trigger.  
  - Outputs: Returns JSON with image file path.  
  - Edge Cases: Telegram API errors, invalid file_id, or network issues.

---

#### 2.2 Image Analysis

**Overview:**  
Analyzes the retrieved image using OpenAI to determine if it depicts a product, a character, or both, extracting key attributes for prompt generation.

**Nodes Involved:**  
- Describe Img

**Node Details:**  

- **Describe Img**  
  - Type: LangChain OpenAI Node (Image Analysis)  
  - Role: AI-based image content analysis using OpenAI GPT-4 vision capabilities.  
  - Config: Uses a YAML-format prompt requesting detailed attributes (brand, colors, font style for products; character name, outfit, colors for characters).  
  - Input: Image URL constructed from Telegram file path.  
  - Output: YAML text describing image content.  
  - Edge Cases: Ambiguous images, incomplete analysis, or API timeouts.

---

#### 2.3 AI Prompt Generation

**Overview:**  
Generates AI prompts for both the image and video clip creation agents based on user input and image analysis.

**Nodes Involved:**  
- In Progress  
- UGCRobo - Image AI Agent  
- Structured Output 2  
- Create Image  
- Wait 3  
- Get Image  
- UGCRobo - Video AI Agent  
- Structured Output 1  
- Split Out  
- Create Video  
- Wait 2  
- Get Video  
- Think

**Node Details:**  

- **In Progress**  
  - Type: Telegram Node  
  - Role: Sends an acknowledgment message to the user that video creation has started.  
  - Config: Text "Got it! I'm now creating your video..." sent to the chat id from Telegram Trigger.  
  - Input: From Describe Img node.  
  - Output: Message sent to user.  
  - Edge Cases: Telegram API failure.

- **UGCRobo - Image AI Agent**  
  - Type: LangChain Agent  
  - Role: Generates a UGC-style image prompt based on user instructions and image analysis.  
  - Config: System prompt enforces natural, candid, amateur iPhone photo style with text accuracy.  
  - Input: User caption from Telegram + image description YAML.  
  - Output: Structured image prompt in YAML format.  
  - Edge Cases: Prompt generation errors, incomplete outputs.

- **Structured Output 2**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the image AI agent output into structured JSON with `image_prompt` and `aspect_ratio_image`.  
  - Config: Auto-fix enabled, expects specific JSON schema.  
  - Input: Text output from Image AI Agent.  
  - Output: Parsed JSON for downstream usage.  
  - Edge Cases: Parsing failures if AI output deviates.

- **Create Image**  
  - Type: HTTP Request  
  - Role: Calls Kie.ai API to generate an image from the AI prompt.  
  - Config: POST with JSON body containing prompt, aspect ratio, and Telegram image URL.  
  - Input: Structured output from Structured Output 2 and Telegram image URL.  
  - Output: Returns task ID for image generation.  
  - Edge Cases: API errors, invalid prompt, authentication failure.

- **Wait 3**  
  - Type: Wait  
  - Role: Waits 60 seconds to allow image generation to complete.  
  - Input: From Create Image.  
  - Output: Triggers Get Image after delay.

- **Get Image**  
  - Type: HTTP Request  
  - Role: Polls Kie.ai API for image generation status and retrieves the generated image URLs.  
  - Config: GET with query param taskId from Create Image response.  
  - Output: Contains URLs of generated images.  
  - Edge Cases: Polling timeout, API errors.

- **UGCRobo - Video AI Agent**  
  - Type: LangChain Agent  
  - Role: Generates UGC-style video prompts (scenes) including dialogue, camera style, and model choice based on user input and image description.  
  - Config: Complex system prompt enforcing conversational, casual realism, and calculating number of clips based on requested video length.  
  - Input: Telegram caption and Describe Img output.  
  - Output: Structured video scenes array.  
  - Edge Cases: Prompt generation errors, mismatch in scene count.

- **Structured Output 1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses video AI agent output into structured JSON array with video prompts and metadata.  
  - Config: Auto-fix enabled, expects scene array schema.  
  - Input: Text output from Video AI Agent.  
  - Output: Parsed scene objects.  
  - Edge Cases: Parsing errors.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of scenes into individual items for processing each video clip separately.  
  - Input: Scenes array from Structured Output 1.  
  - Output: Individual scene objects.  
  - Edge Cases: Empty or malformed scenes array.

- **Create Video**  
  - Type: HTTP Request  
  - Role: Calls Kie.ai API to generate video clips from each scene prompt, including the generated image URL.  
  - Config: POST with video prompt, model, aspect ratio, and image URL from Get Image node.  
  - Input: Each scene item from Split Out + image URL.  
  - Output: Returns taskId for each video clip generation.  
  - Edge Cases: API failure, invalid prompt.

- **Wait 2**  
  - Type: Wait  
  - Role: Waits 150 seconds to allow video clip generation to complete.  
  - Input: From Create Video.  
  - Output: Triggers Get Video node.

- **Get Video**  
  - Type: HTTP Request  
  - Role: Polls Kie.ai API for each video clip generation status and obtains the clip URLs.  
  - Config: GET with taskId query parameter from Create Video response.  
  - Output: Video clip URLs.  
  - Edge Cases: Polling timeout, API errors.

- **Think**  
  - Type: LangChain Think Tool  
  - Role: Double-checks AI outputs for completeness and style adherence.  
  - Input/Output: Connected to both Image and Video AI Agents as a tool to verify outputs.  
  - Edge Cases: None direct; used for internal validation.

---

#### 2.4 Video Clips Aggregation and Merging

**Overview:**  
Aggregates all generated video clip URLs and sends them to an external FFmpeg service to merge them into a single continuous video.

**Nodes Involved:**  
- Aggregate  
- Combine Clips  
- Wait  
- Get Final Video  
- If 2  
- Send Video

**Node Details:**  

- **Aggregate**  
  - Type: Aggregate  
  - Role: Collects all generated video URLs from multiple video clips into an array.  
  - Input: From Get Video node outputs.  
  - Output: Array of URLs.  
  - Edge Cases: Missing URLs, incomplete aggregation.

- **Combine Clips**  
  - Type: HTTP Request  
  - Role: Sends array of video URLs to fal.ai FFmpeg API to merge clips.  
  - Config: POST JSON with `video_urls` array.  
  - Input: From Aggregate.  
  - Output: Returns merged video URL response.  
  - Edge Cases: API errors, failed merging.

- **Wait**  
  - Type: Wait  
  - Role: Waits 100 seconds to allow video merging to complete.  
  - Input: From Combine Clips.  
  - Output: Triggers Get Final Video node.

- **Get Final Video**  
  - Type: HTTP Request  
  - Role: Retrieves the final merged video URL from the FFmpeg service.  
  - Config: GET request to URL returned from Combine Clips.  
  - Output: JSON with final video URL.  
  - Edge Cases: Resource not ready, API failure.

- **If 2**  
  - Type: If  
  - Role: Checks if final video URL is valid (non-empty).  
  - Input: From Get Final Video.  
  - Outputs:  
    - True branch: Send Video  
    - False branch: Wait (retry or error handling)  
  - Edge Cases: Empty or invalid URL.

- **Send Video**  
  - Type: Telegram Node  
  - Role: Sends the final merged video back to the Telegram user chat as a video message.  
  - Config: Uses Telegram API to send video file URL to original chat ID.  
  - Edge Cases: Telegram API failure, video file inaccessible.

---

#### 2.5 Auxiliary and Informational Nodes

- **Sticky Notes:** Several sticky notes provide user instructions, step descriptions, and system overview. They do not affect execution but provide documentation and setup guidance.

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                        | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                          |
|-------------------------|-----------------------------------|-------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger         | Telegram Trigger                  | Entry point: receive Telegram input | -                           | Bot ID                     |                                                                                                |
| Bot ID                  | Set                              | Store Telegram Bot ID                 | Telegram Trigger            | Get Img Path               |                                                                                                |
| Get Img Path            | HTTP Request                     | Retrieve Telegram image file path    | Bot ID                      | Describe Img               |                                                                                                |
| Describe Img            | LangChain OpenAI (Image Analysis)| Analyze image content                 | Get Img Path                | In Progress                |                                                                                                |
| In Progress             | Telegram                         | Notify user video creation started   | Describe Img                | UGCRobo - Image AI Agent   |                                                                                                |
| UGCRobo - Image AI Agent| LangChain Agent                  | Generate image prompt                 | In Progress                 | Structured Output 2        |                                                                                                |
| Structured Output 2     | LangChain Output Parser          | Parse image prompt JSON               | UGCRobo - Image AI Agent    | Create Image               |                                                                                                |
| Create Image            | HTTP Request                    | Call AI image generation API          | Structured Output 2         | Wait 3                     |                                                                                                |
| Wait 3                  | Wait                            | Wait for image generation             | Create Image                | Get Image                  |                                                                                                |
| Get Image               | HTTP Request                    | Poll for generated image URLs         | Wait 3                     | UGCRobo - Video AI Agent, If |                                                                                                |
| UGCRobo - Video AI Agent| LangChain Agent                  | Generate video scene prompts          | Get Image, Telegram Trigger | Structured Output 1        |                                                                                                |
| Structured Output 1     | LangChain Output Parser          | Parse video scene JSON                 | UGCRobo - Video AI Agent    | Split Out                  |                                                                                                |
| Split Out               | Split Out                       | Split scenes array into individual clips | Structured Output 1         | Create Video               |                                                                                                |
| Create Video            | HTTP Request                    | Call AI video clip generation API      | Split Out                  | Wait 2                     |                                                                                                |
| Wait 2                  | Wait                            | Wait for video clip generation         | Create Video                | Get Video                  |                                                                                                |
| Get Video               | HTTP Request                    | Poll for generated video clip URLs     | Wait 2                     | Aggregate                  |                                                                                                |
| Aggregate               | Aggregate                       | Collect all video URLs                 | Get Video                   | Combine Clips              |                                                                                                |
| Combine Clips           | HTTP Request                    | Merge multiple clips into one video    | Aggregate                   | Wait                       |                                                                                                |
| Wait                    | Wait                            | Wait for video merge completion        | Combine Clips               | Get Final Video            |                                                                                                |
| Get Final Video          | HTTP Request                    | Retrieve final merged video URL        | Wait                       | If 2                       |                                                                                                |
| If 2                    | If                              | Check final video URL validity          | Get Final Video             | Send Video / Wait          |                                                                                                |
| Send Video              | Telegram                         | Send final video to Telegram chat       | If 2 (true branch)          | -                          |                                                                                                |
| Think                   | LangChain Think Tool            | Double-check AI outputs                 | UGCRobo - Image AI Agent, UGCRobo - Video AI Agent | UGCRobo - Image AI Agent, UGCRobo - Video AI Agent |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API credentials.  
   - Set to listen for "message" updates, specifically photos and captions.

2. **Create Set Node "Bot ID":**  
   - Type: Set  
   - Create a string field "bot id" and set it to your Telegram Bot token string.

3. **Create HTTP Request Node "Get Img Path":**  
   - Method: GET  
   - URL: `https://api.telegram.org/bot{{ $json["bot id"] }}/getFile?file_id={{ $json.message.photo[2].file_id }}`  
   - Use credentials if required.  
   - Input from "Bot ID".

4. **Create LangChain OpenAI Node "Describe Img":**  
   - Use OpenAI GPT-4 model with image analysis capability.  
   - Prompt: Analyze image to detect product/character with detailed YAML output as per workflow description.  
   - Image URL constructed from Telegram file path: `https://api.telegram.org/file/bot{{ $json["bot id"] }}/{{ $json.result.file_path }}`  
   - Input from "Get Img Path".

5. **Create Telegram Node "In Progress":**  
   - Type: Telegram  
   - Send text "Got it! I'm now creating your video..." to `{{ $json.message.chat.id }}`.  
   - Input from "Describe Img".

6. **Create LangChain Agent Node "UGCRobo - Image AI Agent":**  
   - System prompt enforcing natural, casual UGC-style image prompts with accurate text replication.  
   - Input: User instructions and Describe Img output.  
   - Connect "Think" tool for output validation.

7. **Create LangChain Output Parser Node "Structured Output 2":**  
   - Set JSON schema expecting `image_prompt` and `aspect_ratio_image`.  
   - Input from "UGCRobo - Image AI Agent".

8. **Create HTTP Request Node "Create Image":**  
   - POST to `https://api.kie.ai/api/v1/gpt4o-image/generate`  
   - JSON body includes:  
     - `filesUrl`: Telegram image URL  
     - `prompt`: from structured output image_prompt (escaped)  
     - `size`: from structured output aspect_ratio_image  
     - `nVariants`: 1  
   - Authentication with Kie.ai HTTP header credentials.  
   - Input from "Structured Output 2".

9. **Create Wait Node "Wait 3":**  
   - Wait 60 seconds.  
   - Input from "Create Image".

10. **Create HTTP Request Node "Get Image":**  
    - GET to `https://api.kie.ai/api/v1/gpt4o-image/record-info` with query param `taskId` from Create Image response.  
    - Authentication with Kie.ai creds.  
    - Input from "Wait 3".

11. **Create LangChain Agent Node "UGCRobo - Video AI Agent":**  
    - System prompt for generating multiple UGC video scenes with dialogue, style, and clip count calculation.  
    - Input: User instructions, Describe Img output, and Think tool validation.  
    - Input from "Get Image".

12. **Create LangChain Output Parser Node "Structured Output 1":**  
    - JSON schema expecting array of scenes with `video_prompt`, `aspect_ratio_video`, and `model`.  
    - Input from "UGCRobo - Video AI Agent".

13. **Create Split Out Node "Split Out":**  
    - Field to split: `output.scenes` (array of scenes).  
    - Input from "Structured Output 1".

14. **Create HTTP Request Node "Create Video":**  
    - POST to `https://api.kie.ai/api/v1/veo/generate`  
    - JSON body includes:  
      - `prompt`: each scene's `video_prompt` (escaped)  
      - `model`: scene model  
      - `aspectRatio`: scene aspect ratio  
      - `imageUrls`: first image URL from Get Image node  
    - Authentication with Kie.ai credentials.  
    - Input from "Split Out".

15. **Create Wait Node "Wait 2":**  
    - Wait 150 seconds.  
    - Input from "Create Video".

16. **Create HTTP Request Node "Get Video":**  
    - GET to `https://api.kie.ai/api/v1/veo/record-info` with query param `taskId` from Create Video response.  
    - Authentication with Kie.ai credentials.  
    - Input from "Wait 2".

17. **Create Aggregate Node "Aggregate":**  
    - Aggregate field: `data.response.resultUrls[0]` to collect all clip URLs into an array.  
    - Input from "Get Video".

18. **Create HTTP Request Node "Combine Clips":**  
    - POST to `https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`  
    - JSON body with `video_urls`: array of URLs from Aggregate output.  
    - Authentication with fal.ai credentials.  
    - Input from "Aggregate".

19. **Create Wait Node "Wait":**  
    - Wait 100 seconds.  
    - Input from "Combine Clips".

20. **Create HTTP Request Node "Get Final Video":**  
    - GET request to URL returned by Combine Clips response.  
    - Authentication with fal.ai credentials.  
    - Input from "Wait".

21. **Create If Node "If 2":**  
    - Condition: Check if final video URL is non-empty string.  
    - True branch: Send Video  
    - False branch: Wait (retry or error handling)  
    - Input from "Get Final Video".

22. **Create Telegram Node "Send Video":**  
    - Send video to Telegram chat using video URL from If 2 true branch.  
    - Input from "If 2".

23. **Create LangChain Think Tool "Think":**  
    - Used as an AI tool to double-check outputs from both AI agents.  
    - Connect to both UGCRobo AI agents for output validation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 📌 How to Set Up the AI UGC Video Automation System: This workflow integrates Telegram + N8N + AI models to generate UGC marketing videos automatically. The system flow includes image analysis, prompt generation, media creation, video clip generation, merging, and output delivery. Customization includes choice of AI models (V3 Fast or Quality) and output destinations (Telegram, Google Drive, Dropbox). Cost estimates and usage notes are included in the workflow sticky notes. | See "Sticky Note5" node content within the workflow JSON.                                        |
| The system uses casual, amateur iPhone-style aesthetics for all generated content, emphasizing natural and candid UGC realism in both images and videos. Dialogue is conversational, under 150 characters, and avoids formal sales language, ensuring authenticity.                                                                                                                                                                                                                                            | System prompt in UGCRobo - Video AI Agent and UGCRobo - Image AI Agent nodes.                   |
| Telegram Bot setup requires obtaining the Bot ID and properly configuring credentials in n8n nodes for API interaction. The bot listens for messages with photos and captions that drive the entire workflow.                                                                                                                                                                                                                                                                                                     | "Bot ID" node and Telegram Trigger configuration.                                               |
| Kie.ai API is used for AI-generated image and video content creation. fal.ai API is used for merging video clips into a continuous video. Both require API authentication set in n8n credentials.                                                                                                                                                                                                                                                                                                                 | Kie.ai and fal.ai HTTP Request nodes with HTTP Header Auth credentials.                         |
| The workflow includes multiple wait nodes to handle asynchronous processing delays inherent to AI content generation services. Timings are empirically set to balance responsiveness and reliability.                                                                                                                                                                                                                                                                                                           | Wait nodes (Wait, Wait 2, Wait 3) with configured delay times.                                  |
| The LangChain Think Tool is integrated for output validation of AI-generated content, helping ensure prompt completeness and adherence to style rules before proceeding to media generation.                                                                                                                                                                                                                                                                                                                  | Think node and its integration with AI agent nodes.                                            |
| For developers extending this workflow: additional output destinations can be integrated (e.g., Google Drive, Dropbox) by adding new output nodes after merging or final video retrieval. Additional error handling (e.g., retries, fallbacks) can be implemented on the If nodes controlling flow based on API response presence or validity.                                                                                                                                                                         | Suggested customization based on current workflow structure.                                   |

---

**Disclaimer:** The provided text and workflow analysis are based exclusively on an n8n automated workflow using public and legal data. All content complies with relevant content policies and contains no illegal, offensive, or protected material.