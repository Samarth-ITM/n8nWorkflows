Generate UGC Ads from Google Sheets with Fal.ai Models (nano-banana, WAN2.2, Veo3)

https://n8nworkflows.xyz/workflows/generate-ugc-ads-from-google-sheets-with-fal-ai-models--nano-banana--wan2-2--veo3--8205


# Generate UGC Ads from Google Sheets with Fal.ai Models (nano-banana, WAN2.2, Veo3)

### 1. Workflow Overview

This workflow automates the generation of User Generated Content (UGC) style ads by integrating Google Sheets data with Fal.ai’s AI-powered models (nano-banana, WAN2.2, Veo3) for image and video creation. The primary use case is to create photorealistic product images and corresponding authentic, casual video scenes based on prompts and image URLs stored in Google Sheets, then upload and update these assets back into Google Drive and Sheets for easy access and distribution.

The workflow is logically divided into two main zones:

**1.1 Zone 1: Image Creation**  
- Triggered manually  
- Retrieves image URLs and prompts from Google Sheets  
- Calls Fal.ai’s nano-banana model (via OpenRouter’s Gemini-2.5-flash-image-preview free API) to generate enhanced product images  
- Uploads generated images to Google Drive  
- Updates Google Sheets with new image URLs  

**1.2 Zone 2: Video Generation**  
- Analyzes the generated image using OpenAI to extract scene and character information  
- Uses the Veo3 Fal.ai model to generate a UGC-style 8-second video based on detailed scene descriptions  
- Monitors the video generation status with polling and waits  
- Uploads final videos to Google Drive  
- Updates Google Sheets with video URLs  

Additional nodes handle data formatting, batching, and error handling through conditional checks and waits.

---

### 2. Block-by-Block Analysis

#### 2.1 Zone 1: Create Image

**Overview:**  
This block focuses on fetching prompt and image URLs from Google Sheets, transforming them to direct-access URLs, generating a new product image using Fal.ai’s nano-banana model via OpenRouter’s Gemini API, and uploading the result to Google Drive. Finally, it updates the Google Sheet with the new image URL.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get Data (Google Sheets)  
- Edit Fields (Set)  
- Call Fal.ai API (nannoBanana) (HTTP Request)  
- Get image status (HTTP Request)  
- If (Conditional)  
- Wait (Wait)  
- Get the image (HTTP Request)  
- Analyze image (OpenAI)  
- setImgeURL (Set)  
- CreateImagebyOpernRouter (gemini-2.5-flash-image-preview:free) (HTTP Request)  
- wait20sec (Wait)  
- setBase64data (Code)  
- Convert to File (ConvertToFile)  
- uploadImagetoGdrive (Google Drive)  
- updateImageURL (Google Sheets)  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual workflow execution  
  - No parameters  
  - Outputs to Get Data  

- **Get Data (Google Sheets)**  
  - Retrieves a row from a specific sheet in Google Sheets containing prompt, product image URL, and other metadata  
  - Uses OAuth2 credentials for Google Sheets  
  - Filters applied to specific columns  
  - Outputs to Edit Fields  

- **Edit Fields (Set)**  
  - Extracts Google Drive file IDs from URLs in `presenter` and `product` fields  
  - Converts these IDs into direct download URLs for Google Drive images  
  - Allows downstream nodes to access image data in direct URL form  
  - Outputs to Call Fal.ai API (nannoBanana)  

- **Call Fal.ai API (nannoBanana) (HTTP Request)**  
  - Calls Fal.ai endpoint for nano-banana model to generate an enhanced image based on prompt and product image URL  
  - Uses HTTP Header authentication with Fal AI API key  
  - Sends JSON body with prompt, image URLs, number of images, and output format  
  - Outputs to Get image status  

- **Get image status (HTTP Request)**  
  - Polls the status URL returned from Fal.ai to check if image generation is complete  
  - Uses Fal AI authentication  
  - Sends GET requests repeatedly as triggered by Wait and If nodes  
  - Outputs to If node  

- **If (Conditional)**  
  - Checks if image generation status equals “COMPLETED”  
  - If yes, proceeds to Get the image; if no, triggers Wait to delay and retry Get image status  
  - Implements polling logic for asynchronous image generation  

- **Wait (Wait)**  
  - Waits 10 seconds before retrying image status check  
  - Prevents rate-limiting and excessive requests to Fal.ai API  

- **Get the image (HTTP Request)**  
  - Retrieves the generated image data from Fal.ai once status is completed  
  - Uses Fal AI authentication  
  - Outputs to Analyze image  

- **Analyze image (OpenAI)**  
  - Sends image URL to OpenAI GPT-4o model to analyze and classify the image as product, character, or both  
  - Extracts brand, color scheme, text, product type, character details, visual description, etc.  
  - Outputs structured JSON to Describe Each Scene for Video  

- **setImgeURL (Set)**  
  - Similar to Edit Fields, extracts and converts Google Drive URLs from JSON inputs to direct image URLs  
  - Outputs to CreateImagebyOpernRouter  

- **CreateImagebyOpernRouter (gemini-2.5-flash-image-preview:free) (HTTP Request)**  
  - Calls OpenRouter Gemini API to create an image preview based on prompt and product image URL  
  - Uses OpenRouter HTTP Header authentication  
  - Outputs to wait20sec  

- **wait20sec (Wait)**  
  - Pauses for 10 seconds (despite node name) to allow image processing to complete  
  - Outputs to setBase64data  

- **setBase64data (Code)**  
  - Extracts base64 image data and MIME type from data URI returned by Gemini API  
  - Prepares binary data for file upload  
  - Outputs to Convert to File  

- **Convert to File (ConvertToFile)**  
  - Converts base64 string into a binary file object in n8n  
  - Sets file name and MIME type dynamically based on extracted data  
  - Outputs to uploadImagetoGdrive  

- **uploadImagetoGdrive (Google Drive)**  
  - Uploads the generated image file to a specific folder in Google Drive  
  - Uses OAuth2 Google Drive credentials  
  - Outputs to updateImageURL  

- **updateImageURL (Google Sheets)**  
  - Updates the Google Sheet row with the new image’s webViewLink URL  
  - Matches rows by product field  
  - Ensures Google Sheet is synchronized with generated image assets  

**Potential Failure Points:**  
- Google Sheets or Drive OAuth token expiration or permission issues  
- HTTP request failures or timeouts contacting Fal.ai or OpenRouter APIs  
- Image generation delays causing polling timeout or infinite loops  
- Incorrect or malformed URLs causing failure in regex extraction  
- API quota limits on OpenAI or Fal.ai  
- Data parsing or expression errors in code nodes  

---

#### 2.2 Zone 2: Generate Video

**Overview:**  
This block generates UGC-style videos based on the analyzed image data and scene description. It uses OpenAI to parse and generate detailed scene prompts, calls Fal.ai’s Veo3 model to create videos, manages video processing status, uploads videos to Google Drive, and updates Google Sheets with video URLs.

**Nodes Involved:**  
- Analyze image (OpenAI) [shared with Zone 1]  
- Describe Each Scene for Video (Langchain Agent)  
- OpenAI Chat Model1  
- Structured Output Parser2  
- Veo3 (HTTP Request)  
- Loop Over Items (SplitInBatches)  
- Get the video (HTTP Request)  
- Get the video status (HTTP Request)  
- Video status (Switch)  
- Wait for the video (Wait)  
- HTTP Request (for video download)  
- uploadImagetoGdrive1 (Google Drive)  
- updateVideoURL (Google Sheets)  

**Node Details:**

- **Analyze image (OpenAI)**  
  - Shared with Zone 1, triggers scene analysis to produce structured data for video prompt generation  
  - Outputs to Describe Each Scene for Video  

- **Describe Each Scene for Video (Langchain Agent)**  
  - Uses a custom prompt instructing an AI agent to expand scene input into a realistic 5-second UGC-style video description including characters, background, camera movement, object movement, and sound design  
  - Outputs natural language description to Veo3  

- **OpenAI Chat Model1 + Structured Output Parser2**  
  - Chat model processes scene inputs to structured JSON with characters, scene description, camera movement, object movements, and sound effects  
  - Output parser validates and structures the JSON for downstream use  
  - Sends structured output back to Describe Each Scene for Video, supporting iterative prompt refinement if needed  

- **Veo3 (HTTP Request)**  
  - Calls Fal.ai Veo3 endpoint to convert the scene description and image URL into an 8-second video with audio and 720p resolution  
  - Uses Fal AI HTTP Header authentication  
  - Outputs video generation request response including status and URLs  

- **Loop Over Items (SplitInBatches)**  
  - Manages batch processing for video generation requests and status polling  
  - Splits data into manageable chunks for parallel or sequential processing  

- **Get the video (HTTP Request)**  
  - Retrieves the generated video from Fal.ai once ready  
  - Uses Fal AI authentication  
  - Outputs to Loop Over Items and Wait for the video  

- **Get the video status (HTTP Request)**  
  - Polls the video status URL to check if video is “COMPLETED”, “IN_PROGRESS”, or “IN_QUEUE”  
  - Uses Fal AI authentication  
  - Outputs to Video status switch  

- **Video status (Switch)**  
  - Routes execution based on video status:  
    - COMPLETED → proceed to Get the video  
    - IN_PROGRESS or IN_QUEUE → Wait and retry status check  

- **Wait for the video (Wait)**  
  - Waits a default time before retrying the video status check  
  - Prevents excessive API calls and rate-limiting  

- **HTTP Request (video download)**  
  - Downloads the video file or accesses the video URL for upload  
  - Authenticated with Fal AI credentials  

- **uploadImagetoGdrive1 (Google Drive)**  
  - Uploads the finished video file to a designated Google Drive folder  
  - Uses OAuth2 credentials for Google Drive  

- **updateVideoURL (Google Sheets)**  
  - Updates the corresponding Google Sheet row with the video’s webViewLink URL  
  - Matches rows by product field to ensure accurate syncing  

**Potential Failure Points:**  
- Video generation delays or failures at Fal.ai API  
- Polling loops stuck if status does not update due to network or API issues  
- Authentication errors with Fal AI or Google APIs  
- Batch processing errors if input data is incomplete or malformed  
- Output parsing errors if OpenAI returns unexpected JSON structure  
- Rate limits on API calls leading to throttling or errors  

---

### 3. Summary Table

| Node Name                                      | Node Type                  | Functional Role                                | Input Node(s)                          | Output Node(s)                        | Sticky Note                                                                                 |
|------------------------------------------------|----------------------------|-----------------------------------------------|--------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’                | Manual Trigger             | Workflow manual start trigger                  |                                      | Get Data                            | 🟨 Zone 1: Create Image - Step 1                                                           |
| Get Data                                        | Google Sheets              | Fetch prompt and image URLs from sheet         | When clicking ‘Execute workflow’     | Edit Fields                        | 🟨 Zone 1: Create Image - Step 2                                                           |
| Edit Fields                                     | Set                        | Extract and convert Google Drive URLs          | Get Data                            | Call Fal.ai API (nannoBanana)      | 🟨 Zone 1: Create Image - Step 3                                                           |
| Call Fal.ai API (nannoBanana)                   | HTTP Request               | Call Fal.ai nano-banana model for image gen    | Edit Fields                        | Get image status                   | 🟨 Zone 1: Create Image - Step 4                                                           |
| Get image status                                | HTTP Request               | Poll image generation status                     | Call Fal.ai API (nannoBanana)        | If                               | 🟨 Zone 1: Create Image - Step 5                                                           |
| If                                              | Conditional                | Check if image generation is completed          | Get image status                   | Get the image / Wait               | 🟨 Zone 1: Create Image - Step 6                                                           |
| Wait                                            | Wait                       | Delay before retrying image status check        | If                                | Get image status                   | 🟨 Zone 1: Create Image - Step 6                                                           |
| Get the image                                   | HTTP Request               | Fetch generated image data                       | If                               | Analyze image                     | 🟨 Zone 1: Create Image - Step 7                                                           |
| Analyze image                                  | OpenAI (Image Analysis)    | Analyze image content for scene and character  | Get the image                     | Describe Each Scene for Video      | 🟨 Zone 1: Create Image & 🟫 Zone 2: Generate Video                                         |
| setImgeURL                                      | Set                        | Extract direct URLs from Google Drive links     | Get Data1                        | CreateImagebyOpernRouter           | 🟨 Zone 1: Create Image                                                                     |
| Get Data1                                       | Google Sheets              | Get additional data for image creation          | When clicking ‘Execute workflow’     | setImgeURL                       | 🟨 Zone 1: Create Image                                                                     |
| CreateImagebyOpernRouter (gemini-2.5-flash-image-preview:free) | HTTP Request | Create enhanced image via OpenRouter Gemini API | setImgeURL                      | wait20sec                        | 🟨 Zone 1: Create Image                                                                     |
| wait20sec                                       | Wait                       | Wait for image processing completion            | CreateImagebyOpernRouter           | setBase64data                    | 🟨 Zone 1: Create Image                                                                     |
| setBase64data                                   | Code                       | Extract base64 and MIME from API response       | wait20sec                        | Convert to File                 | 🟨 Zone 1: Create Image                                                                     |
| Convert to File                                 | ConvertToFile              | Convert base64 string to binary file             | setBase64data                    | uploadImagetoGdrive             | 🟨 Zone 1: Create Image                                                                     |
| uploadImagetoGdrive                             | Google Drive               | Upload generated image to Drive                   | Convert to File                  | updateImageURL                 | 🟨 Zone 1: Create Image                                                                     |
| updateImageURL                                 | Google Sheets              | Update sheet with new image URL                   | uploadImagetoGdrive              |                                 | 🟨 Zone 1: Create Image                                                                     |
| Describe Each Scene for Video                   | Langchain Agent            | Generate detailed UGC-style video scene prompt  | Analyze image                   | Veo3                            | 🟫 Zone 2: Generate Video                                                                   |
| OpenAI Chat Model1                             | Langchain OpenAI Chat Model| Process scene input into structured JSON         |                                 | Structured Output Parser2       | 🟫 Zone 2: Generate Video                                                                   |
| Structured Output Parser2                      | Langchain Output Parser    | Validate and structure JSON output                | OpenAI Chat Model1              | Describe Each Scene for Video    | 🟫 Zone 2: Generate Video                                                                   |
| Veo3                                           | HTTP Request               | Call Fal.ai Veo3 model to generate video          | Describe Each Scene for Video    | Loop Over Items                | 🟫 Zone 2: Generate Video                                                                   |
| Loop Over Items                                | SplitInBatches             | Batch processing for video generation             | Veo3 / Get the video            | HTTP Request / Wait for the video | 🟫 Zone 2: Generate Video                                                                   |
| Get the video                                  | HTTP Request               | Retrieve generated video data                      | Video status / Loop Over Items  | Loop Over Items                | 🟫 Zone 2: Generate Video                                                                   |
| Get the video status                           | HTTP Request               | Poll video generation status                        | Wait for the video             | Video status                   | 🟫 Zone 2: Generate Video                                                                   |
| Video status                                  | Switch                     | Route execution based on video status              | Get the video status           | Get the video / Wait for the video | 🟫 Zone 2: Generate Video                                                                   |
| Wait for the video                            | Wait                       | Wait before polling video status again             | Loop Over Items / Video status  | Get the video status           | 🟫 Zone 2: Generate Video                                                                   |
| HTTP Request (video download)                 | HTTP Request               | Download or access video file                        | Loop Over Items                | uploadImagetoGdrive1          | 🟫 Zone 2: Generate Video                                                                   |
| uploadImagetoGdrive1                           | Google Drive               | Upload generated video to Drive                      | HTTP Request (video download)  | updateVideoURL                | 🟫 Zone 2: Generate Video                                                                   |
| updateVideoURL                                | Google Sheets              | Update sheet with new video URL                      | uploadImagetoGdrive1           |                              | 🟫 Zone 2: Generate Video                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Name: “When clicking ‘Execute workflow’”  
   - No parameters. This triggers the workflow manually.

2. **Add Google Sheets node:**  
   - Name: “Get Data”  
   - Operation: Read rows from sheet “nanoBanana” (sheet ID 658195685) in spreadsheet ID `1-oeo9nsFGfDUTh2OVXo1bHeoY1JBvSQx5IbDAH37epY`  
   - Filter by column “video_url” if needed  
   - Use OAuth2 Google Sheets credentials  
   - Connect output from Manual Trigger.

3. **Add Set node:**  
   - Name: “Edit Fields”  
   - Purpose: Extract Google Drive file IDs from `presenter` and `product` fields using regex and construct direct download URLs  
   - Keep other fields unchanged  
   - Connect from “Get Data”.

4. **Add HTTP Request node:**  
   - Name: “Call Fal.ai API (nannoBanana)”  
   - URL: `https://queue.fal.run/fal-ai/{{ $json.model }}/edit`  
   - Method: POST  
   - Body (JSON):  
     ```json
     {
       "prompt": "{{ $json.prompt }}",
       "image_urls": ["{{ $json.product }}"],
       "num_images": 1,
       "output_format": "jpeg"
     }
     ```  
   - Auth: HTTP Header with Fal AI API key  
   - Connect from “Edit Fields”.

5. **Add HTTP Request node:**  
   - Name: “Get image status”  
   - URL: `={{ $json.status_url }}` (from previous response)  
   - Method: GET  
   - Auth: Fal AI HTTP Header  
   - Connect from “Call Fal.ai API (nannoBanana)”.

6. **Add If node:**  
   - Name: “If”  
   - Condition: `$json.status` equals `COMPLETED`  
   - Connect from “Get image status”.

7. **Add Wait node:**  
   - Name: “Wait”  
   - Delay: 10 seconds  
   - Connect from False branch of “If” to “Get image status” (loop for polling).

8. **Add HTTP Request node:**  
   - Name: “Get the image”  
   - URL: `https://queue.fal.run/fal-ai/nano-banana/requests/{{ $json.request_id }}`  
   - Method: GET  
   - Auth: Fal AI HTTP Header  
   - Connect from True branch of “If”.

9. **Add OpenAI node:**  
   - Name: “Analyze image”  
   - Model: GPT-4o  
   - Task: Analyze image URL, return JSON describing product or character details  
   - Input: Image URL extracted or direct download URL from “Get the image”  
   - Auth: OpenAI API credentials  
   - Connect from “Get the image”.

10. **Add Google Sheets node:**  
    - Name: “Get Data1”  
    - Reads sheet “Gemini” (sheet ID 997043272) for additional prompt and image URL data  
    - Auth: Google Sheets OAuth2  
    - Connect from Manual Trigger (parallel branch).

11. **Add Set node:**  
    - Name: “setImgeURL”  
    - Same regex extraction for direct Google Drive URLs on `presenter` and `product` fields  
    - Connect from “Get Data1”.

12. **Add HTTP Request node:**  
    - Name: “CreateImagebyOpernRouter (gemini-2.5-flash-image-preview:free)”  
    - URL: `https://openrouter.ai/api/v1/chat/completions`  
    - Method: POST  
    - Body: JSON with model `google/gemini-2.5-flash-image-preview:free`, prompt text, and image URL in message payload  
    - Auth: OpenRouter HTTP Header  
    - Connect from “setImgeURL”.

13. **Add Wait node:**  
    - Name: “wait20sec”  
    - Delay: 10 seconds  
    - Connect from “CreateImagebyOpernRouter”.

14. **Add Code node:**  
    - Name: “setBase64data”  
    - JavaScript to parse base64 image data, mime type, and file extension from Gemini API response  
    - Outputs object with `data`, `mimeType`, `fileName`  
    - Connect from “wait20sec”.

15. **Add ConvertToFile node:**  
    - Name: “Convert to File”  
    - Converts base64 string to binary file object  
    - Set fileName and mimeType dynamically from previous node output  
    - Connect from “setBase64data”.

16. **Add Google Drive node:**  
    - Name: “uploadImagetoGdrive”  
    - Upload file to folder ID `1WUzYyF-Uo45wCLQaQRAuhsvCYC0lHJ9O` in “My Drive”  
    - Auth: Google Drive OAuth2  
    - Connect from “Convert to File”.

17. **Add Google Sheets node:**  
    - Name: “updateImageURL”  
    - Append or update row in “Gemini” sheet with new `img_url` as uploaded file’s webViewLink  
    - Match by `product` column  
    - Auth: Google Sheets OAuth2  
    - Connect from “uploadImagetoGdrive”.

---

**Video Generation Branch:**

18. **Add Langchain Agent node:**  
    - Name: “Describe Each Scene for Video”  
    - Prompt: Detailed instructions to expand scene input into 5-second UGC-style video description including characters, background, camera movement, movement, and sound design  
    - Connect from “Analyze image”.

19. **Add Langchain OpenAI Chat Model node:**  
    - Name: “OpenAI Chat Model1”  
    - Model: GPT-4o  
    - Connect from “Describe Each Scene for Video”.

20. **Add Langchain Output Parser node:**  
    - Name: “Structured Output Parser2”  
    - Schema to parse characters array, scene description, camera movement, object movements, sound effects  
    - Connect from “OpenAI Chat Model1”.

21. **Connect “Structured Output Parser2” back to “Describe Each Scene for Video”**  
    - Supports iterative refinement of scene prompt.

22. **Add HTTP Request node:**  
    - Name: “Veo3”  
    - POST to `https://queue.fal.run/fal-ai/veo3/image-to-video`  
    - Body: JSON with prompt constructed from structured output fields, image URL, duration 8s, generate audio true, resolution 720p  
    - Auth: Fal AI HTTP Header  
    - Connect from “Describe Each Scene for Video”.

23. **Add SplitInBatches node:**  
    - Name: “Loop Over Items”  
    - Batch size default (1 or more)  
    - Connect from “Veo3”.

24. **Add HTTP Request node:**  
    - Name: “Get the video”  
    - URL: `={{ $('Loop Over Items').item.json.response_url }}`  
    - Auth: Fal AI HTTP Header  
    - Connect from “Loop Over Items”.

25. **Add HTTP Request node:**  
    - Name: “Get the video status”  
    - URL: `={{ $('Loop Over Items').item.json.status_url }}`  
    - Auth: Fal AI HTTP Header  
    - Connect from “Wait for the video”.

26. **Add Switch node:**  
    - Name: “Video status”  
    - Routes based on `status` field: COMPLETED, IN_PROGRESS, IN_QUEUE  
    - COMPLETED → “Get the video”  
    - IN_PROGRESS/IN_QUEUE → “Wait for the video” (loop)  

27. **Add Wait node:**  
    - Name: “Wait for the video”  
    - Default delay (e.g. 10 seconds)  
    - Connect from “Video status” for IN_PROGRESS/IN_QUEUE and to “Get the video status”.

28. **Add HTTP Request node:**  
    - Name: “HTTP Request” (video download)  
    - URL: `={{ $json.video.url }}`  
    - Auth: Fal AI HTTP Header  
    - Connect from “Loop Over Items”.

29. **Add Google Drive node:**  
    - Name: “uploadImagetoGdrive1”  
    - Upload video file to Drive folder `1WUzYyF-Uo45wCLQaQRAuhsvCYC0lHJ9O`  
    - Auth: Google Drive OAuth2  
    - Connect from “HTTP Request”.

30. **Add Google Sheets node:**  
    - Name: “updateVideoURL”  
    - Append or update row with new `video_url` in “Gemini” sheet matching by `product`  
    - Auth: Google Sheets OAuth2  
    - Connect from “uploadImagetoGdrive1”.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                     | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Product Image and Video previews shown in sticky notes with drive thumbnails for visual references of outputs.                                                                 | Google Drive thumbnails linked in sticky notes nodes                                                        |
| Workflow uses Fal.ai models nano-banana, WAN2.2 (disabled in this workflow), and Veo3 for AI-driven content generation.                                                         | Fal.ai API documentation: https://fal.ai/docs                                                                 |
| OpenRouter API used for Gemini-2.5-flash-image-preview free image generation.                                                                                                   | OpenRouter AI: https://openrouter.ai                                                                         |
| OpenAI GPT-4o used for image analysis and scene description generation.                                                                                                        | OpenAI API docs: https://platform.openai.com/docs                                                             |
| Google Sheets and Google Drive OAuth2 credentials required with appropriate file and folder access.                                                                            | Google API documentation: https://developers.google.com/apis-explorer                                        |
| Polling and waiting implemented to handle asynchronous AI model processing and avoid rate limits or infinite loops.                                                           |                                                                                                              |
| Workflow is tagged as an official n8n template (n8n_official_template).                                                                                                         | n8n community templates: https://n8n.io/community/templates                                                   |

---

**Disclaimer:**  
This document is generated from an automated n8n workflow export and respects content policies. All data processed is legal and public. The workflow orchestrates AI-powered media creation and integration with Google services for marketing use cases.