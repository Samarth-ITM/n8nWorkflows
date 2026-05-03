Make UGC ads from app screen recordings with Gemini, Sora 2, and VideoBGRemover

https://n8nworkflows.xyz/workflows/make-ugc-ads-from-app-screen-recordings-with-gemini--sora-2--and-videobgremover-9505


# Make UGC ads from app screen recordings with Gemini, Sora 2, and VideoBGRemover

### 1. Workflow Overview

This n8n workflow automates the creation of User-Generated Content (UGC) style ads from vertical app screen recording videos. It targets app developers, SaaS marketers, and mobile product teams who want to produce engaging ads with AI-generated actors and background compositing.

The workflow is logically structured into four main blocks:

- **1.1 Input Reception**  
  Accepts video URLs and optional metadata either via manual trigger or webhook, normalizes inputs for downstream processing.

- **1.2 Gemini AI Analysis**  
  Downloads the screen recording video, uploads it to Google’s Gemini AI for semantic analysis, and extracts a structured UGC ad plan (hook, problem, solution, CTA, visual and emotional details).

- **1.3 Sora 2 AI Video Generation**  
  Constructs a prompt from Gemini’s output and submits it to Sora 2 AI to generate a naturalistic AI actor video matching the ad structure and emotional journey.

- **1.4 VideoBGRemover Composition & Output**  
  Removes the AI actor’s video background, composites the actor over the original screen recording, mixes audio, then downloads and optionally uploads the final video to Google Drive. The workflow then returns the result via webhook or manual completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block handles the initial input of the workflow. It supports two entry methods: a webhook for live automation and a manual trigger for testing. It extracts and normalizes all necessary parameters for the rest of the flow.

**Nodes Involved:**  
- Webhook Trigger  
- Manual Trigger  
- Extract Webhook Data  
- Sample Input (Edit Here)  
- Merge Triggers  

**Node Details:**

- **Webhook Trigger**  
  - Type: Webhook  
  - Role: Entry point for automated POST requests containing video URL and optional metadata.  
  - Configuration: Path `ugc-screenshot-video`, HTTP POST, responds with all entries.  
  - Inputs: External HTTP POST requests  
  - Outputs: JSON body of request for downstream use  
  - Edge Cases: Invalid or missing parameters, non-POST methods  
  - Notes: Designed for automation with external services  

- **Manual Trigger**  
  - Type: Manual trigger  
  - Role: Allows manual testing and execution inside n8n editor.  
  - Inputs: None  
  - Outputs: Triggers downstream nodes with sample data  
  - Edge Cases: None significant  

- **Extract Webhook Data**  
  - Type: Set node  
  - Role: Parses incoming data to extract `screenshot_video_url`, optional `meta` object, `store_to_drive` boolean, and others. Supports fallback for manual inputs.  
  - Key Expressions: Uses expression syntax to access JSON paths safely (`$json.body?.field ?? $json.field ?? default`)  
  - Inputs: From webhook or manual trigger  
  - Outputs: Normalized JSON with required parameters  
  - Edge Cases: Missing or malformed inputs, empty optional fields  

- **Sample Input (Edit Here)**  
  - Type: Set node  
  - Role: Provides default sample data for manual testing, including a sample vertical video URL and meta info.  
  - Inputs: Manual trigger  
  - Outputs: Sample normalized JSON data  
  - Edge Cases: Should be updated to valid URLs before production use  

- **Merge Triggers**  
  - Type: Merge  
  - Role: Combines outputs from webhook and manual paths into a single stream for processing.  
  - Inputs: From Extract Webhook Data and Sample Input nodes  
  - Outputs: Unified data stream  
  - Edge Cases: None; append mode ensures all inputs pass through  

---

#### 2.2 Gemini AI Analysis

**Overview:**  
Downloads the input video, uploads it to Gemini AI for content understanding, waits for processing, then retrieves and parses a structured JSON ad plan describing the UGC ad’s elements.

**Nodes Involved:**  
- Download Screenshot Video  
- Upload to Gemini  
- Wait for Processing (30s)  
- Gemini – Analyze & Plan  
- Extract & Parse  

**Node Details:**

- **Download Screenshot Video**  
  - Type: HTTP Request  
  - Role: Downloads the input screen recording video from the provided URL.  
  - Configuration: URL dynamically set from input `screenshot_video_url`  
  - Inputs: Normalized input data  
  - Outputs: Binary video data for upload  
  - Edge Cases: Invalid URL, network timeouts, unsupported video formats  

- **Upload to Gemini**  
  - Type: HTTP Request  
  - Role: Uploads the binary video to Gemini’s generativelanguage API using multipart upload with specific headers for content length and type.  
  - Configuration: Uses `GEMINI_KEY` from n8n variables for authentication; sends video as binary data  
  - Inputs: Binary video data from previous node  
  - Outputs: Upload confirmation with file URI  
  - Edge Cases: API key invalid/missing, upload failures, video too large  

- **Wait for Processing**  
  - Type: Wait node  
  - Role: Pauses workflow 30 seconds to allow asynchronous processing on Gemini’s side.  
  - Inputs: Upload response  
  - Outputs: Passes data forward after delay  
  - Edge Cases: Fixed wait time may be insufficient or excessive depending on API latency  

- **Gemini – Analyze & Plan**  
  - Type: HTTP Request  
  - Role: Sends a text + file URI prompt to Gemini’s model to generate a structured JSON describing the UGC ad plan.  
  - Configuration: Uses model `gemini-2.5-pro`, sends JSON with prompt specifying output format and constraints.  
  - Inputs: File URI from upload node  
  - Outputs: Gemini JSON content with ad structure, emotional journey, visual details, and recommended video duration  
  - Edge Cases: API rate limits, malformed response, unexpected JSON output  

- **Extract & Parse**  
  - Type: Code (JavaScript)  
  - Role: Extracts the text content from Gemini’s response, cleans markdown syntax, parses JSON, and outputs structured fields for downstream nodes.  
  - Key Expressions: Uses regex to remove markdown code blocks, JSON.parse for content  
  - Inputs: Gemini API response  
  - Outputs: Parsed JSON with keys: ad_topic, icp_summary, ad_structure, visual_details, emotional_journey, duration  
  - Edge Cases: Parsing errors if Gemini response format changes or invalid JSON  

---

#### 2.3 Sora 2 AI Video Generation

**Overview:**  
Builds a natural language prompt for Sora 2 AI based on Gemini’s ad plan, submits the prompt to generate an AI actor video, then polls the Sora 2 API until the video generation completes.

**Nodes Involved:**  
- Build Sora Prompt  
- Sora 2 – Submit (fal.ai)  
- Sora – Check Status  
- Wait 20s (polling)  
- Sora Completed? (If)  
- Sora – Get Result  

**Node Details:**

- **Build Sora Prompt**  
  - Type: Code (JavaScript)  
  - Role: Constructs a detailed text prompt for Sora 2 AI using the Gemini ad structure and visual/emotional details. Extracts camera style and actor description.  
  - Key Expressions: String template combining `visual_details`, `ad_structure`, `emotional_journey`, and `ad_topic`  
  - Output: JSON with `sora_prompt` string and `duration`  
  - Edge Cases: Missing fields may produce incomplete prompts; text length constraints  

- **Sora 2 – Submit (fal.ai)**  
  - Type: HTTP Request  
  - Role: Sends the prompt to fal.ai’s Sora 2 text-to-video endpoint, specifying vertical 9:16 aspect ratio, 720p resolution, and duration from Gemini.  
  - Configuration: Authentication via `FAL_KEY` variable in header  
  - Inputs: Prompt and duration from previous node  
  - Outputs: Request ID for status polling  
  - Edge Cases: API key invalid, request quota exceeded, invalid prompt format  

- **Sora – Check Status**  
  - Type: HTTP Request  
  - Role: Polls the Sora 2 API to check the status of the video generation job using the request ID.  
  - Inputs: Request ID from submit node  
  - Outputs: JSON with status field (`COMPLETED`, `IN_PROGRESS`, etc.)  
  - Edge Cases: Network errors, API downtime, unexpected status values  

- **Wait 20s**  
  - Type: Wait node  
  - Role: Delays polling by 20 seconds before re-checking status.  
  - Edge Cases: Fixed wait time may cause longer than necessary waits  

- **Sora Completed? (If)**  
  - Type: If node  
  - Role: Branches workflow depending on whether the Sora job status is `COMPLETED`.  
  - True: Proceed to get result  
  - False: Loop back to wait and check again  
  - Edge Cases: Infinite loop risk if job never completes  

- **Sora – Get Result**  
  - Type: HTTP Request  
  - Role: Retrieves the completed video URL and metadata for the AI actor video.  
  - Inputs: Request ID  
  - Outputs: Video URL and metadata JSON  
  - Edge Cases: Result unavailable if job failed or expired  

---

#### 2.4 VideoBGRemover Composition & Output

**Overview:**  
Submits the AI actor video to VideoBGRemover to remove background and composite it over the original screen recording video. Polls until composition is done, downloads the final video, optionally uploads to Google Drive, and returns the final output.

**Nodes Involved:**  
- VBR – Create Job (Foreground=Sora)  
- VBR – Start Composition  
- VBR – Check Status  
- Wait 20s (VBR)  
- VBR Completed? (If)  
- Prepare Final URL  
- Download Final Video  
- Upload to Google Drive  
- Build Drive Response  
- From Webhook? (If)  
- Respond to Webhook  
- Manual Test Complete  

**Node Details:**

- **VBR – Create Job (Foreground=Sora)**  
  - Type: HTTP Request  
  - Role: Creates a VideoBGRemover job with the AI actor video URL as foreground input.  
  - Authentication: Uses `VIDEOBGREMOVER_KEY` variable in header  
  - Inputs: Video URL from Sora Get Result node  
  - Outputs: Job ID and metadata  
  - Edge Cases: API key invalid, job creation failure  

- **VBR – Start Composition**  
  - Type: HTTP Request  
  - Role: Starts the composition job with parameters specifying:  
    - Background: video (the original screen recording URL)  
    - Audio mixing: background at 30% volume, foreground at 100%  
    - Export format preset: medium quality H264  
    - Position: bottom-right corner implied by template `ai_ugc_ad`  
  - Inputs: Job ID from create job node  
  - Outputs: Composition started confirmation  
  - Edge Cases: Composition errors, invalid background URL  

- **VBR – Check Status**  
  - Type: HTTP Request  
  - Role: Polls the composition job status until `completed`.  
  - Inputs: Job ID  
  - Outputs: Status JSON  
  - Edge Cases: API errors, indefinite waiting if job stuck  

- **Wait 20s (VBR)**  
  - Type: Wait node  
  - Role: Delays between status polls for composition  
  - Edge Cases: Fixed polling interval may delay completion  

- **VBR Completed? (If)**  
  - Type: If node  
  - Role: Checks if composition job status is `completed`  
  - True: Proceed to next step  
  - False: Loop back to wait and check again  
  - Edge Cases: Infinite loop risk if job fails silently  

- **Prepare Final URL**  
  - Type: Set node  
  - Role: Extracts `processed_video_url` and video length from VBR result for download and output  
  - Inputs: Completed VBR job status JSON  
  - Outputs: Final video URL and metadata  

- **Download Final Video**  
  - Type: HTTP Request  
  - Role: Downloads the composited video file as binary data  
  - Inputs: Final URL from Prepare Final URL node  
  - Outputs: Binary video data  

- **Upload to Google Drive**  
  - Type: Google Drive node  
  - Role: Uploads the final video file to Google Drive root folder with auto-generated name `ugc_final_[timestamp].mp4`  
  - Inputs: Binary data from download node  
  - Outputs: Google Drive file metadata including shareable link  
  - Edge Cases: Google OAuth failures, quota exceeded  

- **Build Drive Response**  
  - Type: Set node  
  - Role: Builds a JSON response containing Google Drive share URL, file ID, and source tag for downstream use or webhook response  
  - Inputs: Google Drive upload metadata  
  - Outputs: JSON response  

- **From Webhook? (If)**  
  - Type: If node  
  - Role: Detects whether the workflow was triggered by webhook or manual trigger using `source` field  
  - True: Responds to webhook with JSON response  
  - False: Sets manual test complete output  

- **Respond to Webhook**  
  - Type: Respond to Webhook node  
  - Role: Sends JSON response back to webhook caller with 200 status code  
  - Inputs: JSON response from Build Drive Response  

- **Manual Test Complete**  
  - Type: Set node  
  - Role: Final output for manual runs to log or inspect results  

---

### 3. Summary Table

| Node Name                    | Node Type                | Functional Role                             | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                       |
|------------------------------|--------------------------|---------------------------------------------|----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------|
| Webhook Trigger              | Webhook                  | Entry point for webhook requests             | -                                | Extract Webhook Data               |                                                                                                                  |
| Manual Trigger               | Manual Trigger           | Entry point for manual testing                | -                                | Sample Input (Edit Here)           |                                                                                                                  |
| Extract Webhook Data         | Set                      | Normalize input data                          | Webhook Trigger                  | Merge Triggers                    |                                                                                                                  |
| Sample Input (Edit Here)     | Set                      | Supplies sample input for manual testing     | Manual Trigger                   | Merge Triggers                    |                                                                                                                  |
| Merge Triggers              | Merge                    | Merge webhook and manual trigger data        | Extract Webhook Data, Sample Input | Download Screenshot Video          |                                                                                                                  |
| Download Screenshot Video    | HTTP Request             | Download input video from URL                 | Merge Triggers                  | Upload to Gemini                  |                                                                                                                  |
| Upload to Gemini             | HTTP Request             | Upload video to Gemini API                     | Download Screenshot Video        | Wait for Processing               |                                                                                                                  |
| Wait for Processing          | Wait                     | Wait 30 seconds for Gemini processing         | Upload to Gemini                 | Gemini – Analyze & Plan           |                                                                                                                  |
| Gemini – Analyze & Plan      | HTTP Request             | Analyze video & generate UGC ad plan          | Wait for Processing             | Extract & Parse                  |                                                                                                                  |
| Extract & Parse              | Code                     | Parse Gemini JSON response                     | Gemini – Analyze & Plan          | Build Sora Prompt                |                                                                                                                  |
| Build Sora Prompt            | Code                     | Build text prompt for Sora 2 AI                | Extract & Parse                 | Sora 2 – Submit (fal.ai)          |                                                                                                                  |
| Sora 2 – Submit (fal.ai)    | HTTP Request             | Submit prompt for AI actor video generation    | Build Sora Prompt               | Sora – Check Status              |                                                                                                                  |
| Sora – Check Status          | HTTP Request             | Poll Sora API job status                        | Sora 2 – Submit (fal.ai), Wait 20s | Sora Completed?                  |                                                                                                                  |
| Wait 20s                    | Wait                     | Wait 20 seconds for Sora job completion        | Sora Completed? (False branch)  | Sora – Check Status              |                                                                                                                  |
| Sora Completed?              | If                       | Branch on Sora job completion                   | Sora – Check Status             | Sora – Get Result / Wait 20s      |                                                                                                                  |
| Sora – Get Result            | HTTP Request             | Retrieve completed AI actor video URL           | Sora Completed? (True branch)   | VBR – Create Job (Foreground=Sora) |                                                                                                                  |
| VBR – Create Job (Foreground=Sora) | HTTP Request       | Create VideoBGRemover job with actor video      | Sora – Get Result               | VBR – Start Composition          |                                                                                                                  |
| VBR – Start Composition      | HTTP Request             | Start background removal & compositing job      | VBR – Create Job                | VBR – Check Status               |                                                                                                                  |
| VBR – Check Status           | HTTP Request             | Poll VideoBGRemover job status                   | VBR – Start Composition, Wait 20s (VBR) | VBR Completed?                   |                                                                                                                  |
| Wait 20s (VBR)              | Wait                     | Wait 20 seconds between VBR status polls         | VBR Completed? (False branch)   | VBR – Check Status               |                                                                                                                  |
| VBR Completed?               | If                       | Branch on VideoBGRemover job completion           | VBR – Check Status              | Prepare Final URL / Wait 20s (VBR) |                                                                                                                  |
| Prepare Final URL            | Set                      | Extract final composited video URL and metadata | VBR Completed? (True branch)    | Download Final Video             |                                                                                                                  |
| Download Final Video         | HTTP Request             | Download composited video file                     | Prepare Final URL               | Upload to Google Drive           |                                                                                                                  |
| Upload to Google Drive       | Google Drive             | Upload final video to Google Drive                 | Download Final Video            | Build Drive Response             |                                                                                                                  |
| Build Drive Response         | Set                      | Build response JSON with Google Drive links       | Upload to Google Drive          | From Webhook?                   |                                                                                                                  |
| From Webhook?                | If                       | Branch to respond to webhook or complete manually | Build Drive Response            | Respond to Webhook / Manual Test Complete |                                                                                                                  |
| Respond to Webhook           | Respond to Webhook       | Send JSON response to webhook caller               | From Webhook? (True branch)     | -                               |                                                                                                                  |
| Manual Test Complete         | Set                      | Final output for manual testing                     | From Webhook? (False branch)    | -                               |                                                                                                                  |
| 📋 Overview                 | Sticky Note              | Workflow purpose and usage overview                | -                              | -                               | Contains detailed workflow purpose and usage info with link to full docs                                         |
| 🔑 API Keys                 | Sticky Note              | API keys setup instructions                         | -                              | -                               | Explains required API keys and how to set them in n8n variables                                                  |
| 📥 Inputs                   | Sticky Note              | Input data requirements and example                  | -                              | -                               | Details expected input JSON structure and video recommendations                                                  |
| 🔄 Workflow                 | Sticky Note              | Step-by-step workflow outline                        | -                              | -                               | Describes the major processing steps with approximate times                                                    |
| 📝 Sora Prompt              | Sticky Note              | Sora prompt template and key elements                | -                              | -                               | Explains prompt composition for Sora 2 AI generation                                                            |
| 💾 Google Drive             | Sticky Note              | Google Drive upload setup instructions               | -                              | -                               | How to connect Google Drive and what output to expect                                                           |
| 🚀 Usage                    | Sticky Note              | How to use workflow manually or via webhook          | -                              | -                               | Manual and automated usage instructions with tips                                                                |
| Section: Gemini             | Sticky Note              | Marks Gemini AI analysis section                      | -                              | -                               |                                                                                                                  |
| Section: Sora 2             | Sticky Note              | Marks Sora 2 AI video generation section              | -                              | -                               |                                                                                                                  |
| Section: VideoBGRemover     | Sticky Note              | Marks VideoBGRemover composition section              | -                              | -                               |                                                                                                                  |
| Section: Output             | Sticky Note              | Marks final output and response section               | -                              | -                               |                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `ugc-screenshot-video`  
   - Response Mode: Respond with all entries

2. **Create Manual Trigger:**  
   - Type: Manual Trigger  
   - No parameters

3. **Create Set Node "Extract Webhook Data":**  
   - Extract and normalize data:  
     - `screenshot_video_url`: from `$json.body?.screenshot_video_url` or `$json.screenshot_video_url`  
     - `meta`: from `$json.body?.meta` or `$json.meta` or default `{}`  
     - `store_to_drive`: boolean from `$json.body?.store_to_drive` or `$json.store_to_drive` or false  
     - `reference_image_url`: optional string, default empty  
     - `source`: fixed string `"webhook"`  

4. **Create Set Node "Sample Input (Edit Here)":**  
   - Provide sample data for manual runs:  
     - `screenshot_video_url`: example vertical video URL  
     - `meta`: object with sample product name, benefits array, tone  
     - `store_to_drive`: true  
     - `reference_image_url`: empty string  
     - `source`: `"manual"`

5. **Create Merge Node "Merge Triggers":**  
   - Mode: Append  
   - Inputs: from "Extract Webhook Data" and "Sample Input (Edit Here)"

6. **Create HTTP Request "Download Screenshot Video":**  
   - Method: GET  
   - URL: Expression from `screenshot_video_url` field  
   - Response: Binary data (video)

7. **Create HTTP Request "Upload to Gemini":**  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files?key={{ $vars.GEMINI_KEY }}`  
   - Authentication: None (uses key in URL)  
   - Headers:  
     - `X-Goog-Upload-Command`: `start, upload, finalize`  
     - `X-Goog-Upload-Header-Content-Length`: binary file size  
     - `X-Goog-Upload-Header-Content-Type`: `video/{{ $binary.data.fileExtension }}`  
     - `Content-Type`: `video/mp4`  
   - Body: Binary video data from previous node

8. **Create Wait Node "Wait for Processing":**  
   - Duration: 30 seconds

9. **Create HTTP Request "Gemini – Analyze & Plan":**  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent?key={{ $vars.GEMINI_KEY }}`  
   - Method: POST  
   - Headers: `Content-Type: application/json`  
   - Body: JSON with contents array including file URI and text prompt that instructs Gemini to analyze the video and return a strict JSON ad plan with fields: ad_topic, icp_summary, ad_structure (hook, problem, solution, cta), visual_details, emotional_journey, duration. Use meta if available.  
   - Generation config: temperature 0.7, topK 40, topP 0.95, maxOutputTokens 8192

10. **Create Code Node "Extract & Parse":**  
    - JavaScript code to:  
      - Extract text from Gemini response candidates  
      - Remove markdown code blocks  
      - Parse JSON  
      - Return parsed fields: ad_topic, icp_summary, ad_structure, visual_details, emotional_journey, duration  

11. **Create Code Node "Build Sora Prompt":**  
    - Compose a string prompt for Sora 2 AI using fields from Gemini output:  
      Format:  
      ```  
      [Actor & setting description] filming a UGC ad about [ad_topic].  
      They start by saying: [hook], then explain: [problem], present the solution: [solution], and end with: [cta].  
      [Emotional journey]. [Camera style].  
      Natural hand gestures and authentic expressions throughout. Raw selfie video of ONLY the person talking - no overlays or graphics.  
      ```  
    - Extract camera style from visual details string or default to "Handheld selfie-style close-up".

12. **Create HTTP Request "Sora 2 – Submit (fal.ai)":**  
    - URL: `https://queue.fal.run/fal-ai/sora-2/text-to-video`  
    - Method: POST  
    - Headers: Authorization `Key {{ $vars.FAL_KEY }}`, Content-Type `application/json`  
    - Body JSON:  
      - prompt: from Build Sora Prompt  
      - aspect_ratio: "9:16"  
      - resolution: "720p"  
      - duration: from Build Sora Prompt

13. **Create HTTP Request "Sora – Check Status":**  
    - URL: `https://queue.fal.run/fal-ai/sora-2/requests/{{ $json.request_id }}/status`  
    - Method: GET  
    - Headers: Authorization `Key {{ $vars.FAL_KEY }}`  
    - Configure to never error on unknown status  

14. **Create Wait Node "Wait 20s":**  
    - Duration: 20 seconds

15. **Create If Node "Sora Completed?":**  
    - Condition: `$json.status` equals "COMPLETED" (case-sensitive)  
    - True: Proceed to "Sora – Get Result"  
    - False: Loop back to "Wait 20s"

16. **Create HTTP Request "Sora – Get Result":**  
    - URL: `https://queue.fal.run/fal-ai/sora-2/requests/{{ $('Sora 2 – Submit (fal.ai)').item.json.request_id }}`  
    - Method: GET  
    - Headers: Authorization `Key {{ $vars.FAL_KEY }}`

17. **Create HTTP Request "VBR – Create Job (Foreground=Sora)":**  
    - URL: `https://api.videobgremover.com/api/v1/jobs`  
    - Method: POST  
    - Headers: `X-API-Key: {{ $vars.VIDEOBGREMOVER_KEY }}`, `Content-Type: application/json`  
    - Body JSON: `{ "video_url": "{{ $json.video.url }}" }`

18. **Create HTTP Request "VBR – Start Composition":**  
    - URL: `https://api.videobgremover.com/api/v1/jobs/{{ $json.id }}/start`  
    - Method: POST  
    - Headers: Same as above  
    - Body JSON:  
      ```json
      {
        "background": {
          "type": "composition",
          "composition": {
            "template": "ai_ugc_ad",
            "background_type": "video",
            "background_url": "{{ $('Merge Triggers').item.json.screenshot_video_url }}",
            "background_audio_enabled": true,
            "background_audio_volume": 0.3,
            "export_format": "h264",
            "export_preset": "medium"
          }
        }
      }
      ```

19. **Create HTTP Request "VBR – Check Status":**  
    - URL: `https://api.videobgremover.com/api/v1/jobs/{{ $('VBR – Create Job (Foreground=Sora)').item.json.id }}/status`  
    - Method: GET  
    - Headers: `X-API-Key: {{ $vars.VIDEOBGREMOVER_KEY }}`  

20. **Create Wait Node "Wait 20s (VBR)":**  
    - Duration: 20 seconds

21. **Create If Node "VBR Completed?":**  
    - Condition: `$json.status` equals "completed" (case-sensitive)  
    - True: Proceed to "Prepare Final URL"  
    - False: Loop back to "Wait 20s (VBR)"

22. **Create Set Node "Prepare Final URL":**  
    - Assign:  
      - `final_url` = `$json.processed_video_url`  
      - `length_seconds` = `$json.length_seconds` or fallback to meta duration

23. **Create HTTP Request "Download Final Video":**  
    - Method: GET  
    - URL: Expression from `final_url`  
    - Response format: file (binary)

24. **Create Google Drive Node "Upload to Google Drive":**  
    - Operation: Upload  
    - Binary Property Name: `data` (from downloaded video)  
    - File Name: `ugc_final_{{ new Date().getTime() }}.mp4`  
    - Folder: Root or configurable  
    - Authenticate with OAuth2 Google Drive credentials

25. **Create Set Node "Build Drive Response":**  
    - Build object:  
      - url: Google Drive `webViewLink`  
      - drive_file_id: Google Drive file id  
      - source: "google_drive"

26. **Create If Node "From Webhook?":**  
    - Condition: `source` equals `"webhook"`  
    - True: Send response to webhook  
    - False: Manual test completion

27. **Create Respond to Webhook Node:**  
    - Response body: from "Build Drive Response"  
    - Response code: 200

28. **Create Set Node "Manual Test Complete":**  
    - Assign `final_result` from last response for inspection

29. **Connect all nodes as per the logical flow:**  
    - Webhook Trigger → Extract Webhook Data → Merge Triggers → Download Screenshot Video → Upload to Gemini → Wait for Processing → Gemini – Analyze & Plan → Extract & Parse → Build Sora Prompt → Sora 2 Submit → Sora Check Status → Wait 20s → loop until done → Sora Get Result → VBR Create Job → VBR Start Composition → VBR Check Status → Wait 20s (VBR) → loop until done → Prepare Final URL → Download Final Video → Upload to Google Drive → Build Drive Response → From Webhook? → Respond or Manual Complete  
    - Manual Trigger → Sample Input → Merge Triggers (same path as webhook)

30. **Set required environment variables with API keys in n8n Settings → Variables:**  
    - `GEMINI_KEY` for Gemini API  
    - `FAL_KEY` for Sora 2 fal.ai API  
    - `VIDEOBGREMOVER_KEY` for VideoBGRemover API

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Full documentation and usage instructions available at https://docs.videobgremover.com/                                                            | Workflow overview sticky note                       |
| Recommended input videos are vertical (9:16) app screen recordings, 4-12 seconds duration, publicly accessible URLs                                | Input sticky note                                   |
| API keys must be securely stored as n8n variables and referenced with `$vars.KEY_NAME`                                                             | API Keys sticky note                                |
| Workflow processing time is approximately 5-8 minutes including Gemini analysis, Sora video generation, and background removal/composition        | Workflow sticky note                                |
| Sora prompt structure includes actor description, emotional journey, and script segments (hook, problem, solution, CTA) to ensure contextual AI video generation | Sora Prompt sticky note                             |
| Google Drive upload node requires OAuth2 credential setup, files named automatically with timestamp for uniqueness                                 | Google Drive sticky note                            |
| Manual trigger useful for testing and debugging with sample input before deploying webhook automation                                               | Usage sticky note                                   |

---

**Disclaimer:** The provided description and analysis are based on a fully automated n8n workflow designed for legal and public data processing with no inclusion of illegal or protected content.