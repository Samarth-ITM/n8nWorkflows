Transform Selfies into Professional LinkedIn Headshots with Nano Banana Pro & Telegram

https://n8nworkflows.xyz/workflows/transform-selfies-into-professional-linkedin-headshots-with-nano-banana-pro---telegram-11458


# Transform Selfies into Professional LinkedIn Headshots with Nano Banana Pro & Telegram

### 1. Workflow Overview

This workflow automates converting user-submitted selfies into professional LinkedIn profile headshots using the Nano Banana Pro AI model via Fal.ai API. It supports two input methods: a Telegram bot where users send photos directly, and a manual trigger allowing users to input an image URL directly. The workflow sends the image for AI enhancement, polls the API for completion, then downloads and uploads the enhanced image to Google Drive and FTP storage. It includes validation, secure user filtering, and automated file naming with timestamps.

Logical blocks:

- **1.1 Input Reception**: Receives images either from Telegram photo messages or a manual trigger with image URL.
- **1.2 User Authorization and Input Filtering**: Ensures only authorized Telegram users can proceed.
- **1.3 Image Retrieval and FTP Parameter Setup**: Downloads Telegram image files, sets FTP paths and base URLs for storage.
- **1.4 Image Upload to FTP**: Uploads original images to FTP space to generate accessible URLs.
- **1.5 AI Request Creation (Nano Banana Pro)**: Sends the image URL to Fal.ai API with a detailed prompt to create a professional headshot.
- **1.6 AI Processing Status Polling**: Waits and polls the API until the AI completes processing.
- **1.7 Download Enhanced Image and Upload to Storage**: Retrieves the final AI-generated image, then uploads it to Google Drive and FTP.
- **1.8 Workflow Control and Error Handling**: Includes conditional checks for completion and retry logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives incoming images either via Telegram messages or manual workflow execution with an image URL.

**Nodes Involved:**  
- Telegram Trigger  
- When clicking ‘Execute workflow’  
- Switch

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages with photos or other content.  
  - Key Config: Triggers on any message update. Uses Telegram API credentials.  
  - Input: Incoming Telegram message updates.  
  - Output: JSON containing message and photo metadata.  
  - Edge Cases: Trigger may activate for non-photo messages; filtered later.

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution with pre-defined image URL.  
  - Input: Manual user initiation.  
  - Output: Triggers downstream nodes with static or set image URL.  
  - Edge Cases: User must update the fixed image URL node.

- **Switch**  
  - Type: Switch  
  - Role: Routes input based on content type: Text, Audio (voice), or Image.  
  - Key Expressions: Checks existence of message.text, message.voice.file_id, or message.photo[0].  
  - Input: Output from Sanitaze node (authorized Telegram message data).  
  - Output: Routes only image messages to the next block.  
  - Edge Cases: Non-photo messages or unauthorized users are discarded.

---

#### 1.2 User Authorization and Input Filtering

**Overview:**  
Filters incoming Telegram messages to allow only a specific Telegram user ID to proceed.

**Nodes Involved:**  
- Sanitaze  
- Sticky Note (contextual instruction)

**Node Details:**

- **Sanitaze**  
  - Type: Code  
  - Role: Checks if the Telegram message sender's ID matches a predefined authorized user ID.  
  - Config: JavaScript checks `$input.first().json.message.from.id !== 11111` (replace with user ID).  
  - Input: Telegram Trigger output.  
  - Output: Passes authorized messages; otherwise returns `{ unauthorized: true }` to stop the flow.  
  - Edge Cases: Unauthorized users' data is blocked early.

- **Sticky Note** (related)  
  - Instructions to replace `11111` with the authorized Telegram user ID.

---

#### 1.3 Image Retrieval and FTP Parameter Setup

**Overview:**  
Downloads Telegram photo files, sets FTP parameters for storage paths and URLs.

**Nodes Involved:**  
- Get Image  
- Set FTP params  
- Merge  
- Sticky Note (FTP parameters explanation)

**Node Details:**

- **Get Image**  
  - Type: Telegram  
  - Role: Downloads the highest resolution Telegram photo file (index 2).  
  - Config: Uses Telegram API credentials.  
  - Input: Routed from Switch node for image messages.  
  - Output: Binary data of the image file and metadata.  
  - Edge Cases: Telegram API file download errors, missing file IDs.

- **Set FTP params**  
  - Type: Set  
  - Role: Defines FTP storage path and base URL for image upload and retrieval.  
  - Config: User must set `ftp_path` (e.g., `/public_html/images/`) and `base_url` (e.g., `https://website.com/images/`).  
  - Input: Output from Get Image.  
  - Output: JSON with FTP parameters merged with image data.  
  - Edge Cases: Incorrect FTP paths or URLs can cause upload or retrieval failures.

- **Merge**  
  - Type: Merge  
  - Role: Combines FTP params with image data for downstream upload.  
  - Input: Outputs from Set FTP params and Get Image.  
  - Output: Combined dataset for FTP upload.

- **Sticky Note**  
  - Explains how to set FTP server path and base URL, with alternative instructions on using manual trigger if FTP is not available.

---

#### 1.4 Image Upload to FTP

**Overview:**  
Uploads the original Telegram image to the configured FTP server to make it accessible via URL.

**Nodes Involved:**  
- Upload to FTP  
- FTP Image URL  
- Sticky Note (general workflow steps and API key instructions)

**Node Details:**

- **Upload to FTP**  
  - Type: FTP  
  - Role: Uploads the image file to the path defined by concatenating FTP path and filename extracted from the uploaded file’s path.  
  - Credentials: FTP BunnyCDN (configured via FTP OAuth credentials).  
  - Input: Combined data from Merge node.  
  - Output: FTP upload result with file path.

- **FTP Image URL**  
  - Type: Set  
  - Role: Constructs the accessible image URL by combining base_url and the uploaded file’s filename.  
  - Input: Output from Upload to FTP.  
  - Output: JSON with `image` property holding the full image URL string.

- **Sticky Note6**  
  - Instructions to get Fal.ai API Key and set Authorization header in "Create Image" node.

- **Sticky Note1**  
  - Provides a comprehensive overview of the workflow and setup steps.

---

#### 1.5 AI Request Creation (Nano Banana Pro)

**Overview:**  
Sends a POST request to the Fal.ai Nano Banana Pro API to transform the image URL into a professional LinkedIn headshot.

**Nodes Involved:**  
- Create Image

**Node Details:**

- **Create Image**  
  - Type: HTTP Request  
  - Role: Sends JSON body with prompt details and image URL to Nano Banana Pro API endpoint.  
  - Config:  
    - URL: `https://queue.fal.run/fal-ai/nano-banana-pro/edit`  
    - Method: POST  
    - Headers: Includes `Authorization: Key YOURAPIKEY` and `Content-Type: application/json`.  
    - Body: JSON containing the transformation prompt, image URL, resolution 1K, PNG output, etc.  
  - Input: Receives the image URL from FTP Image URL node.  
  - Output: JSON containing a `request_id` for polling status.  
  - Edge Cases: API authentication errors, invalid image URLs, network timeouts.

---

#### 1.6 AI Processing Status Polling

**Overview:**  
Polls the Fal.ai API to check if the AI image processing is completed, with wait intervals and conditional checks.

**Nodes Involved:**  
- Wait 60 sec.  
- Get status  
- Completed? (If)

**Node Details:**

- **Wait 60 sec.**  
  - Type: Wait  
  - Role: Pauses execution for 10 seconds (parameter `amount`=10).  
  - Input: After Create Image and on status check failure.  
  - Output: Triggers next status check.

- **Get status**  
  - Type: HTTP Request  
  - Role: Sends GET request to `https://queue.fal.run/fal-ai/nano-banana-pro/requests/{{request_id}}/status` endpoint to query job status.  
  - Authentication: Uses Fal.run API key header.  
  - Input: Request ID from Create Image node.  
  - Output: JSON with current status (`COMPLETED` or other).  
  - Edge Cases: API errors, network issues.

- **Completed?**  
  - Type: If  
  - Role: Checks if the status field equals `"COMPLETED"`.  
  - True Output: Proceeds to download final image.  
  - False Output: Loops back to Wait 60 sec. node.  
  - Edge Cases: Infinite loop if job is stuck or failed. No explicit failure handling.

---

#### 1.7 Download Enhanced Image and Upload to Storage

**Overview:**  
Fetches the enhanced image from the API, then uploads it to Google Drive and FTP for storage.

**Nodes Involved:**  
- Get Url image  
- Get File image  
- Upload Image (Google Drive)  
- Upload to FTP1  
- Sticky Note8 (instruction)

**Node Details:**

- **Get Url image**  
  - Type: HTTP Request  
  - Role: Retrieves the final image details from Fal.ai API using the request ID.  
  - Config: GET request to `https://queue.fal.run/fal-ai/nano-banana-pro/requests/{{request_id}}`.  
  - Output: JSON containing image URL and metadata.

- **Get File image**  
  - Type: HTTP Request  
  - Role: Downloads the image file from the URL found in the previous node.  
  - Configured to download the first image URL in the response JSON.  
  - Output: Binary image data.

- **Upload Image**  
  - Type: Google Drive  
  - Role: Uploads the AI-enhanced image file to a configured Google Drive folder.  
  - Config: Filename uses current timestamp plus original file name, storing in a specific folder ID.  
  - Credentials: Google Drive OAuth2 account.  
  - Output: Confirms upload success.

- **Upload to FTP1**  
  - Type: FTP  
  - Role: Uploads the same enhanced image file to FTP storage with a timestamped filename.  
  - Credentials: FTP BunnyCDN.  
  - Output: FTP upload result.

- **Sticky Note8**  
  - Instructs user to upload CV/Linkedin images to Google Drive or FTP.

---

#### 1.8 Workflow Control and Error Handling

**Overview:**  
Controls flow with conditional checks, loops, and user instructions throughout the workflow.

**Nodes Involved:**  
- Sticky Notes (various)  
- Switch  
- Sanitaze  
- Completed?

**Node Details:**

- Sticky notes provide user setup instructions for API keys, FTP parameters, Telegram user ID, and workflow overview.  
- Switch node routes different Telegram message types.  
- Sanitaze node enforces user authorization.  
- Completed? node loops the polling mechanism until completion.

Edge cases include lack of explicit error handling for API failures, retries after network errors, or unauthorized inputs. The workflow assumes successful API responses and user configuration.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                                 | Input Node(s)            | Output Node(s)       | Sticky Note                                                                                                  |
|---------------------|---------------------|------------------------------------------------|--------------------------|----------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger    | Receives Telegram messages                       |                          | Sanitaze             |                                                                                                              |
| Sanitaze            | Code                | Filters authorized Telegram users                | Telegram Trigger          | Switch               | ## STEP 1 - Sanitaze Replace XXXX with your Telegram user ID                                                 |
| Switch              | Switch              | Routes messages based on content type            | Sanitaze                  | Get Image (Image)     |                                                                                                              |
| Get Image           | Telegram             | Downloads Telegram photo file                     | Switch                    | Set FTP params, Merge |                                                                                                              |
| Set FTP params       | Set                  | Sets FTP path and base URL                        | Get Image                 | Merge                | ## STEP 2 - Ftp Params for upload image - Set Server Path and Base url. Alternatively use manual trigger.    |
| Merge               | Merge                | Combines FTP params with image data               | Get Image, Set FTP params | Upload to FTP         |                                                                                                              |
| Upload to FTP        | FTP                  | Uploads original image to FTP                      | Merge                     | FTP Image URL         |                                                                                                              |
| FTP Image URL        | Set                  | Constructs public URL for uploaded image          | Upload to FTP             | Create Image          | ## STEP 3 - GET NANO BANANO PRO API KEY (YOURAPIKEY) Create account at fal.ai and set Authorization header.  |
| Create Image        | HTTP Request         | Sends image and prompt to Nano Banana Pro API     | FTP Image URL             | Wait 60 sec.          |                                                                                                              |
| Wait 60 sec.         | Wait                 | Waits before polling status                        | Create Image, Completed?  | Get status            |                                                                                                              |
| Get status           | HTTP Request         | Polls Fal.ai for job status                        | Wait 60 sec.              | Completed?            |                                                                                                              |
| Completed?           | If                   | Checks if AI processing is complete                | Get status                | Get Url image (true), Wait 60 sec. (false) |                                                                                                              |
| Get Url image        | HTTP Request         | Retrieves final image info from API                | Completed? (true)          | Get File image        |                                                                                                              |
| Get File image       | HTTP Request         | Downloads the final enhanced image                  | Get Url image             | Upload Image, Upload to FTP1 |                                                                                                              |
| Upload Image         | Google Drive          | Uploads enhanced image to Google Drive              | Get File image            |                      | ## STEP 4 - GET NEW IMAGE Get your CV / Linkedin image and upload it on Google Drive or your FTP Space        |
| Upload to FTP1       | FTP                  | Uploads enhanced image to FTP                        | Get File image            |                      |                                                                                                              |
| When clicking ‘Execute workflow’ | Manual Trigger   | Allows manual workflow start                        |                          | Fix Image Url         |                                                                                                              |
| Fix Image Url        | Set                  | Sets image URL manually for manual trigger          | When clicking ‘Execute workflow’ | Create Image          |                                                                                                              |
| Sticky Note6         | Sticky Note           | Instructions for API key setup                      |                          |                      | ## STEP 3 - GET NANO BANANO PRO API KEY (YOURAPIKEY) Create account at fal.ai and set Authorization header.  |
| Sticky Note7         | Sticky Note           | FTP parameter setup instructions                    |                          |                      | ## STEP 2 - Ftp Params for upload image Set Server Path and Base url. Alternatively use manual trigger.       |
| Sticky Note8         | Sticky Note           | Instructions for uploading final images             |                          |                      | ## STEP 4 - GET NEW IMAGE Get your CV / Linkedin image and upload it on Google Drive or your FTP Space        |
| Sticky Note          | Sticky Note           | Sanitaze user ID instruction                         |                          |                      | ## STEP 1 - Sanitaze Replace XXXX with your Telegram user ID                                                 |
| Sticky Note1         | Sticky Note           | Workflow overview and setup instructions            |                          |                      | ## LinkedIn Profile Photo Maker from Selfie with AI Nano Banana Pro and Telegram Overview and Setup steps     |
| Sticky Note2         | Sticky Note           | Example image "From selfie"                          |                          |                      | ## From selfie ![image](https://n3wstorage.b-cdn.net/n3witalia/selfie.jpg)                                    |
| Sticky Note3         | Sticky Note           | Example image "To Linkedin Headshot"                 |                          |                      | ## To Linkedin Headshot ![image](https://n3wstorage.b-cdn.net/n3witalia/cv_top.jpg)                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API credentials.  
   - Set to trigger on "message" updates.

2. **Create Code node named Sanitaze**  
   - Type: Code  
   - JavaScript code:  
     ```js
     if ($input.first().json.message.from.id !== 11111) { // Replace with your Telegram user ID
       return { unauthorized: true };
     } else {
       return $input.all();
     }
     ```  
   - Connect Telegram Trigger → Sanitaze.

3. **Add Switch node**  
   - Type: Switch  
   - Configure with three outputs:  
     - Text: condition checks if `message.text` exists  
     - Audio: condition checks if `message.voice.file_id` exists  
     - Image: condition checks if `message.photo[0]` exists  
   - Connect Sanitaze → Switch.

4. **Create Telegram node named Get Image**  
   - Type: Telegram  
   - Configure to download file with fileId: `{{$json.message.photo[2].file_id}}` (highest resolution).  
   - Connect Switch (Image output) → Get Image.

5. **Create Set node named Set FTP params**  
   - Type: Set  
   - Add string fields:  
     - `ftp_path`: e.g., `/public_html/images/`  
     - `base_url`: e.g., `https://website.com/images/`  
   - Connect Get Image → Set FTP params.

6. **Create Merge node**  
   - Type: Merge  
   - Mode: Combine  
   - Connect Get Image → Merge (Input 1) and Set FTP params → Merge (Input 2).

7. **Create FTP node named Upload to FTP**  
   - Type: FTP  
   - Credentials: Configure with your FTP server (e.g., BunnyCDN).  
   - Path: `={{$json.ftp_path}}{{$json.result.file_path.split('/').pop()}}`  
   - Operation: Upload  
   - Connect Merge → Upload to FTP.

8. **Create Set node named FTP Image URL**  
   - Type: Set  
   - Assign field `image` with value: `={{ $json.base_url }}{{$json.result.file_path.split('/').pop()}}` (construct accessible URL).  
   - Connect Upload to FTP → FTP Image URL.

9. **Create HTTP Request node named Create Image**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/nano-banana-pro/edit`  
   - Authentication: HTTP Header Auth with header:  
     - Name: `Authorization`  
     - Value: `Key YOURAPIKEY` (replace with your Fal.ai API key)  
   - Headers: `Content-Type: application/json`  
   - Body (JSON):  
     ```json
     {
       "prompt": "Transform this photo into a professional LinkedIn headshot. Keep my face realistic and recognizable. Frame me in a 3/4 angle from the shoulders up, with an open, confident posture and a friendly, professional expression (slight smile).Enhance the lighting to soft, even studio-style illumination. Use a clean, neutral, blurred background in professional tones such as light gray, blue, or beige, with no distractions. Improve sharpness, skin tone, and contrast in a natural way. Maintain clean, professional clothing: blazer or shirt, solid colors, no logos.Overall style: modern, credible, suitable for a LinkedIn profile photo.",
       "image_urls": ["{{$json.image}}"],
       "resolution": "1K",
       "num_images": 1,
       "aspect_ratio": "auto",
       "output_format": "png"
     }
     ```  
   - Connect FTP Image URL → Create Image.

10. **Create Wait node named Wait 60 sec.**  
    - Type: Wait  
    - Amount: 10 seconds  
    - Connect Create Image → Wait 60 sec.

11. **Create HTTP Request node named Get status**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/nano-banana-pro/requests/{{$json.request_id}}/status`  
    - Authentication: same HTTP Header Auth as Create Image.  
    - Connect Wait 60 sec. → Get status.

12. **Create If node named Completed?**  
    - Type: If  
    - Condition: `$json.status` equals `"COMPLETED"`  
    - Connect Get status → Completed?

13. **Loop on False**  
    - Connect Completed? (false) → Wait 60 sec. (to poll again).

14. **Create HTTP Request node named Get Url image**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/nano-banana-pro/requests/{{$json.request_id}}`  
    - Authentication: same HTTP Header Auth.  
    - Connect Completed? (true) → Get Url image.

15. **Create HTTP Request node named Get File image**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{$json.images[0].url}}` (from Get Url image response).  
    - Connect Get Url image → Get File image.

16. **Create Google Drive node named Upload Image**  
    - Type: Google Drive  
    - Credentials: OAuth2 Google Drive account.  
    - Folder ID: your target folder for uploads.  
    - File name: `={{$now.format('yyyyLLddHHmmss')}}-{{$json.images[0].file_name}}`  
    - Connect Get File image → Upload Image.

17. **Create FTP node named Upload to FTP1**  
    - Type: FTP  
    - Credentials: same FTP as before.  
    - Path: `={{$json.ftp_path}}{{$now.format('yyyyLLddHHmmss')}}-{{$json.images[0].file_name}}`  
    - Operation: Upload  
    - Connect Get File image → Upload to FTP1.

18. **Manual Trigger path setup (optional)**  
    - Create Manual Trigger node named When clicking ‘Execute workflow’.  
    - Connect to Set node named Fix Image Url, which sets `image` field to your static image URL.  
    - Connect Fix Image Url → Create Image node.  

19. **Add Sticky Notes** for user instructions:  
    - API key setup  
    - FTP parameters  
    - Telegram user ID configuration  
    - Workflow overview and example images.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                               |
|----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Create an account at [fal.ai](https://fal.ai/) and obtain your API key for Nano Banana Pro API usage.                           | Fal.ai API key setup                          |
| FTP parameters must be configured correctly to enable image uploads and accessible URLs.                                         | FTP setup instructions                        |
| Replace the Telegram user ID in the Sanitaze node with your actual Telegram user ID to restrict access.                          | User authorization for Telegram input        |
| Workflow supports two input methods: Telegram photos or manual image URL trigger.                                                 | Workflow operation modes                       |
| Example images showing the transformation from selfie to LinkedIn headshot are included in sticky notes.                         | Visual workflow context                        |
| Google Drive and FTP credentials need to be set up with proper permissions for file upload nodes.                                | Storage credentials and folder/path settings |
| The workflow currently lacks explicit failure handling for API errors or networking issues; monitoring is recommended.           | Error and edge case considerations            |

---

**Disclaimer:**  
The provided text derives exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.