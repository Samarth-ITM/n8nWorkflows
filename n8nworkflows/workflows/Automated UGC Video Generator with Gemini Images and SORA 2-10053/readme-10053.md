Automated UGC Video Generator with Gemini Images and SORA 2

https://n8nworkflows.xyz/workflows/automated-ugc-video-generator-with-gemini-images-and-sora-2-10053


# Automated UGC Video Generator with Gemini Images and SORA 2

---

### 1. Workflow Overview

This workflow automates the creation of User-Generated Content (UGC) style product videos by combining AI-generated images and video creation services. It is designed for marketing teams or content creators who want to produce short promotional videos based on product and video description prompts. The workflow is triggered via a webhook receiving prompts and API keys, then:

- Generates a product image using Google's Gemini AI image model ("Gemini 2.5 Flash Image").
- Creates a short video from a video prompt using OpenAI's SORA 2 video generation model.
- Polls the video generation status until completion.
- Downloads the completed video.
- Uploads the video to Google Drive.
- Logs the video metadata (product, URL, status, timestamp) in Google Sheets.

Logical blocks:

1.1 Input Reception  
1.2 Image Generation with Gemini AI  
1.3 Image Data Extraction  
1.4 Video Generation with OpenAI SORA 2  
1.5 Video Status Polling and Completion Check  
1.6 Video Download and Upload to Google Drive  
1.7 Logging to Google Sheets

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives an HTTP POST request containing product and video prompts, along with Gemini and OpenAI API keys. Acts as the workflow entry point.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook (HTTP endpoint)  
    - Configuration: Listens on path `create-ugc-video` for POST requests, returns the response from the last node in the chain.  
    - Key inputs expected in JSON body:  
      - `productPrompt` (string): text prompt for image generation  
      - `videoPrompt` (string): text prompt for video generation  
      - `geminiApiKey` (string): API key for Gemini image generation  
      - `openaiApiKey` (string): API key for OpenAI video generation  
    - Output connection: passes webhook data to "Nano Banana" node.  
    - Edge cases: Missing or invalid keys, malformed JSON, unsupported HTTP methods.

---

#### 1.2 Image Generation with Gemini AI

- **Overview:**  
Calls the Gemini AI API to generate an image based on the provided product prompt.

- **Nodes Involved:**  
  - Nano Banana (HTTP Request)

- **Node Details:**  
  - **Nano Banana**  
    - Type: HTTP Request  
    - Configuration:  
      - POST request to Gemini's image generation endpoint `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent` with API key from webhook.  
      - JSON body contains the product prompt text in the required structure.  
    - Uses expressions to dynamically set URL with API key and body with prompt text.  
    - Input: webhook JSON body  
    - Output: Gemini API JSON response including generated image data.  
    - Edge cases: API key invalid or expired, API rate limits, network timeouts, unexpected response formats.

---

#### 1.3 Image Data Extraction

- **Overview:**  
Processes the Gemini AI response to extract the image data in base64 format and passes along relevant prompts and API keys for subsequent steps.

- **Nodes Involved:**  
  - Extract Image (Code Node)

- **Node Details:**  
  - **Extract Image**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Extracts first candidate's image data inline base64 from Gemini response JSON.  
      - Passes through original prompts and OpenAI API key from webhook.  
    - Inputs: output of "Nano Banana" and webhook data accessed via expressions.  
    - Outputs: JSON with `imageBase64`, `productPrompt`, `videoPrompt`, and `openaiApiKey`.  
    - Edge cases: Missing or malformed image data, no candidates returned, expression failures if webhook data missing.

---

#### 1.4 Video Generation with OpenAI SORA 2

- **Overview:**  
Initiates video generation by calling OpenAI's SORA 2 API with the provided video prompt.

- **Nodes Involved:**  
  - SORA 2 (HTTP Request)

- **Node Details:**  
  - **SORA 2**  
    - Type: HTTP Request  
    - Configuration:  
      - POST to `https://api.openai.com/v1/videos` with JSON body specifying model "sora-2", prompt from extracted data, 720x1280 size, 8 seconds length.  
      - Authorization header with Bearer token using OpenAI API key passed from webhook.  
    - Input: Extracted data JSON  
    - Output: Video generation initiation response including video `id`.  
    - Edge cases: Invalid API key, prompt too long/short, API failures, network issues.

---

#### 1.5 Video Status Polling and Completion Check

- **Overview:**  
Polls the OpenAI video generation API to check if the video is ready. If not ready, waits 1 minute and retries.

- **Nodes Involved:**  
  - Get Video (HTTP Request)  
  - Video Complete? (If node)  
  - Wait & Retry (Wait node)

- **Node Details:**  
  - **Get Video**  
    - Type: HTTP Request  
    - Configuration:  
      - GET request to `https://api.openai.com/v1/videos/{videoId}` using video ID from "SORA 2" node.  
      - Authorization header with OpenAI API key.  
    - Input: Output from "SORA 2" or "Wait & Retry".  
    - Output: Video status JSON including `status`.  
    - Edge cases: Video ID invalid, API errors, network timeouts.

  - **Video Complete?**  
    - Type: If Node  
    - Configuration:  
      - Checks if `status` field in JSON equals "completed".  
    - Output: True branch leads to video download; false branch leads to wait node.  
    - Edge cases: Missing status field, unexpected status values.

  - **Wait & Retry**  
    - Type: Wait  
    - Configuration:  
      - Wait time set to 1 minute before retrying "Get Video".  
    - Edge cases: Workflow execution limits if video generation takes too long.

---

#### 1.6 Video Download and Upload to Google Drive

- **Overview:**  
Once the video is completed, downloads the video file from OpenAI and uploads it to Google Drive as an MP4 file.

- **Nodes Involved:**  
  - Download Video File (HTTP Request)  
  - Upload file (Google Drive)

- **Node Details:**  
  - **Download Video File**  
    - Type: HTTP Request  
    - Configuration:  
      - GET request to `https://api.openai.com/v1/videos/{videoId}/content` with Authorization header.  
      - Response expected as a file (binary data).  
    - Input: Video ID from "Video Complete?" node.  
    - Output: Video file binary data.  
    - Edge cases: Authorization errors, file size limits, network issues.

  - **Upload file**  
    - Type: Google Drive  
    - Configuration:  
      - Uploads video file as `{videoId}.mp4` to root folder of Google Drive.  
      - Uses configured Google Drive OAuth2 credentials.  
    - Input: Binary video file from previous node.  
    - Output: Google Drive file metadata including webViewLink.  
    - Edge cases: OAuth token expiration, quota limits, upload failures.

---

#### 1.7 Logging to Google Sheets

- **Overview:**  
Appends a new row to a Google Sheet logging the product prompt, video URL, generation status, and creation timestamp.

- **Nodes Involved:**  
  - Add to Google Sheets

- **Node Details:**  
  - **Add to Google Sheets**  
    - Type: Google Sheets (Append operation)  
    - Configuration:  
      - Appends to the first sheet (gid=0) of a specified Google Sheet document.  
      - Columns: Product prompt, Video URL (Google Drive link), Status, Created timestamp (ISO string).  
      - Uses Google Sheets OAuth2 credentials.  
    - Input: Data from upload node and video status.  
    - Edge cases: API rate limits, sheet access permissions, malformed data.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                              | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                      |
|---------------------|--------------------|----------------------------------------------|--------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Webhook             | Webhook            | Receives POST request with prompts and keys | -                        | Nano Banana             | Receives POST request with prompts and API keys                                                |
| Nano Banana         | HTTP Request       | Generates product image using Gemini AI       | Webhook                  | Extract Image           | Generates product image using Gemini AI from prompt                                            |
| Extract Image       | Code               | Extracts base64 image data from Gemini response | Nano Banana              | SORA 2                  | Extracts base64 image data from Gemini API response                                            |
| SORA 2              | HTTP Request       | Creates video from prompt using OpenAI SORA  | Extract Image            | Get Video               | Creates video from prompt using OpenAI SORA model                                              |
| Get Video           | HTTP Request       | Checks SORA video generation status           | SORA 2, Wait & Retry     | Video Complete?          | Checks SORA video generation status via API call                                              |
| Video Complete?     | If                 | Determines if video rendering is finished     | Get Video                | Download Video File, Wait & Retry | Determines if video rendering is finished or pending                                         |
| Wait & Retry        | Wait               | Pauses 1 minute then retries status check     | Video Complete? (false)  | Get Video               | Pauses 1 minute then checks video status again                                                |
| Download Video File | HTTP Request       | Downloads completed video file from OpenAI    | Video Complete? (true)   | Upload file             | Downloads completed video file from OpenAI API endpoint                                       |
| Upload file         | Google Drive       | Saves video to Google Drive as MP4             | Download Video File      | Add to Google Sheets    | Saves video to Google Drive as MP4 file                                                       |
| Add to Google Sheets| Google Sheets      | Logs video details: product, URL, status, time| Upload file              | -                       | Logs video details: product, URL, status, timestamp                                           |
| Sticky Note         | Sticky Note        | Workflow overview and credits                   | -                        | -                       | # UGC Video Generator Workflow ... Created by [Daniel Shashko](https://www.linkedin.com/in/daniel-shashko/) |
| Sticky Note1        | Sticky Note        | Explanation of webhook node                     | -                        | -                       | Receives POST request with prompts and API keys                                               |
| Sticky Note2        | Sticky Note        | Explanation of Gemini image generation          | -                        | -                       | Generates product image using Gemini AI from prompt                                           |
| Sticky Note3        | Sticky Note        | Explanation of image extraction                  | -                        | -                       | Extracts base64 image data from Gemini API response                                           |
| Sticky Note4        | Sticky Note        | Explanation of SORA 2 video generation           | -                        | -                       | Creates video from prompt using OpenAI SORA model                                             |
| Sticky Note5        | Sticky Note        | Explanation of video status check                 | -                        | -                       | Checks SORA video generation status via API call                                             |
| Sticky Note6        | Sticky Note        | Explanation of video completion decision          | -                        | -                       | Determines if video rendering is finished or pending                                         |
| Sticky Note7        | Sticky Note        | Explanation of wait and retry                      | -                        | -                       | Pauses 1 minute then checks video status again                                               |
| Sticky Note8        | Sticky Note        | Explanation of video file download                 | -                        | -                       | Downloads completed video file from OpenAI API endpoint                                      |
| Sticky Note9        | Sticky Note        | Explanation of video upload                         | -                        | -                       | Saves video to Google Drive as MP4 file                                                      |
| Sticky Note10       | Sticky Note        | Explanation of Google Sheets logging                | -                        | -                       | Logs video details: product, URL, status, timestamp                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `create-ugc-video`  
   - Response Mode: Last node  
   - Purpose: Receive JSON body with `productPrompt`, `videoPrompt`, `geminiApiKey`, `openaiApiKey`.

2. **Add HTTP Request node named "Nano Banana":**  
   - Connect from Webhook  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent?key={{ $json.body.geminiApiKey }}`  
   - Body Content-Type: JSON  
   - Body:  
     ```json
     {
       "contents": [{
         "parts": [{
           "text": "{{ $json.body.productPrompt }}"
         }]
       }]
     }
     ```  
   - Purpose: Call Gemini API to generate image.

3. **Add Code node named "Extract Image":**  
   - Connect from "Nano Banana"  
   - JavaScript:  
     ```js
     const data = $input.first().json;
     const imageData = data.candidates[0].content.parts.find(p => p.inlineData);
     return {
       imageBase64: imageData.inlineData.data,
       productPrompt: $('Webhook').first().json.body.productPrompt,
       videoPrompt: $('Webhook').first().json.body.videoPrompt,
       openaiApiKey: $('Webhook').first().json.body.openaiApiKey
     };
     ```  
   - Purpose: Extract base64 image and pass prompts and keys.

4. **Add HTTP Request node named "SORA 2":**  
   - Connect from "Extract Image"  
   - Method: POST  
   - URL: `https://api.openai.com/v1/videos`  
   - Headers:  
     - Authorization: `Bearer {{ $json.openaiApiKey }}`  
     - Content-Type: `application/json`  
   - Body:  
     ```json
     {
       "model": "sora-2",
       "prompt": "{{ $json.videoPrompt }}",
       "size": "720x1280",
       "seconds": "8"
     }
     ```  
   - Purpose: Initiate video generation.

5. **Add HTTP Request node named "Get Video":**  
   - Connect from "SORA 2" (and later from Wait node)  
   - Method: GET  
   - URL: `https://api.openai.com/v1/videos/{{ $('SORA 2').item.json.id }}`  
   - Headers:  
     - Authorization: `Bearer {{ $('Extract Image').item.json.openaiApiKey }}`  
   - Purpose: Poll video status.

6. **Add If node named "Video Complete?":**  
   - Connect from "Get Video"  
   - Condition: Check if `{{$json.status}}` equals `completed`  
   - True output to "Download Video File" node  
   - False output to "Wait & Retry" node

7. **Add Wait node named "Wait & Retry":**  
   - Connect from "Video Complete?" false branch  
   - Wait Duration: 1 minute  
   - After wait, connect back to "Get Video" node  
   - Purpose: Retry polling until video is ready.

8. **Add HTTP Request node named "Download Video File":**  
   - Connect from "Video Complete?" true branch  
   - Method: GET  
   - URL: `https://api.openai.com/v1/videos/{{ $json.id }}/content`  
   - Headers:  
     - Authorization: `Bearer {{ $('Webhook').first().json.body.openaiApiKey }}`  
   - Response Format: File (binary)  
   - Purpose: Download the completed video file.

9. **Add Google Drive node named "Upload file":**  
   - Connect from "Download Video File"  
   - Operation: Upload file  
   - File Name: `{{ $json.id }}.mp4`  
   - Folder: Root folder or specify as needed  
   - Credentials: Google Drive OAuth2  
   - Purpose: Save video to Google Drive.

10. **Add Google Sheets node named "Add to Google Sheets":**  
    - Connect from "Upload file"  
    - Operation: Append row to sheet with:  
      - Status: `{{ $json.status }}`  
      - Created: `{{ new Date().toISOString() }}`  
      - Product: `{{ $('Extract Image').item.json.productPrompt }}`  
      - Video URL: `{{ $('Upload file').item.json.webViewLink }}`  
    - Sheet Name and Document ID: Set to your Google Sheet  
    - Credentials: Google Sheets OAuth2  
    - Purpose: Log video metadata.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow created by Daniel Shashko. LinkedIn profile: https://www.linkedin.com/in/daniel-shashko/             | Author credit                                                                                      |
| This workflow integrates Google Gemini image generation and OpenAI SORA 2 video API to automate UGC video creation. | Project overview                                                                                   |
| Requires Google Sheets OAuth and Google Drive OAuth credentials configured in n8n before use.                 | Credential requirements                                                                            |
| API keys for Gemini and OpenAI must be passed securely via webhook JSON payload.                              | Security note                                                                                      |
| Gemini API endpoint documentation: https://developers.generativelanguage.googleapis.com                       | Reference for Gemini API usage                                                                     |
| OpenAI Video API documentation (SORA 2): https://platform.openai.com/docs/api-reference/videos               | Reference for OpenAI video generation API                                                         |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---