Leverage Telegram and Gemini for Seamless Integrations using Kie.ai's Veo3.1 API

https://n8nworkflows.xyz/workflows/leverage-telegram-and-gemini-for-seamless-integrations-using-kie-ai-s-veo3-1-api-10393


# Leverage Telegram and Gemini for Seamless Integrations using Kie.ai's Veo3.1 API

### 1. Workflow Overview

This workflow automates the process of generating user-generated content (UGC) style video advertisements based on images and captions received via Telegram messages. It leverages Google Gemini (PaLM) for AI-driven script generation and Kie.ai's Veo3.1 API for video generation. The workflow is designed for marketers or content creators who want to quickly produce authentic, engaging short-form video ads from Telegram inputs.

**Logical Blocks:**

- **1.1 Input Reception:** Receiving Telegram messages containing product images and captions.
- **1.2 Data Extraction & Preparation:** Extracting image file IDs and captions, downloading images, and uploading them to Kie.ai storage.
- **1.3 AI Script Generation:** Using Google Gemini and Langchain to generate a UGC-style video script from the image caption.
- **1.4 Video Generation Request:** Sending the script and uploaded image URL to Kie.ai’s Veo3 API to create a video generation task.
- **1.5 Video Status Polling:** Periodically checking if the video generation is complete.
- **1.6 Video Retrieval and Delivery:** Downloading the generated HD video and sending it back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming Telegram messages that trigger the workflow, capturing user-submitted product images and captions.

- **Nodes Involved:**  
  - Receive Message

- **Node Details:**

  - **Receive Message**  
    - Type: Telegram Trigger  
    - Role: Entry point; listens for new Telegram messages  
    - Configuration: Watches for "message" updates only  
    - Credentials: Uses Telegram API credentials  
    - Input: External Telegram messages  
    - Output: JSON containing full Telegram message object  
    - Version: 1.2  
    - Failure/Potential Issues: Telegram API connection/auth errors, webhook misconfiguration, message format variations  
    - Notes: Starts the workflow when a user sends a message

#### 2.2 Data Extraction & Preparation

- **Overview:**  
  Extracts relevant data from the Telegram message (image file IDs and captions), downloads the image, and uploads it to Kie.ai storage to prepare for video generation.

- **Nodes Involved:**  
  - Extract Image & Caption  
  - Download Telegram Image  
  - Upload image to Kie Storage

- **Node Details:**

  - **Extract Image & Caption**  
    - Type: Set  
    - Role: Data extraction and assignment  
    - Configuration: Extracts caption text and the third photo file ID from the Telegram message JSON  
    - Key Expressions:  
      - `message.caption = {{$json.message.caption}}`  
      - `message.photo[3].file_id = {{$json.message.photo[2].file_id}}` (note the zero-based index)  
    - Input: Output from Receive Message  
    - Output: JSON with extracted caption and photo file ID  
    - Version: 3.4  
    - Potential Issues: Missing photo array or caption in message; index out of range errors

  - **Download Telegram Image**  
    - Type: Telegram  
    - Role: Downloads image file from Telegram servers using file ID  
    - Configuration: Uses photo file ID extracted above to download the image  
    - Key Expression: `fileId = {{$json.message.photo[3].file_id}}`  
    - Credentials: Telegram API  
    - Input: Output from Extract Image & Caption  
    - Output: Contains file path for the downloaded image  
    - Version: 1.2  
    - Potential Issues: Invalid file ID, Telegram download failure, file size limits

  - **Upload image to Kie Storage**  
    - Type: HTTP Request  
    - Role: Uploads the downloaded image from Telegram to Kie.ai cloud storage for use in video generation  
    - Configuration:  
      - POST to Kie.ai file upload endpoint  
      - Body parameters include the full Telegram file URL (constructed from the downloaded file path), upload path, and filename  
      - Authorization header with Bearer token (placeholder `<TOKEN>`)  
    - Input: Output from Download Telegram Image  
    - Output: JSON including uploaded image URL for downstream use  
    - Version: 4.3  
    - Potential Issues: HTTP failure, invalid token, incorrect file URL, upload path issues

#### 2.3 AI Script Generation

- **Overview:**  
  Generates a 30-second UGC-style ad script based on the product caption and image using Google Gemini via Langchain agent node.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Generate Script

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: Langchain Google Gemini Chat Model  
    - Role: AI language model backend for script generation  
    - Configuration: Default options, uses Google PaLM API credentials  
    - Credentials: Google Gemini (PaLM) API  
    - Input: Prompt text from Generate Script node (via Langchain agent)  
    - Output: AI-generated text response  
    - Version: 1  
    - Potential Issues: API quota limits, authentication failure, latency

  - **Generate Script**  
    - Type: Langchain Agent  
    - Role: Defines and sends prompt to Google Gemini model to generate ad script  
    - Configuration:  
      - System message defines persona as professional UGC ad scriptwriter  
      - Instructions to create authentic, friendly, relatable 30-second Instagram Reels or TikTok script  
      - Input variables in prompt: caption from 'Extract Image & Caption' node and file path from downloaded image  
      - Includes benefits, sensory details, emotional payoff, and call to action  
    - Input: Output from Download Telegram Image and Google Gemini Chat Model  
    - Output: AI-generated script text (field: `output`)  
    - Version: 3  
    - Potential Issues: Expression evaluation errors, missing input data, AI response errors

#### 2.4 Video Generation Request

- **Overview:**  
  Uploads the script and the image URL to Kie.ai Veo3 API to start the video generation task.

- **Nodes Involved:**  
  - Create Video Generation Task

- **Node Details:**

  - **Create Video Generation Task**  
    - Type: HTTP Request  
    - Role: Initiates video generation with Kie.ai Veo3 API  
    - Configuration:  
      - POST request to `https://api.kie.ai/api/v1/veo/generate`  
      - Body parameters include:  
        - `prompt`: AI-generated script text  
        - `imageUrls`: URL from Kie storage upload response (`$json.data.downloadUrl`)  
        - `model`: `veo3_fast`  
        - `aspectRatio`: `9:16` (vertical video, suitable for Reels/TikTok)  
        - `seeds`, `enableFallback`, `enableTranslation`, `generationType` set accordingly  
      - Authorization header with Bearer token  
    - Input: Output from Upload image to Kie Storage and Generate Script  
    - Output: Task details including `taskId` used for status polling  
    - Version: 4.3  
    - Potential Issues: API errors, invalid tokens, malformed payload, network issues

#### 2.5 Video Status Polling

- **Overview:**  
  Periodically checks if the video generation is complete by querying the Kie.ai API, with a wait-and-retry loop.

- **Nodes Involved:**  
  - Check Video Status  
  - Is Video Ready?  
  - Wait 30 Seconds

- **Node Details:**

  - **Check Video Status**  
    - Type: HTTP Request  
    - Role: Queries Kie.ai Veo3 API for current video generation status  
    - Configuration:  
      - GET request to `https://api.kie.ai/api/v1/veo/record-info`  
      - Query param: `taskId` from previous task creation response  
      - Authorization header  
    - Input: Output from Create Video Generation Task or Wait 30 Seconds  
    - Output: Status JSON with fields including `completeTime`  
    - Version: 4.3  
    - Potential Issues: API errors, invalid task ID, auth failures

  - **Is Video Ready?**  
    - Type: If  
    - Role: Conditional node that checks if `completeTime` field is empty or not  
    - Configuration: Checks if `{{$json.data.completeTime}}` is empty (video not ready)  
    - Input: Output from Check Video Status  
    - Output:  
      - If video not ready (completeTime empty) → Wait 30 Seconds (loop)  
      - If video ready → proceeds to next step  
    - Version: 2.2  
    - Potential Issues: JSON path errors, unexpected API response format

  - **Wait 30 Seconds**  
    - Type: Wait  
    - Role: Delays the workflow for 30 seconds before rechecking video status  
    - Configuration: Wait duration set to 30 seconds  
    - Input: Output from Is Video Ready? (not ready path)  
    - Output: Triggers Check Video Status again  
    - Version: 1.1  
    - Potential Issues: Workflow timeout if video generation takes too long

#### 2.6 Video Retrieval and Delivery

- **Overview:**  
  Once the video is ready, retrieves the HD video URL, downloads the video, and sends it back to the user via Telegram.

- **Nodes Involved:**  
  - Get HD Video URL  
  - Download Generated Video  
  - Send Video to User

- **Node Details:**

  - **Get HD Video URL**  
    - Type: HTTP Request  
    - Role: Fetches URL of 1080p video from Kie.ai API  
    - Configuration:  
      - GET request to `https://api.kie.ai/api/v1/veo/get-1080p-video`  
      - Query parameter: `taskId`  
      - Authorization header  
    - Input: Output from Is Video Ready? (video ready path)  
    - Output: JSON containing video URLs  
    - Version: 4.3  
    - Potential Issues: API failures, missing video, auth errors

  - **Download Generated Video**  
    - Type: HTTP Request  
    - Role: Downloads the actual video file from the URL received above  
    - Configuration:  
      - URL taken from `$('Check Video Status').item.json.data.response.resultUrls[0]` (first available URL)  
    - Input: Output from Get HD Video URL  
    - Output: Binary video file data  
    - Version: 4.3  
    - Potential Issues: Download failures, broken URL, network issues

  - **Send Video to User**  
    - Type: Telegram  
    - Role: Sends the downloaded video back to the Telegram chat where the original message came from  
    - Configuration:  
      - Chat ID set dynamically from original Telegram message (`{{$json.message.chat.id}}`)  
      - Operation: sendVideo  
      - Uses binary data from downloaded video  
      - Credentials: Telegram API  
    - Input: Output from Download Generated Video  
    - Output: Confirmation of message sent  
    - Version: 1.2  
    - Potential Issues: Telegram API limits, video file size restrictions, invalid chat ID

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                     | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                   |
|-------------------------|-----------------------------------|-----------------------------------|----------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Receive Message         | Telegram Trigger                   | Entry point - receive Telegram msg| -                          | Extract Image & Caption          | ##  🖼️ Extract Image & Caption\nThis node extracts the product image file ID and its caption text from the Telegram message input for use in later workflow steps. |
| Extract Image & Caption | Set                               | Extract image file ID and caption | Receive Message             | Download Telegram Image          | (Same as above)                                                                              |
| Download Telegram Image | Telegram                          | Downloads image from Telegram      | Extract Image & Caption     | Generate Script                 |                                                                                              |
| Google Gemini Chat Model| Langchain Google Gemini Chat Model| AI language model for script       | Generate Script (ai_languageModel) | Generate Script              |                                                                                              |
| Generate Script         | Langchain Agent                   | Generates UGC ad script            | Download Telegram Image, Google Gemini Chat Model | Upload image to Kie Storage  | ## 📝 Generate Script \nThis node creates a highly engaging ad script for the product using AI, written in a natural, UGC-style tone. |
| Upload image to Kie Storage | HTTP Request                   | Upload image to Kie.ai storage     | Generate Script             | Create Video Generation Task     | ## ☁️ Upload to Kie Storage\nUploads the downloaded Telegram image to Kie.ai storage via API for further video generation. |
| Create Video Generation Task | HTTP Request                 | Start video generation task        | Upload image to Kie Storage | Check Video Status              | ## 🎥 Create Video Task\nSends the script and image URL to Kie.ai’s Veo3 API to start the UGC video generation process. |
| Check Video Status      | HTTP Request                      | Poll video generation status       | Create Video Generation Task, Wait 30 Seconds | Is Video Ready?               | ## 🛰️ Check Video Status\nChecks the current generation progress of the video on Kie.ai Veo3 using the task ID. |
| Is Video Ready?         | If                                | Checks if video generation complete| Check Video Status          | Wait 30 Seconds (if not ready), Get HD Video URL (if ready) | ##  🔄 Video Status Loop\nChecks if the video generation is complete.\nIf not, waits 30 seconds before rechecking until it’s ready. |
| Wait 30 Seconds         | Wait                              | Waits 30 seconds before retrying   | Is Video Ready?             | Check Video Status              | (Same as above)                                                                              |
| Get HD Video URL        | HTTP Request                      | Fetch HD video URL                 | Is Video Ready?             | Download Generated Video         | ## 🎬 Retrieve & Send Video\n\nFetches the HD video URL from Kie.ai, downloads the generated video, and sends it back to the user on Telegram. |
| Download Generated Video| HTTP Request                      | Downloads video file               | Get HD Video URL            | Send Video to User              | (Same as above)                                                                              |
| Send Video to User      | Telegram                          | Sends video back to Telegram user  | Download Generated Video    | -                               | (Same as above)                                                                              |
| Sticky Note             | Sticky Note                      | Notes for Generate Script node     | -                          | -                               | ## 📝 Generate Script \nThis node creates a highly engaging ad script for the product using AI, written in a natural, UGC-style tone. |
| Sticky Note1            | Sticky Note                      | Notes for Extract Image & Caption  | -                          | -                               | ##  🖼️ Extract Image & Caption\nThis node extracts the product image file ID and its caption text from the Telegram message input for use in later workflow steps. |
| Sticky Note2            | Sticky Note                      | Notes for Upload to Kie Storage    | -                          | -                               | ## ☁️ Upload to Kie Storage\nUploads the downloaded Telegram image to Kie.ai storage via API for further video generation. |
| Sticky Note3            | Sticky Note                      | Notes for Create Video Task        | -                          | -                               | ## 🎥 Create Video Task\nSends the script and image URL to Kie.ai’s Veo3 API to start the UGC video generation process. |
| Sticky Note4            | Sticky Note                      | Notes for Check Video Status       | -                          | -                               | ## 🛰️ Check Video Status\nChecks the current generation progress of the video on Kie.ai Veo3 using the task ID. |
| Sticky Note5            | Sticky Note                      | Notes for Video Status Loop        | -                          | -                               | ##  🔄 Video Status Loop\nChecks if the video generation is complete.\nIf not, waits 30 seconds before rechecking until it’s ready. |
| Sticky Note6            | Sticky Note                      | Notes for Retrieve & Send Video    | -                          | -                               | ## 🎬 Retrieve & Send Video\n\nFetches the HD video URL from Kie.ai, downloads the generated video, and sends it back to the user on Telegram. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Set to listen for "message" updates only  
   - Configure Telegram API credentials  
   - Position: Start of workflow

2. **Add Set Node to Extract Image & Caption**  
   - Type: Set  
   - Assign two new fields:  
     - `message.caption` = `{{$json.message.caption}}`  
     - `message.photo[3].file_id` = `{{$json.message.photo[2].file_id}}` (third photo index)  
   - Connect output of Telegram Trigger to this node

3. **Add Telegram Node to Download Image**  
   - Type: Telegram  
   - Operation: Download file using `fileId` from extracted photo file ID  
   - Use Telegram API credentials  
   - Connect output of Set node to this node

4. **Add Langchain Google Gemini Chat Model Node**  
   - Type: Langchain Google Gemini Chat Model  
   - Configure with Google PaLM API credentials  
   - Position near Generate Script node (used as AI backend)

5. **Add Langchain Agent Node to Generate Script**  
   - Type: Langchain Agent  
   - Configure system message with UGC ad script instructions:  
     ```
     You are a professional UGC ad scriptwriter.
     Generate a 30-second UGC-style video script for Instagram Reels or TikTok
     that sounds authentic, friendly, and relatable.
     Product details:
     {{ $('Extract Image & Caption').item.json.message.caption }}
     {{ $json.result.file_path }}
     If possible, mention benefits, sensory details, and an emotional payoff.
     End with a short, clear call to action like "link in bio" or "try it yourself".
     ```  
   - Connect Telegram download image node to this node’s input  
   - Connect Google Gemini Chat Model node as AI languageModel input to this node

6. **Add HTTP Request Node to Upload Image to Kie Storage**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://kieai.redpandaai.co/api/file-url-upload`  
   - Body parameters:  
     - `fileUrl`: `"https://api.telegram.org/file/bot<bot-token>/" + {{$json.result.file_path}}`  
     - `uploadPath`: `images/downloaded`  
     - `fileName`: `my-downloaded-image.jpg`  
   - Headers: Authorization Bearer token (set your valid token)  
   - Connect output of Generate Script node here

7. **Add HTTP Request Node to Create Video Generation Task**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/veo/generate`  
   - Body parameters:  
     - `prompt`: `{{$json.output}}` (script text from Generate Script)  
     - `imageUrls`: `{{$json.data.downloadUrl}}` (from upload response)  
     - `model`: `veo3_fast`  
     - `aspectRatio`: `9:16`  
     - `seeds`: `12345`  
     - `enableFallback`: `false`  
     - `enableTranslation`: `true`  
     - `generationType`: `TEXT_2_VIDEO`  
   - Headers: Authorization Bearer token  
   - Connect output of Upload image to Kie Storage node here

8. **Add HTTP Request Node to Check Video Status**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/veo/record-info`  
   - Query parameter: `taskId = {{$json.data.taskId}}`  
   - Headers: Authorization Bearer token  
   - Connect output of Create Video Generation Task node here

9. **Add If Node to Check if Video Is Ready**  
   - Condition: Check if `{{$json.data.completeTime}}` is empty  
   - If TRUE (empty): Video not ready → connect to Wait 30 Seconds node  
   - If FALSE (not empty): Video ready → connect to Get HD Video URL node

10. **Add Wait Node**  
    - Type: Wait  
    - Duration: 30 seconds  
    - Connect output of If node (not ready path) to this node  
    - Connect output of Wait node back to Check Video Status node (loop)

11. **Add HTTP Request Node to Get HD Video URL**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.kie.ai/api/v1/veo/get-1080p-video`  
    - Query parameter: `taskId = {{$json.data.taskId}}`  
    - Headers: Authorization Bearer token  
    - Connect output of If node (ready path) here

12. **Add HTTP Request Node to Download Generated Video**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{$json.response.resultUrls[0]}}` (from previous node)  
    - Connect output of Get HD Video URL here

13. **Add Telegram Node to Send Video to User**  
    - Type: Telegram  
    - Operation: sendVideo  
    - Chat ID: `{{$json.message.chat.id}}` (from original Receive Message node)  
    - Send binary video data as video file  
    - Use Telegram API credentials  
    - Connect output of Download Generated Video here

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| UGC-style video scripts should sound authentic, friendly, and relatable—like real users sharing experiences. | Guidance in Generate Script node system message      |
| Kie.ai Veo3 API requires Bearer token authorization for all requests; ensure tokens are valid and refreshed. | API header requirements in HTTP Request nodes       |
| Telegram image download uses file ID that may be in a nested photo array; indexing carefully is critical.      | Extract Image & Caption node extraction logic        |
| Video generation may take several minutes; the workflow includes a 30-second polling loop to handle this.     | Video Status Loop sticky note and Wait node          |
| Kie.ai storage upload requires the full Telegram file URL constructed from the bot token and downloaded file path. | Upload image to Kie Storage HTTP Request configuration |
| Workflow assumes vertical video format (9:16) optimized for Instagram Reels and TikTok platforms.               | Aspect ratio setting in Create Video Generation Task  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow built using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled are legal and publicly accessible.