Generate Videos from Images with Telegram, GPT-4.1 & Seedance/Veo3 Integration

https://n8nworkflows.xyz/workflows/generate-videos-from-images-with-telegram--gpt-4-1---seedance-veo3-integration-6976


# Generate Videos from Images with Telegram, GPT-4.1 & Seedance/Veo3 Integration

### 1. Workflow Overview

This workflow automates the generation of short-form videos from images sent via Telegram by leveraging OpenAI GPT-4.1, LangChain AI Agent, Google Drive and Sheets for storage and logging, and the Wavespeed Seedance/Veo3 API for image-to-video conversion. It is designed to receive a photo with caption on Telegram, create a video prompt using AI, upload the image, log the activity, request video generation, poll for completion, and finally send the generated video back to the Telegram user.

**Logical Blocks:**

- **1.1 Input Reception and Validation**  
  Captures Telegram messages and filters those containing photos.

- **1.2 AI Processing: Video Prompt Generation**  
  Uses LangChain Agent with GPT-4.1-mini to generate a video prompt and fetch the latest image URL from Google Sheets.

- **1.3 Image Upload and Logging**  
  Downloads the Telegram photo, uploads it to Google Drive, and logs the upload info in Google Sheets.

- **1.4 Video Generation Request and Polling**  
  Sends the prompt and image URL to the Wavespeed Seedance API, waits for processing, polls for completion status.

- **1.5 Output Delivery**  
  Sends the generated video back to the Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
  Listens for incoming Telegram messages, filters messages to only those containing photos.

- **Nodes Involved:**  
  - Telegram Trigger  
  - If (conditional node)

- **Node Details:**  

  **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Entry point, listens for Telegram "message" updates.  
  - Config: Listens to all messages; no filtering at this stage.  
  - Input/Output: Output connects to If node.  
  - Edge cases: No photo in message → filtered out by If node.

  **If**  
  - Type: Conditional node  
  - Role: Checks if the incoming message contains a photo array.  
  - Config: Condition tests if message.photo array exists (not empty).  
  - Input: Telegram Trigger output.  
  - Output: Main True branch leads to AI Agent; False branch leads to Telegram node for fallback (not detailed).  
  - Edge cases: Messages without photos are ignored or handled separately.

#### 2.2 AI Processing: Video Prompt Generation

- **Overview:**  
  Uses LangChain AI Agent with OpenAI GPT-4.1-mini to generate a JSON output containing a video prompt and an image URL from Google Sheets.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  
  - Google Sheets1  
  - Think (tool node)

- **Node Details:**  

  **AI Agent**  
  - Type: LangChain Agent node  
  - Role: Generates effective video prompt JSON from user’s Telegram text input, referencing the latest image URL.  
  - Config:  
    - Input text from Telegram message caption.  
    - System message instructs generating only `{ prompt, image_url }` JSON output.  
  - Inputs: Condition True from If node; AI memory and tools linked to Simple Memory and Think nodes.  
  - Outputs: JSON with prompt and image_url, passed to Wavespeed Post.

  **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4.1-mini language model for AI Agent.  
  - Config: Model set to "gpt-4.1-mini".  
  - Inputs: From AI Agent (ai_languageModel role).  
  - Edge cases: API key/auth errors, rate limits.

  **Simple Memory**  
  - Type: LangChain memory buffer window  
  - Role: Maintains conversational context keyed by Telegram chat ID.  
  - Config: Session key extracted from Telegram chat id.  
  - Input: AI Agent’s memory input.  
  - Edge cases: If session key missing, context lost.

  **Google Sheets1**  
  - Type: Google Sheets Tool node  
  - Role: Provides access to Google Sheet storing image URLs; AI Agent uses this to fetch last image URL.  
  - Config: Connected to specific Google Sheet by document ID and sheet name.  
  - Inputs: AI Agent tool input.  
  - Edge cases: Sheet not found, permission errors.

  **Think**  
  - Type: LangChain toolThink node  
  - Role: Handles internal AI tool thinking process; connected to AI Agent as tool.  
  - Inputs: AI Agent ai_tool.  
  - No parameters.

#### 2.3 Image Upload and Logging

- **Overview:**  
  Downloads the original Telegram photo, uploads it to Google Drive, and appends a new row in Google Sheets logging the upload timestamp and Google Drive link.

- **Nodes Involved:**  
  - Telegram  
  - Google Drive  
  - Google Sheets

- **Node Details:**  

  **Telegram**  
  - Type: Telegram node (file resource)  
  - Role: Downloads the highest-resolution photo from Telegram message.  
  - Config: Uses file_id of 4th photo size item in message.photo array.  
  - Input: False branch from If node (possibly fallback) or from earlier nodes.  
  - Output: Binary file data passed to Google Drive node.  
  - Edge cases: Photo array less than 4 sizes, file download errors.

  **Google Drive**  
  - Type: Google Drive upload node  
  - Role: Uploads downloaded photo to a specified Google Drive folder.  
  - Config:  
    - Filename includes current timestamp.  
    - Upload destination folder specified by folderId.  
  - Input: Telegram photo file.  
  - Output: Metadata including webContentLink (download URL) for logging.  
  - Edge cases: Permission errors, folder not found.

  **Google Sheets**  
  - Type: Google Sheets append node  
  - Role: Logs the upload with current timestamp and Google Drive image URL.  
  - Config:  
    - Appends row with "Date" and "Image Uploaded on Google Drive" columns.  
    - Targets specific Google Sheet document and sheet name.  
  - Input: Google Drive output JSON with webContentLink.  
  - Edge cases: Sheet permission, schema mismatch.

#### 2.4 Video Generation Request and Polling

- **Overview:**  
  Sends the AI-generated video prompt and image URL to Wavespeed Seedance API to initiate video generation, waits for processing, polls for completion status.

- **Nodes Involved:**  
  - Wavespeed Post  
  - Wait 15  
  - Wavespeed Get  
  - If1  
  - Wait another 15 Seconds

- **Node Details:**  

  **Wavespeed Post**  
  - Type: HTTP Request node  
  - Role: Sends POST request to Wavespeed Seedance API endpoint for image-to-video generation.  
  - Config:  
    - URL hardcoded to Seedance v1 pro i2v 480p endpoint.  
    - Body includes camera_fixed=false, duration=5 seconds, image (from AI Agent’s image_url), prompt (from AI Agent’s prompt), seed=-1.  
    - Auth via HTTP header (API key credential).  
  - Input: AI Agent JSON output with prompt and image_url.  
  - Output: JSON with prediction id for polling.

  **Wait 15**  
  - Type: Wait node  
  - Role: Delays 15 seconds to allow video processing to start.  
  - Input: Wavespeed Post output.  
  - Output: To Wavespeed Get.

  **Wavespeed Get**  
  - Type: HTTP Request node  
  - Role: Polls Wavespeed API for prediction result status and outputs.  
  - Config:  
    - URL dynamically formed using prediction id from post response.  
    - Auth with same API key.  
  - Input: After Wait 15.  
  - Output: JSON with status and video output URLs.

  **If1**  
  - Type: Conditional node  
  - Role: Checks if Wavespeed prediction status is "completed".  
  - Config: Condition tests if `data.status == "completed"`.  
  - Input: Wavespeed Get output.  
  - Output:  
    - True branch sends video to Telegram2 node.  
    - False branch leads to Wait another 15 Seconds.

  **Wait another 15 Seconds**  
  - Type: Wait node  
  - Role: Delays 15 seconds before retrying status check.  
  - Input: If1 False branch.  
  - Output: Loops back to Wavespeed Get node.

- Edge cases:  
  - API errors or timeouts.  
  - Prediction never reaches completed status → potential endless loop or needs timeout handling.  
  - Network failures.  

#### 2.5 Output Delivery

- **Overview:**  
  Sends the generated video file back to the Telegram chat.

- **Nodes Involved:**  
  - Telegram2

- **Node Details:**  

  **Telegram2**  
  - Type: Telegram node (sendVideo operation)  
  - Role: Sends the generated video file to Telegram user.  
  - Config:  
    - Uses video URL from Wavespeed Get node's outputs array.  
    - Specifies chatId (hardcoded or dynamic).  
  - Input: If1 True branch.  
  - Edge cases: Telegram API errors, invalid file URLs.

---

### 3. Summary Table

| Node Name       | Node Type                     | Functional Role                              | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                         |
|-----------------|-------------------------------|----------------------------------------------|---------------------|-----------------------|--------------------------------------------------------------------------------------------------|
| Telegram Trigger| Telegram Trigger              | Entry point: listens for Telegram messages   | -                   | If                    | Telegram Trigger & IF Node                                                                        |
| If              | If (conditional)              | Filters messages containing photos            | Telegram Trigger    | AI Agent (true), Telegram (false) | Telegram Trigger & IF Node                                                                        |
| AI Agent        | LangChain Agent               | Generates video prompt JSON from user input  | If (true)           | Wavespeed Post        | Video Prompt & Image URL Agent                                                                    |
| OpenAI Chat Model| LangChain OpenAI Chat Model  | Provides GPT-4.1-mini model for AI Agent     | AI Agent (ai_languageModel) | AI Agent          | Video Prompt & Image URL Agent                                                                    |
| Simple Memory   | LangChain Memory Buffer       | Maintains chat context keyed by chat ID      | AI Agent (ai_memory) | AI Agent              | Video Prompt & Image URL Agent                                                                    |
| Google Sheets1  | Google Sheets Tool            | Provides latest image URL for AI Agent       | AI Agent (ai_tool)   | AI Agent              | Video Prompt & Image URL Agent                                                                    |
| Think           | LangChain ToolThink           | AI internal thinking tool                      | AI Agent (ai_tool)   | AI Agent              | Video Prompt & Image URL Agent                                                                    |
| Telegram        | Telegram                      | Downloads photo from Telegram message         | If (false) or other  | Google Drive          | Upload Image URL                                                                                   |
| Google Drive    | Google Drive upload           | Uploads photo to Drive, outputs public URL    | Telegram             | Google Sheets          | Upload Image URL                                                                                   |
| Google Sheets   | Google Sheets append          | Logs upload date and image URL                 | Google Drive         | -                     | Upload Image URL                                                                                   |
| Wavespeed Post  | HTTP Request                 | Requests video generation from Seedance API  | AI Agent             | Wait 15                | Wavespeed (Seedance, Veo3 Image-to-text API)                                                     |
| Wait 15        | Wait                          | Delays 15 seconds for processing               | Wavespeed Post       | Wavespeed Get          | Wavespeed (Seedance, Veo3 Image-to-text API)                                                     |
| Wavespeed Get  | HTTP Request                 | Polls video generation status                   | Wait 15              | If1                    | Wavespeed (Seedance, Veo3 Image-to-text API)                                                     |
| If1             | If (conditional)              | Checks if video generation is complete         | Wavespeed Get        | Telegram2 (true), Wait another 15 Seconds (false) | Wavespeed (Seedance, Veo3 Image-to-text API)                                                     |
| Wait another 15 Seconds | Wait                  | Delays 15 seconds before next status check     | If1 (false)          | Wavespeed Get          | Wavespeed (Seedance, Veo3 Image-to-text API)                                                     |
| Telegram2       | Telegram                      | Sends generated video file back to user        | If1 (true)           | -                      | Telegram Output                                                                                   |
| Sticky Note     | Sticky Note                   | Annotation for Telegram Trigger & If block     | -                    | -                      | Telegram Trigger & IF Node                                                                        |
| Sticky Note1    | Sticky Note                   | Annotation for AI Agent block                    | -                    | -                      | Video Prompt & Image URL Agent                                                                    |
| Sticky Note2    | Sticky Note                   | Annotation for Upload Image URL block            | -                    | -                      | Upload Image URL                                                                                   |
| Sticky Note3    | Sticky Note                   | Annotation for Wavespeed Seedance API block      | -                    | -                      | Wavespeed (Seedance, Veo3 Image-to-text API)                                                     |
| Sticky Note4    | Sticky Note                   | Annotation for Telegram Output block               | -                    | -                      | Telegram Output                                                                                   |
| Sticky Note5    | Sticky Note                   | Full workflow explanation and credits            | -                    | -                      | See detailed content in notes below                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure updates to listen for "message" updates.  
   - Set up webhook and credentials for your Telegram Bot.

2. **Add If Node to Filter Messages with Photos**  
   - Condition: Check if `message.photo` exists and is not empty.  
   - True branch proceeds to AI Agent.  
   - False branch proceeds to Telegram node for fallback or ignore.

3. **Set up AI Agent Block**  
   - Add LangChain Agent node.  
   - Connect OpenAI Chat Model node configured with GPT-4.1-mini model.  
   - Add Simple Memory node for conversational context, keyed by Telegram chat ID (`{{$json.message.chat.id}}`).  
   - Add Google Sheets Tool node linked to your Google Sheet storing image URLs.  
   - Add Think node as AI tool connector.  
   - Configure AI Agent:  
     - Input text: Telegram message text.  
     - System message: Instruct to output JSON with "prompt" and "image_url" only, fetch latest image_url from Google Sheet.  
   - Connect OpenAI Chat Model and Simple Memory as AI Agent languageModel and memory inputs.  
   - Connect Google Sheets Tool and Think nodes as AI tools for AI Agent.

4. **Download Telegram Image**  
   - Add Telegram node configured to download file resource.  
   - Use file_id of the highest resolution photo (`{{$json.message.photo[3].file_id}}`).  
   - Connect from If node False branch or from AI Agent output as needed.

5. **Upload Image to Google Drive**  
   - Add Google Drive node to upload file.  
   - Configure folderId to your target Google Drive folder.  
   - Set filename to include timestamp (e.g., `"Image Upload - {{$now}}"`).  
   - Connect Telegram download node output to Google Drive.

6. **Log Image Upload to Google Sheets**  
   - Add Google Sheets append node.  
   - Configure sheet name and document ID to your logging sheet.  
   - Map columns:  
     - "Date" → current timestamp (`{{$now}}`)  
     - "Image Uploaded on Google Drive" → Google Drive file webContentLink (`{{$json.webContentLink}}`)  
   - Connect Google Drive output to Google Sheets.

7. **Send Video Generation Request to Wavespeed API**  
   - Add HTTP Request node (Wavespeed Post).  
   - Method: POST  
   - URL: `https://api.wavespeed.ai/api/v3/bytedance/seedance-v1-pro-i2v-480p`  
   - Authentication: HTTP Header with your Wavespeed API key.  
   - Body parameters (JSON):  
     - camera_fixed: false  
     - duration: 5  
     - image: `{{$json.message.content.image_url}}` (from AI Agent output)  
     - prompt: `{{$json.message.content.prompt}}` (from AI Agent output)  
     - seed: -1  
   - Connect AI Agent output to this node.

8. **Wait 15 Seconds**  
   - Add Wait node configured to delay 15 seconds.  
   - Connect Wavespeed Post node output to Wait.

9. **Poll Wavespeed API for Video Status**  
   - Add HTTP Request node (Wavespeed Get).  
   - Method: GET  
   - URL: `https://api.wavespeed.ai/api/v3/predictions/{{$json.data.id}}/result`  
   - Authentication: Same HTTP Header credentials.  
   - Connect Wait output to this node.

10. **Check Video Generation Completion**  
    - Add If node (If1).  
    - Condition: `{{$json.data.status}} == "completed"`  
    - True branch connects to Telegram2 node; False branch connects to next Wait node.

11. **Wait Another 15 Seconds for Retry**  
    - Add Wait node configured for another 15 seconds delay.  
    - Connect If1 False branch to this Wait node.  
    - Connect Wait output back to Wavespeed Get node for polling loop.

12. **Send Generated Video Back to Telegram**  
    - Add Telegram node (Telegram2), configured with "sendVideo" operation.  
    - Set file input to video URL from Wavespeed Get output (`{{$json.data.outputs[0]}}`).  
    - Set chatId to the Telegram chat ID or specific user.  
    - Connect If1 True branch to this node.

13. **Add Sticky Notes**  
    - Add Sticky Note nodes for annotations on each block:  
      - Input reception and validation  
      - AI prompt generation  
      - Image upload and logging  
      - Wavespeed API integration  
      - Telegram output  
      - Full workflow explanation with video tutorial link

14. **Configure Credentials**  
    - Telegram Bot credentials for Trigger and send operations.  
    - Google Drive OAuth2 credentials for file upload.  
    - Google Sheets OAuth2 credentials for reading and appending.  
    - Wavespeed API key for HTTP Request authentication.  
    - OpenAI API key for LangChain OpenAI Chat Model.

15. **Test Workflow**  
    - Deploy and test by sending a photo with caption to your Telegram bot.  
    - Monitor logs for errors in API calls or data flows.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 🎥 Telegram Image-to-Video Generator Agent (Veo3 / Seedance Integration). This automation allows generation of engaging short videos from Telegram image inputs and prompts, ideal for repurposing content into reels for platforms like TikTok and Instagram.                                                                                                                                                                                                                                                       | Sticky Note5 content                                                                                   |
| ⚠️ This workflow uses community nodes and credential-based HTTP API calls. Ensure you have configured credentials for Telegram, Google Drive, Google Sheets, OpenAI, and Wavespeed before running.                                                                                                                                                                                                                                                                                                                   | Sticky Note5 content                                                                                   |
| 🛠️ The workflow is derived from a tutorial built as two separate workflows: one for Telegram input and image upload with prompt agent, another for video generation API calls.                                                                                                                                                                                                                                                                                                                                   | Sticky Note5 content                                                                                   |
| ✨ Full video tutorial available: [https://youtu.be/iaZHef5bZAc&list=PL05w1TE8X3baEGOktlXtRxsztOjeOb8Vg&index=1](https://youtu.be/iaZHef5bZAc&list=PL05w1TE8X3baEGOktlXtRxsztOjeOb8Vg&index=1)                                                                                                                                                                                                                                                                                                                           | Sticky Note5 content                                                                                   |
| Suggested use cases include generating short videos from images, reformatting static images into dynamic reels, and repurposing visual content for social media.                                                                                                                                                                                                                                                                                                                                                   | Sticky Note5 content                                                                                   |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal or protected material. All handled data is legal and public.