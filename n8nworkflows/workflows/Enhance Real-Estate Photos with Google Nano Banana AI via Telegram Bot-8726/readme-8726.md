Enhance Real-Estate Photos with Google Nano Banana AI via Telegram Bot

https://n8nworkflows.xyz/workflows/enhance-real-estate-photos-with-google-nano-banana-ai-via-telegram-bot-8726


# Enhance Real-Estate Photos with Google Nano Banana AI via Telegram Bot

### 1. Workflow Overview

This workflow automates the enhancement of real-estate photos sent via Telegram using Google’s Nano Banana AI service through the Wavespeed API. It is designed for real-estate agents, social media managers, or anyone needing quick, AI-powered photo enhancements directly from Telegram chats.

The workflow is logically grouped into the following blocks:

- **1.1 Telegram Input Reception**: Listens for incoming photo messages via Telegram and downloads the highest resolution image available.
- **1.2 Temporary Storage Upload**: Uploads the downloaded photo to Google Drive to provide a stable, accessible URL for external AI processing.
- **1.3 AI Processing Request**: Sends a POST request to the Nano Banana API to initiate asynchronous image enhancement, specifying a detailed prompt for real-estate photo enhancement.
- **1.4 Async Polling for Result**: Waits and polls the Nano Banana API until the enhancement job is completed.
- **1.5 Result Handling and Delivery**: Once complete, retrieves the enhanced image and sends it back to the original Telegram chat.
- **1.6 Workflow Orchestration and Error Handling**: Controls the conditional flow and retry waits to handle asynchronous processing and potential delays.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

**Overview:**  
This block triggers on new photo messages from Telegram users and downloads the highest quality version of the photo sent in the message.

**Nodes Involved:**  
- Telegram Trigger  
- Get a file

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages (specifically photo messages).  
  - Configuration: Triggers on "message" updates; no extra filters applied.  
  - Expressions: None (default trigger).  
  - Input: External Telegram updates.  
  - Output: Message JSON including photo array with file IDs.  
  - Edge cases: No photos sent, message with unsupported media type, Telegram API downtime.  

- **Get a file**  
  - Type: Telegram node (file resource)  
  - Role: Downloads the highest resolution photo available from the Telegram message.  
  - Configuration: Uses expression `{{$json.message.photo[3].file_id}}` to select the highest resolution photo (index 3).  
  - Input: Telegram Trigger node output.  
  - Output: File content and metadata.  
  - Edge cases: Photo array shorter than 4 items, file download failure, Telegram API rate limits.  

#### 2.2 Temporary Storage Upload

**Overview:**  
Uploads the downloaded photo to Google Drive to provide a persistent URL accessible by the Nano Banana API.

**Nodes Involved:**  
- Upload file

**Node Details:**

- **Upload file**  
  - Type: Google Drive node  
  - Role: Uploads the image file to a specified Google Drive folder (default "My Drive").  
  - Configuration: File named "Image"; folder chosen dynamically or default.  
  - Input: File from "Get a file" node.  
  - Output: Google Drive file metadata including webContentLink URL.  
  - Edge cases: Google Drive authentication failure, quota exceeded, permission denied.  
  - Credentials: Requires Google Drive OAuth2 credentials.  

#### 2.3 AI Processing Request

**Overview:**  
Sends a POST request to the Nano Banana Wavespeed API to initiate the photo enhancement job asynchronously.

**Nodes Involved:**  
- Nano Banana POST Request

**Node Details:**

- **Nano Banana POST Request**  
  - Type: HTTP Request node  
  - Role: Calls Wavespeed Nano Banana API with a detailed prompt for real-estate photo enhancement.  
  - Configuration:  
    - URL: `https://api.wavespeed.ai/api/v3/google/nano-banana/edit`  
    - Method: POST  
    - Body: JSON with:  
      - `images`: array containing the Google Drive `webContentLink` URL from the upload node.  
      - `prompt`: detailed instructions to brighten, enhance, tidy up, and keep layout intact.  
      - Flags for output format and sync mode.  
    - Authentication: HTTP header with Wavespeed API key.  
  - Input: Google Drive URL from "Upload file" node.  
  - Output: Job metadata including job `id` for polling.  
  - Edge cases: API authentication failure, invalid URL or prompt, network timeout.  
  - Credentials: Generic HTTP header authentication for Wavespeed API key.  

#### 2.4 Async Polling for Result

**Overview:**  
In this block, the workflow waits for 15 seconds, then polls the Nano Banana API for the job’s status. If not complete, waits again and polls again until completion.

**Nodes Involved:**  
- Wait 15 Secs  
- GET Result from Nano Banana Node  
- If  
- Wait 15 Secs Again

**Node Details:**

- **Wait 15 Secs**  
  - Type: Wait node  
  - Role: Pauses workflow for 15 seconds to allow asynchronous processing.  
  - Configuration: 15 seconds delay.  
  - Input: From "Nano Banana POST Request".  
  - Output: Triggers polling.  

- **GET Result from Nano Banana Node**  
  - Type: HTTP Request node  
  - Role: Polls the Nano Banana API's result endpoint for job status and output.  
  - Configuration:  
    - URL constructed dynamically using `{{$json.data.id}}` from previous POST response.  
    - Method: GET  
    - Authentication: HTTP header with Wavespeed API key.  
  - Input: From "Wait 15 Secs" or "Wait 15 Secs Again".  
  - Output: Job status and, if complete, the enhanced image data.  
  - Edge cases: Network errors, job ID not found, API rate limiting.  

- **If**  
  - Type: If node  
  - Role: Checks if the job status is `"completed"`.  
  - Configuration: Condition: `{{$json.data.status}} == "completed"`  
  - Input: Output of "GET Result from Nano Banana Node".  
  - Outputs:  
    - True: proceed to send the enhanced image back.  
    - False: wait another 15 seconds before polling again.  

- **Wait 15 Secs Again**  
  - Type: Wait node  
  - Role: Delays again for 15 seconds before re-polling.  
  - Input: False branch of "If" node.  
  - Output: Loops back to "GET Result from Nano Banana Node".  

#### 2.5 Result Handling and Delivery

**Overview:**  
Upon job completion, this block sends the enhanced image back to the original Telegram chat.

**Nodes Involved:**  
- Send a photo message

**Node Details:**

- **Send a photo message**  
  - Type: Telegram node (sendPhoto operation)  
  - Role: Sends the enhanced photo back to the user who initiated the request.  
  - Configuration:  
    - `file`: Uses expression to access the enhanced image output from the API response.  
    - `chatId`: Dynamically extracted from the original Telegram message sender ID (`{{$('Telegram Trigger').item.json.message.from.id}}`).  
    - Operation: `sendPhoto`  
  - Input: True branch of "If" node (completed job).  
  - Output: Telegram message sent confirmation.  
  - Edge cases: Telegram API send failure, invalid chat ID, image data issues.  

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                    | Input Node(s)              | Output Node(s)                  | Sticky Note                                                        |
|---------------------------|-------------------------|----------------------------------|----------------------------|---------------------------------|------------------------------------------------------------------|
| Telegram Trigger          | Telegram Trigger        | Receive incoming Telegram photos | External                   | Get a file                      | Telegram Trigger                                                  |
| Get a file                | Telegram (file)         | Download Telegram photo file      | Telegram Trigger           | Upload file                    | Download from Telegram                                            |
| Upload file               | Google Drive            | Upload photo to Google Drive      | Get a file                 | Nano Banana POST Request        | Get Image and Upload                                             |
| Nano Banana POST Request  | HTTP Request            | Send photo enhancement request   | Upload file                | Wait 15 Secs                   | POST Request to Nano Banana                                      |
| Wait 15 Secs              | Wait                    | Wait before polling result        | Nano Banana POST Request   | GET Result from Nano Banana Node | Get Results from Nano Banana                                     |
| GET Result from Nano Banana Node | HTTP Request    | Poll Nano Banana API for results  | Wait 15 Secs / Wait 15 Secs Again | If                        | Get Results from Nano Banana                                     |
| If                        | If                      | Check if enhancement completed    | GET Result from Nano Banana Node | Send a photo message / Wait 15 Secs Again | Get Results from Nano Banana                  |
| Wait 15 Secs Again        | Wait                    | Wait before re-polling            | If (false branch)          | GET Result from Nano Banana Node | Get Results from Nano Banana                                     |
| Send a photo message      | Telegram                | Send enhanced photo back          | If (true branch)           | None                          | Get Results from Nano Banana                                     |
| Sticky Note               | Sticky Note             | Visual note                      | None                      | None                          | Get Image and Upload                                             |
| Sticky Note1              | Sticky Note             | Visual note                      | None                      | None                          | Get Results from Nano Banana                                     |
| Sticky Note2              | Sticky Note             | Visual note                      | None                      | None                          | Telegram Trigger                                                  |
| Sticky Note3              | Sticky Note             | Visual note                      | None                      | None                          | Download from Telegram                                            |
| Sticky Note4              | Sticky Note             | Visual note                      | None                      | None                          | POST Request to Nano Banana                                      |
| Sticky Note5              | Sticky Note             | Visual note                      | None                      | None                          | Full workflow explanation and usage context (see content)       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Set to listen for "message" updates.  
   - No additional filters.  
   - Connect the output to next node.  
   - Credentials: Provide your Telegram Bot API token.

2. **Add "Get a file" Telegram node**  
   - Type: Telegram (file resource)  
   - Parameters:  
     - Resource: File  
     - File ID: Expression `{{$json.message.photo[3].file_id}}` (highest resolution photo).  
   - Connect input from "Telegram Trigger".  

3. **Add Google Drive "Upload file" node**  
   - Type: Google Drive  
   - Parameters:  
     - Name: "Image"  
     - Drive: Select "My Drive" or your preferred drive.  
     - Folder ID: Choose a folder or leave blank for root.  
   - Connect input from "Get a file".  
   - Credentials: Configure Google Drive OAuth2 credentials.

4. **Add HTTP Request "Nano Banana POST Request" node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.wavespeed.ai/api/v3/google/nano-banana/edit`  
     - Method: POST  
     - Authentication: HTTP Header Auth (Wavespeed API key).  
     - Body Content Type: JSON  
     - JSON Body:  
       ```json
       {
         "enable_base64_output": false,
         "enable_sync_mode": false,
         "images": [
           "{{ $json.webContentLink }}"
         ],
         "output_format": "jpeg",
         "prompt": "Enhance this apartment photo to make it appealing for real-estate listing. Brighten the overall lighting and create a warm, inviting atmosphere. Improve color balance, contrast, and sharpness to highlight the space. Tidy up and declutter visible items (e.g., remove small messes, smooth out wrinkles on furniture, clear counters). Add natural daylight tones through the windows and enhance shadows for depth. Do not alter permanent fixtures such as flooring, walls, windows, doors, or built-in cabinetry. Keep the original layout intact but ensure the space looks clean, modern, and move-in ready."
       }
       ```
   - Connect input from "Upload file".  
   - Credentials: Add Wavespeed API key as HTTP header.

5. **Add "Wait 15 Secs" node**  
   - Type: Wait  
   - Parameters: Amount = 15 seconds  
   - Connect input from "Nano Banana POST Request".

6. **Add HTTP Request "GET Result from Nano Banana Node"**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: Expression `https://api.wavespeed.ai/api/v3/predictions/{{ $json.data.id }}/result`  
     - Method: GET  
     - Authentication: HTTP Header Auth (Wavespeed API key).  
   - Connect input from "Wait 15 Secs".

7. **Add If node**  
   - Type: If  
   - Parameters: Condition to check if `{{$json.data.status}}` equals `completed`.  
   - Connect input from "GET Result from Nano Banana Node".

8. **Add "Send a photo message" Telegram node**  
   - Type: Telegram  
   - Parameters:  
     - Operation: sendPhoto  
     - File: Expression `{{$json.data.outputs[0]}}` (enhanced image).  
     - Chat ID: Expression `{{$('Telegram Trigger').item.json.message.from.id}}`.  
   - Connect this node to the true branch of the If node.

9. **Add "Wait 15 Secs Again" node**  
   - Type: Wait  
   - Parameters: Amount = 15 seconds  
   - Connect input from the false branch of the If node.

10. **Loop back "Wait 15 Secs Again" to "GET Result from Nano Banana Node"**  
    - Connect output of "Wait 15 Secs Again" back to input of "GET Result from Nano Banana Node" to continue polling until completion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| 🤖 Telegram Image Editor with Nano Banana: Send an image to your Telegram bot, and this workflow will automatically enhance it with Google’s Nano Banana (via Wavespeed API), then return the polished version back to the same chat—seamlessly. Perfect for real-estate agents, social media managers, or anyone wanting automated photo enhancements without manual editing.                                                                                                                                                                                                                                         | Workflow description sticky note                   |
| Step-by-step video tutorials of workflows like these available at [www.youtube.com/@automatewithmarc](https://www.youtube.com/@automatewithmarc)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Video tutorial link                               |
| Apps & Services used: Telegram Bot API, Google Drive for temporary storage, Wavespeed / Google Nano Banana for AI-powered image editing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Integration context                               |
| Setup instructions: Connect Telegram Bot API token, Wavespeed API key, and Google Drive OAuth2 credentials in n8n before deploying. Customize the Nano Banana prompt to adapt to different image styles as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                            | Setup and customization notes                      |
| Consider adding logging (e.g., to Google Sheets or Airtable) to track processed images for audit or analytics purposes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Suggested enhancement                              |

---

This document fully describes the "Telegram to Nano Banana Workflow" for enhancing real-estate photos with AI via Telegram. It covers the logical structure, detailed node explanations, a comprehensive node table, and step-by-step reproduction instructions, enabling both human developers and automation agents to understand, operate, and modify the workflow effectively.