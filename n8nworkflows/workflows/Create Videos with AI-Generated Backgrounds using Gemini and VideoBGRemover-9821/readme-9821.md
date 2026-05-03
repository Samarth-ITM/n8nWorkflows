Create Videos with AI-Generated Backgrounds using Gemini and VideoBGRemover

https://n8nworkflows.xyz/workflows/create-videos-with-ai-generated-backgrounds-using-gemini-and-videobgremover-9821


# Create Videos with AI-Generated Backgrounds using Gemini and VideoBGRemover

### 1. Workflow Overview

This workflow automates the creation of videos composited on AI-generated backgrounds. It leverages Gemini’s Nano Banana model for generating photorealistic backgrounds from text prompts and VideoBGRemover's API to remove foreground video backgrounds and composite the video onto the generated background. The final output is saved to Google Drive with shareable links.

**Target Use Cases:**  
- Marketing videos with custom AI-generated environments  
- Product demos or AI avatars placed in tailored scenes  
- Social media content featuring unique backgrounds  
- Automated batch processing via webhook or manual triggers  

**Logical Blocks:**  
- **1.1 Input Reception:** Accepts video URL, background prompt, and aspect ratio via webhook or manual input  
- **1.2 AI Background Generation:** Calls Gemini Nano Banana to produce a background image from prompt text  
- **1.3 Background Image Handling:** Saves generated image to Google Drive and makes it publicly accessible  
- **1.4 Video Background Removal & Composition:** Uploads video to VideoBGRemover, triggers composition with AI background, polls for job completion  
- **1.5 Final Output:** Downloads composed video, uploads it to Google Drive, and prepares response payload  
- **1.6 Response Handling:** Sends response back via webhook or prepares manual output  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives input either via a webhook POST request or manual trigger with sample data. Extracts and normalizes necessary parameters.

**Nodes Involved:**  
- Webhook Trigger  
- Manual Trigger  
- Extract Webhook Data  
- Sample Inputs (Edit Here)  
- Merge Triggers  

**Node Details:**  

- **Webhook Trigger**  
  - Type: webhook  
  - Role: Entry point for automated POST requests containing video URL, background prompt, and optional aspect ratio  
  - Config: POST method, path `ai-background-gen`, response mode set to respond from a later node  
  - Inputs: none  
  - Outputs: to Extract Webhook Data  
  - Failure: invalid/missing parameters, HTTP errors  

- **Manual Trigger**  
  - Type: manual trigger  
  - Role: Allows manual workflow execution for testing  
  - Inputs: none  
  - Outputs: to Sample Inputs (Edit Here)  

- **Extract Webhook Data**  
  - Type: set node  
  - Role: Extracts input data from webhook body or directly from JSON; sets default aspect ratio to "1:1"  
  - Expressions: Uses `{{$json.body?.video_url ?? $json.video_url}}` pattern for safe extraction  
  - Outputs: normalized JSON with keys: `video_url`, `background_prompt`, `aspect_ratio`, `source: "webhook"`  
  - Inputs: from Webhook Trigger  
  - Outputs: to Merge Triggers  

- **Sample Inputs (Edit Here)**  
  - Type: set node  
  - Role: Provides default values for manual testing  
  - Fields preset: example video URL, detailed background prompt, aspect ratio "16:9", source "manual"  
  - Outputs: to Merge Triggers  

- **Merge Triggers**  
  - Type: merge (append mode)  
  - Role: Combines input from webhook or manual trigger for unified downstream processing  
  - Inputs: from Extract Webhook Data and Sample Inputs  
  - Outputs: to 1. Generate Background Image (Gemini)  

---

#### 1.2 AI Background Generation

**Overview:**  
Generates a high-quality background image from the provided text prompt using Gemini's Nano Banana model and extracts the image data for further use.

**Nodes Involved:**  
- 1. Generate Background Image (Gemini)  
- Extract Image Data  
- 2. Save Background Image to Drive  
- 3. Make Background Image Public  

**Node Details:**  

- **1. Generate Background Image (Gemini)**  
  - Type: HTTP Request  
  - Role: Sends POST request to Gemini API v1beta image generation endpoint  
  - Config:  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent`  
    - Headers: `x-goog-api-key` from environment variable `$vars.GEMINI_KEY`, `Content-Type: application/json`  
    - Body: JSON with prompt text, requested aspect ratio, specifying image response modality  
  - Inputs: from Merge Triggers  
  - Outputs: to Extract Image Data  
  - Failures: API key invalid, quota exceeded, malformed prompt, network errors  

- **Extract Image Data**  
  - Type: Code (JavaScript)  
  - Role: Parses Gemini response to extract base64 PNG image, converts to binary buffer, and creates a data URL for downstream use  
  - Key Expressions: accesses `$input.item.json.candidates[0].content.parts[0].inlineData.data` and mimeType  
  - Outputs: JSON with keys `background_image_data`, `background_image_mime`, `background_image_url`; binary data field for Google Drive upload  
  - Inputs: from Gemini HTTP Request  
  - Outputs: to Save Background Image to Drive  

- **2. Save Background Image to Drive**  
  - Type: Google Drive  
  - Role: Uploads generated image to Google Drive root folder  
  - Config:  
    - Filename dynamically generated from prompt snippet and timestamp  
    - Upload binary data from previous node  
    - Simplified output enabled  
  - Inputs: from Extract Image Data  
  - Outputs: to Make Background Image Public  
  - Failures: Google OAuth expired, quota exceeded, permission denied  

- **3. Make Background Image Public**  
  - Type: Google Drive (Share file)  
  - Role: Sets file permission to "anyone with link can view"  
  - Inputs: from Save Background Image to Drive (uses file ID)  
  - Outputs: to Create Job (Upload Video)  
  - Failures: permission errors, invalid file ID  

---

#### 1.3 Video Background Removal & Composition

**Overview:**  
Uploads the foreground video to VideoBGRemover, starts the AI composition process with the generated background image, and polls for job completion.

**Nodes Involved:**  
- 4. Create Job (Upload Video)  
- 5. Start Composition (AI Background)  
- 6. Check Job Status  
- Is Complete? (If)  
- Has Failed? (If)  
- Wait 20s  
- Build Error Response  

**Node Details:**  

- **4. Create Job (Upload Video)**  
  - Type: HTTP Request  
  - Role: Sends video URL to VideoBGRemover API to create a new processing job  
  - Config:  
    - URL: `https://api.videobgremover.com/api/v1/jobs`  
    - Headers: `X-API-Key` from `$vars.VIDEOBGREMOVER_KEY`, `Content-Type: application/json`  
    - Body: JSON with `video_url` from merged input  
  - Inputs: from Make Background Image Public  
  - Outputs: to Start Composition (AI Background)  
  - Failure: invalid video URL, API key errors, rate limits  

- **5. Start Composition (AI Background)**  
  - Type: HTTP Request  
  - Role: Starts composition job with background image URL (public Google Drive link) and composition template "centered"  
  - Config:  
    - URL constructed using job ID from previous node  
    - JSON body specifies composition options and export preset  
    - Headers: same API key and content type  
  - Inputs: from Create Job  
  - Outputs: to Check Job Status  

- **6. Check Job Status**  
  - Type: HTTP Request  
  - Role: Polls VideoBGRemover API for job status using job ID  
  - Config:  
    - URL constructed with job ID  
    - Header with API key  
  - Inputs: from Start Composition and from Wait 20s (loop)  
  - Outputs: to If nodes Is Complete? and Has Failed?  

- **Is Complete?**  
  - Type: If node  
  - Condition: Checks if job status equals "completed"  
  - True output: proceeds to Download Video  
  - False output: passes to Has Failed?  

- **Has Failed?**  
  - Type: If node  
  - Condition: Checks if job status equals "failed"  
  - True output: Build Error Response  
  - False output: Wait 20s (to retry polling)  

- **Wait 20s**  
  - Type: Wait node  
  - Role: Delays workflow for 20 seconds before re-polling job status to avoid API throttling  

- **Build Error Response**  
  - Type: Set node  
  - Role: Creates a structured error response JSON with job ID, error message, and source  
  - Inputs: from Has Failed?  

---

#### 1.4 Final Output Handling

**Overview:**  
Downloads the processed video from VideoBGRemover, uploads it to Google Drive, compiles a success response, and sends it back via webhook or manual output.

**Nodes Involved:**  
- 7. Download Video  
- 8. Upload Video to Drive  
- Build Success Response  
- From Webhook? (If)  
- Respond to Webhook  
- Manual Test Complete  

**Node Details:**  

- **7. Download Video**  
  - Type: HTTP Request  
  - Role: Downloads the processed video file using the URL from job status response  
  - Config: Response format set to "file" to handle binary data  
  - Inputs: from Is Complete?  

- **8. Upload Video to Drive**  
  - Type: Google Drive  
  - Role: Uploads downloaded video to Google Drive root folder with dynamic filename  
  - Inputs: from Download Video  
  - Outputs: to Build Success Response  
  - Failures: OAuth, quota, permission issues  

- **Build Success Response**  
  - Type: Set node  
  - Role: Constructs detailed JSON response containing video and background image metadata and shareable URLs, job and export IDs, success message, and source of trigger  
  - Inputs: from Upload Video to Drive  

- **From Webhook?**  
  - Type: If node  
  - Condition: Checks if source is "webhook"  
  - True output: to Respond to Webhook node  
  - False output: to Manual Test Complete node  

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Returns JSON payload to HTTP caller  
  - Inputs: from From Webhook?  

- **Manual Test Complete**  
  - Type: Set node  
  - Role: Prepares output for manual execution with final result JSON  

---

#### 1.5 Informational Sticky Notes

Sticky notes provide documentation, examples, and setup instructions at various workflow positions:  
- Overview of workflow purpose and timing  
- API key setup instructions with URLs and variable names  
- Input data requirements and examples  
- Description of Gemini Nano Banana image generation model and prompt tips  
- Detailed processing steps and timings  
- Google Drive connection instructions  
- Usage instructions for manual, webhook, or batch operation  
- Example inputs, generated image preview, and final composite video preview with links  

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                   |
|-----------------------------------|---------------------|-----------------------------------------------|--------------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| 📋 Overview                       | Sticky Note         | Workflow purpose & high-level info            |                                      |                                       | Overview of AI background generation, use cases, processing time, and setup instructions      |
| 🔑 API Key Setup                 | Sticky Note         | API key acquisition instructions              |                                      |                                       | Detailed Gemini and VideoBGRemover API key setup with pricing and variable names             |
| 📥 Inputs                        | Sticky Note         | Input requirements and format                  |                                      |                                       | Describes required inputs: video_url, background_prompt, optional aspect_ratio                |
| 🎨 Nano Banana                  | Sticky Note         | Gemini image generation explanation            |                                      |                                       | Details on Gemini Nano Banana model and prompt tips                                          |
| 🔄 Processing                   | Sticky Note         | Stepwise processing overview                    |                                      |                                       | Steps: generate background, remove bg, compose, upload                                      |
| 💾 Google Drive                 | Sticky Note         | Google Drive connection instructions           |                                      |                                       | Setup guide for Google Drive node authorization and folder selection                         |
| 🚀 Usage                       | Sticky Note         | How to use workflow (manual, webhook, batch)  |                                      |                                       | Instructions for manual testing, webhook POST, and batch automation                         |
| Webhook Trigger                 | Webhook             | Entry point for webhook POST requests          |                                      | Extract Webhook Data                  |                                                                                              |
| Manual Trigger                 | Manual Trigger       | Starts manual workflow testing                  |                                      | Sample Inputs (Edit Here)             |                                                                                              |
| Extract Webhook Data            | Set                 | Extracts and normalizes webhook input data      | Webhook Trigger                      | Merge Triggers                       |                                                                                              |
| Sample Inputs (Edit Here)       | Set                 | Provides manual test input values                | Manual Trigger                      | Merge Triggers                       |                                                                                              |
| Merge Triggers                 | Merge               | Merges webhook and manual triggers              | Extract Webhook Data, Sample Inputs  | 1. Generate Background Image (Gemini)|                                                                                              |
| 1. Generate Background Image (Gemini) | HTTP Request        | Calls Gemini API to generate background image   | Merge Triggers                      | Extract Image Data                   |                                                                                              |
| Extract Image Data              | Code                 | Extracts base64 image from Gemini response       | 1. Generate Background Image        | 2. Save Background Image to Drive    |                                                                                              |
| 2. Save Background Image to Drive | Google Drive         | Uploads generated background image               | Extract Image Data                  | 3. Make Background Image Public      |                                                                                              |
| 3. Make Background Image Public  | Google Drive         | Makes background image publicly accessible       | 2. Save Background Image to Drive  | 4. Create Job (Upload Video)          |                                                                                              |
| 4. Create Job (Upload Video)     | HTTP Request         | Uploads video URL to VideoBGRemover to create job | 3. Make Background Image Public     | 5. Start Composition (AI Background) |                                                                                              |
| 5. Start Composition (AI Background) | HTTP Request        | Starts video composition with AI background      | 4. Create Job (Upload Video)         | 6. Check Job Status                  |                                                                                              |
| 6. Check Job Status             | HTTP Request         | Polls VideoBGRemover job status                   | 5. Start Composition, Wait 20s      | Is Complete?, Has Failed?            |                                                                                              |
| Is Complete?                   | If                   | Checks if job status is "completed"               | 6. Check Job Status                 | 7. Download Video, Has Failed?       |                                                                                              |
| Has Failed?                   | If                   | Checks if job status is "failed"                  | 6. Check Job Status                 | Build Error Response, Wait 20s        |                                                                                              |
| Wait 20s                      | Wait                  | Waits 20 seconds before rechecking job status     | Has Failed?                        | 6. Check Job Status                  |                                                                                              |
| Build Error Response           | Set                   | Constructs error JSON response                      | Has Failed?                       | From Webhook?                       |                                                                                              |
| 7. Download Video             | HTTP Request           | Downloads composed video file                        | Is Complete?                     | 8. Upload Video to Drive             |                                                                                              |
| 8. Upload Video to Drive      | Google Drive           | Uploads final video to Google Drive                  | 7. Download Video                | Build Success Response               |                                                                                              |
| Build Success Response        | Set                   | Constructs success JSON response                      | 8. Upload Video to Drive          | From Webhook?                       |                                                                                              |
| From Webhook?                 | If                     | Checks if trigger source is webhook                  | Build Success Response, Build Error Response | Respond to Webhook, Manual Test Complete |                                                                                              |
| Respond to Webhook            | Respond to Webhook     | Sends JSON response back to HTTP client             | From Webhook?                    |                                       |                                                                                              |
| Manual Test Complete          | Set                   | Prepares output for manual workflow execution         | From Webhook?                    |                                       |                                                                                              |
| Section: Gemini               | Sticky Note            | Marks Gemini image generation section                |                                      |                                       |                                                                                              |
| 📸 Input                      | Sticky Note            | Example input data and video preview                  |                                      |                                       |                                                                                              |
| 📸 Generated                  | Sticky Note            | Shows example generated background image              |                                      |                                       |                                                                                              |
| Section: VideoBGRemover       | Sticky Note            | Marks VideoBGRemover composition section               |                                      |                                       |                                                                                              |
| Section: Output               | Sticky Note            | Marks final output section                              |                                      |                                       |                                                                                              |
| 📸 Final                      | Sticky Note            | Shows example final composed video                      |                                      |                                       |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Nodes:**  
   - Add a **Webhook Trigger** node:  
     - HTTP Method: POST  
     - Path: `ai-background-gen`  
     - Response Mode: `responseNode`  
   - Add a **Manual Trigger** node for manual testing.

2. **Extract Input Data:**  
   - Add a **Set** node named `Extract Webhook Data` connected from the Webhook Trigger.  
   - Configure to set:  
     - `video_url` = `{{$json.body?.video_url ?? $json.video_url}}`  
     - `background_prompt` = `{{$json.body?.background_prompt ?? $json.background_prompt}}`  
     - `aspect_ratio` = `{{$json.body?.aspect_ratio ?? $json.aspect_ratio ?? '1:1'}}`  
     - `source` = `"webhook"`  
   - Add another **Set** node named `Sample Inputs (Edit Here)` connected from Manual Trigger.  
   - Fill with example values for `video_url`, `background_prompt`, `aspect_ratio` ("16:9"), and `source` = `"manual"`.  
   - Add a **Merge** node named `Merge Triggers` set to "Append" mode, connecting both `Extract Webhook Data` and `Sample Inputs (Edit Here)`.

3. **Generate Background Image with Gemini:**  
   - Add an **HTTP Request** node named `1. Generate Background Image (Gemini)` connected from `Merge Triggers`.  
   - Configure:  
     - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent`  
     - Method: POST  
     - Headers:  
       - `x-goog-api-key`: set to environment variable `$vars.GEMINI_KEY`  
       - `Content-Type`: `application/json`  
     - Body (JSON):  
       ```json
       {
         "contents": [{"parts": [{"text": "{{$json.background_prompt}}"}]}],
         "generationConfig": {
           "responseModalities": ["Image"],
           "imageConfig": {"aspectRatio": "{{$json.aspect_ratio}}"}
         }
       }
       ```  
     - Response Format: JSON  

4. **Extract Image Data:**  
   - Add a **Code** node named `Extract Image Data` connected from Gemini HTTP Request.  
   - JavaScript code extracts base64 image and mimeType, converts base64 to binary buffer, and outputs a data URL for VideoBGRemover.

5. **Save Background Image to Google Drive:**  
   - Add a **Google Drive** node named `2. Save Background Image to Drive` connected from the code node.  
   - Operation: Upload file  
   - File Name: dynamically generated from prompt snippet + timestamp, e.g.,  
     `ai_bg_{{ $json.background_prompt.substring(0,30).replace(/[^a-zA-Z0-9]/g, '_') }}_{{ new Date().getTime() }}.png`  
   - Binary Property: `data`  
   - Folder: Root or configured folder  
   - Credential: Google Drive OAuth2  

6. **Make Background Image Public:**  
   - Add another **Google Drive** node named `3. Make Background Image Public` connected from previous node.  
   - Operation: Share file  
   - File ID: `{{$json.id}}`  
   - Permissions: Anyone with link can view (role: reader, type: anyone)  

7. **Upload Video and Start Composition (VideoBGRemover):**  
   - Add an **HTTP Request** node named `4. Create Job (Upload Video)` connected from previous node.  
   - POST to `https://api.videobgremover.com/api/v1/jobs` with JSON body `{"video_url": "{{$json.video_url}}"}`  
   - Headers include `X-API-Key` with `$vars.VIDEOBGREMOVER_KEY`.  
   - Add an **HTTP Request** node named `5. Start Composition (AI Background)` connected from create job node.  
   - POST to `https://api.videobgremover.com/api/v1/jobs/{{$json.id}}/start`  
   - JSON body specifies composition template "centered" with background image URL from Google Drive public link:  
     ```
     {
       "background": {
         "type": "composition",
         "composition": {
           "template": "centered",
           "background_type": "image",
           "background_url": "https://drive.google.com/uc?id={{$json.id}}",
           "export_format": "h264",
           "export_preset": "medium"
         }
       }
     }
     ```  
   - Headers same as above.

8. **Poll Job Status:**  
   - Add an **HTTP Request** node `6. Check Job Status` connected from start composition node and from a wait node (loop).  
   - GET request to `https://api.videobgremover.com/api/v1/jobs/{{$json.id}}/status` with API key header.  
   - Add an **If** node `Is Complete?` checking if `{{$json.status}} === 'completed'`  
   - True: continue to download video  
   - False: connect to another **If** node `Has Failed?` checking if status is "failed"  
     - True: build error response  
     - False: connect to **Wait** node (20 seconds) which loops back to check status.

9. **Download Composed Video:**  
   - Add **HTTP Request** node `7. Download Video` connected from `Is Complete?` true path.  
   - URL: `{{$json.processed_video_url}}`  
   - Response type: file  

10. **Upload Composed Video to Google Drive:**  
    - Add Google Drive node `8. Upload Video to Drive` connected from download video node.  
    - Upload binary file, filename with job ID and timestamp.  

11. **Build Success and Error Responses:**  
    - Add **Set** node `Build Success Response` connected from upload video node.  
    - Constructs JSON including video metadata, background image metadata, job ids, success message, and source.  
    - Add **Set** node `Build Error Response` connected from `Has Failed?` node.  
    - Constructs failure JSON with error message and source.  

12. **Respond to Webhook or Manual Output:**  
    - Add **If** node `From Webhook?` checking if source is "webhook".  
    - True: connect to **Respond to Webhook** node to send JSON response to HTTP client.  
    - False: connect to **Set** node `Manual Test Complete` to output final result for manual workflow execution.

13. **Add Sticky Notes:**  
    - Add explanatory sticky notes for overview, API key setup, inputs, Gemini section, VideoBGRemover section, output section, example inputs and outputs, and usage instructions, positioned logically.

14. **Credentials Setup:**  
    - Configure environment variables or n8n credentials for:  
      - Gemini API key as `GEMINI_KEY`  
      - VideoBGRemover API key as `VIDEOBGREMOVER_KEY`  
      - Google Drive OAuth2 credential for file upload and sharing  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                     |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Full documentation of VideoBGRemover API including job creation and composition details                               | https://docs.videobgremover.com/                                  |
| Gemini API key creation and usage information                                                                         | https://aistudio.google.com/apikey                                |
| Supported aspect ratios and image generation parameters for Gemini                                                    | https://ai.google.dev/gemini-api/docs/image-generation#aspect_ratios |
| Example input video with AI actor and background prompt                                                               | https://videos.videobgremover.com/public-videos/assets/ai-actor.mp4 |
| Example generated AI background image and final composed video previews                                               | Provided as images and video links in sticky notes within workflow |
| Pricing information for Gemini image generation (~$0.03/image) and VideoBGRemover (~$0.50-$2.00 per minute)           | Mentioned in API Key Setup sticky note                            |
| Workflow processing time estimate: 2-4 minutes per video                                                              | Overview sticky note                                              |

---

**Disclaimer:**  
The content provided above is exclusively derived from an n8n automated workflow configuration. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.