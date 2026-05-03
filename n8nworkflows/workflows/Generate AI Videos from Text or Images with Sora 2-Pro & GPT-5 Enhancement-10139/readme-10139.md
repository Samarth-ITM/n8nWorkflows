Generate AI Videos from Text or Images with Sora 2/Pro & GPT-5 Enhancement

https://n8nworkflows.xyz/workflows/generate-ai-videos-from-text-or-images-with-sora-2-pro---gpt-5-enhancement-10139


# Generate AI Videos from Text or Images with Sora 2/Pro & GPT-5 Enhancement

### 1. Workflow Overview

This workflow automates the generation of AI videos using OpenAI's Sora 2 (including Pro variant) enhanced by GPT-5 prompt refinement, orchestrated via the fal.ai platform. It supports two input modes: text-to-video and image-to-video. Users submit either a textual prompt or an image through a web form, choose video parameters (aspect ratio, duration, model), and receive a generated video URL upon completion.

The workflow’s logic is organized into these main functional blocks:

- **1.1 Input Reception:** Captures user inputs (prompt, aspect ratio, model, duration, optional image) via a web form.
- **1.2 Input Mode Routing:** Determines if the input includes an image; branches to image or text processing accordingly.
- **1.3 Image-to-Video Processing:** Uploads the image to a temporary hosting service, then calls fal.ai's image-to-video API.
- **1.4 Text-to-Video Processing:** Uses GPT-5 to refine the raw text prompt into a cinematic and detailed format, parses the JSON output, then calls fal.ai's text-to-video API.
- **1.5 Status Polling Loop:** Waits, polls video generation status until completion.
- **1.6 Video Retrieval & Redirect:** Fetches the final video URL and redirects the user to the video.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user input through a web form with required fields for prompt, aspect ratio, model choice, duration, and an optional image file upload.

- **Nodes Involved:**  
  - Video Input Form  
  - Note: Form Trigger1 (documentation sticky note)

- **Node Details:**  
  - **Video Input Form**  
    - Type: Form Trigger  
    - Role: Webhook to receive user submissions.  
    - Config: Fields include Prompt (text, required), Aspect Ratio (dropdown: 9:16 or 16:9, required), Model (checkbox single select: sora-2 or sora-2-pro, required), Length (checkbox single select: 4s, 8s, 12s, required), Image (optional file upload, types jpg/jpeg/png).  
    - Input: HTTP form submission.  
    - Output: JSON with user data.  
    - Notes: Validates required fields; triggers workflow on form submission.  
    - Edge cases: Empty required fields cause rejection; large images (>10MB) may fail upload later.

---

#### 2.2 Input Mode Routing

- **Overview:**  
  Branches workflow based on presence of an uploaded image; routes to image-to-video path if image exists, else text-to-video path.

- **Nodes Involved:**  
  - Input Mode Router  
  - Note: Mode Router (documentation sticky note)

- **Node Details:**  
  - **Input Mode Router**  
    - Type: Switch  
    - Role: Checks if the `Image.filename` field is non-empty.  
    - Config: Two outputs: "Image to Video" if image is present, "Text to Video" if not.  
    - Input: Output of Video Input Form node.  
    - Output: Routes to either image or text processing branches.  
    - Edge cases: Filename field missing or malformed causes default to text branch.

---

#### 2.3 Image-to-Video Processing

- **Overview:**  
  Uploads the user image to tmpfiles.org, modifies URL for direct API access, then calls fal.ai's image-to-video endpoint with user parameters.

- **Nodes Involved:**  
  - Temp Image Upload  
  - Image-to-Video Call  
  - Note: Image Upload  
  - Note: Image-to-Video

- **Node Details:**  
  - **Temp Image Upload**  
    - Type: HTTP Request  
    - Role: Uploads image file to temporary hosting service.  
    - Config: POST multipart/form-data to `https://tmpfiles.org/api/v1/upload` with file field bound to form upload.  
    - Input: Image file from Video Input Form.  
    - Output: JSON containing uploaded image URL.  
    - Edge cases: Upload failure, large file rejection, network errors.  
    - Note: URL is adjusted by replacing `.org/` with `.org/dl/` for direct download access.  

  - **Image-to-Video Call**  
    - Type: HTTP Request  
    - Role: Sends prompt, image URL, aspect ratio, duration to fal.ai image-to-video API.  
    - Config: POST JSON body with prompt (raw from form), resolution set to "auto", aspect ratio cleaned of label text, duration parsed as integer seconds, image URL modified for direct access. Endpoint path appends `/pro` if model selected is sora-2-pro.  
    - Input: Output of Temp Image Upload and form data.  
    - Output: JSON containing request_id.  
    - Credentials: fal.ai HTTP Header Auth for authorization.  
    - Edge cases: API errors, auth failures, invalid parameters.

---

#### 2.4 Text-to-Video Processing

- **Overview:**  
  Refines the raw text prompt using GPT-5 into a professionally structured cinematic prompt, validates output JSON, then submits it to fal.ai text-to-video API.

- **Nodes Involved:**  
  - Prompt Refiner (LangChain LLM node)  
  - Refiner Model (OpenAI GPT-5 Chat)  
  - JSON Output Parser  
  - Text-to-Video Call  
  - Notes: Prompt Refiner, JSON Parser, Text-to-Video

- **Node Details:**  
  - **Prompt Refiner**  
    - Type: LangChain Chain LLM Node  
    - Role: Submits user prompt, aspect ratio, and duration to GPT-5 with detailed instructions to produce an enhanced cinematic prompt.  
    - Config: Includes a comprehensive system prompt guiding GPT-5 to output JSON with fields: prompt (50-4000 chars), aspect_ratio (16:9 or 9:16), duration (4, 8, or 12 seconds).  
    - Input: User form data (prompt, aspect ratio, length).  
    - Output: Raw GPT-5 response.  
    - Edge cases: GPT-5 timeout, malformed responses, rate limits.

  - **Refiner Model**  
    - Type: OpenAI GPT-5 Chat Node (underlying model for Prompt Refiner).  
    - Role: Executes the GPT-5 prompt refinement.  
    - Config: Uses OpenAI credentials with GPT-5 model.  
    - Input/Output: Connected internally to Prompt Refiner node.

  - **JSON Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Validates and extracts GPT-5 JSON response against a strict schema ensuring required fields and correct formats.  
    - Config: Schema enforces prompt length, aspect ratio enum, and duration enum.  
    - Input: GPT-5 output.  
    - Output: Parsed JSON with clean parameters for API call.  
    - Edge cases: Parsing errors, invalid schema data.

  - **Text-to-Video Call**  
    - Type: HTTP Request  
    - Role: Sends refined prompt, aspect ratio, duration, and resolution=720p to fal.ai text-to-video API endpoint (appends `/pro` for sora-2-pro).  
    - Config: JSON POST body with cleaned prompt (line breaks removed), credentials via HTTP header auth.  
    - Input: Parsed prompt JSON from JSON Output Parser.  
    - Output: JSON with request_id.  
    - Edge cases: API failures, auth errors, malformed request.

---

#### 2.5 Status Polling Loop

- **Overview:**  
  Implements asynchronous polling with a wait cycle to check the status of video generation until completion is reported.

- **Nodes Involved:**  
  - Wait 60 Seconds  
  - Status Check  
  - Status Router  
  - Note: Polling Loop1

- **Node Details:**  
  - **Wait 60 Seconds**  
    - Type: Wait  
    - Role: Pauses workflow execution for 60 seconds before next status check.  
    - Input: Triggered after video generation API calls.  
    - Output: Triggers Status Check node.  
    - Edge cases: Excessive polling may cause rate limits.

  - **Status Check**  
    - Type: HTTP Request  
    - Role: Calls fal.ai API to get current status of video generation by request_id.  
    - Config: GET request to `/requests/{request_id}/status` with authorization.  
    - Input: request_id from prior API call.  
    - Output: JSON with status field.  
    - Edge cases: API errors, invalid request_id.

  - **Status Router**  
    - Type: Switch  
    - Role: Routes workflow based on status field: "COMPLETED" proceeds to video retrieval; others loop back to wait.  
    - Config: Two outputs: "Done" if status = COMPLETED; "Progress" otherwise.  
    - Input: Status Check output.  
    - Output: Routes to Retrieve Video or Wait 60 Seconds nodes.  
    - Edge cases: Unexpected statuses causing indefinite loops.

---

#### 2.6 Video Retrieval & Redirect

- **Overview:**  
  Retrieves final video metadata and redirects the user’s browser to the video URL.

- **Nodes Involved:**  
  - Retrieve Video  
  - Video Redirect

- **Node Details:**  
  - **Retrieve Video**  
    - Type: HTTP Request  
    - Role: Fetches final video details from fal.ai API for the given request_id.  
    - Config: GET request to `/requests/{request_id}` with authorization.  
    - Input: Completed status request_id.  
    - Output: JSON containing video URL and metadata.  
    - Edge cases: API errors, missing video URL.

  - **Video Redirect**  
    - Type: Form Node (completion operation)  
    - Role: Sends HTTP redirect response to user’s browser pointing to final video URL.  
    - Config: Redirect URL set dynamically from Retrieve Video node’s JSON path `video.url`.  
    - Input: Output of Retrieve Video node.  
    - Output: HTTP redirect response to client.  
    - Edge cases: Missing or invalid video URL causing redirect failure.

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                         | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                         |
|---------------------|-------------------------------|---------------------------------------|-----------------------|---------------------------|---------------------------------------------------------------------------------------------------|
| Video Input Form     | Form Trigger                  | Captures user inputs via web form     | (Start)               | Input Mode Router         | ## 📝 Video Input Form Purpose: Captures user prompt, ratio, model, duration, image.              |
| Input Mode Router    | Switch                       | Routes flow based on presence of image| Video Input Form      | Temp Image Upload, Prompt Refiner | ## 🔀 Input Mode Router Branches: image-to-video if file, else text-to-video.                       |
| Temp Image Upload    | HTTP Request                 | Uploads image to tmpfiles.org          | Input Mode Router (Image to Video) | Image-to-Video Call       | ## 🖼️ Temp Image Upload Uploads reference image to tmpfiles.org; swaps URL for direct access.     |
| Image-to-Video Call  | HTTP Request                 | Calls fal.ai image-to-video API        | Temp Image Upload     | Wait 60 Seconds           | ## 🖼️ Image-to-Video Call Sends raw prompt + image URL to fal.ai Sora 2 image endpoint.           |
| Prompt Refiner       | LangChain Chain LLM          | Enhances text prompt with GPT-5        | Input Mode Router (Text to Video)  | Text-to-Video Call        | ## 🤖 Prompt Refiner Uses GPT-5 to refine prompt for cinematic quality; outputs JSON.             |
| Refiner Model        | OpenAI GPT-5 Chat Node       | Executes GPT-5 prompt refinement       | Prompt Refiner (internal) | Prompt Refiner           |                                                                                                   |
| JSON Output Parser   | LangChain Output Parser      | Validates GPT-5 JSON output             | Prompt Refiner        | Text-to-Video Call        | ## 🔍 JSON Output Parser Validates GPT-5 response against schema (prompt length, ratio, duration). |
| Text-to-Video Call   | HTTP Request                 | Calls fal.ai text-to-video API          | JSON Output Parser    | Wait 60 Seconds           | ## 🎥 Text-to-Video Call Submits refined prompt to fal.ai text endpoint; returns request_id.      |
| Wait 60 Seconds      | Wait                         | Waits 60 seconds before polling again  | Image-to-Video Call, Text-to-Video Call, Status Router (Progress) | Status Check             | ## ⏳ Status Polling Loop Waits 60s, checks Sora status, loops until COMPLETED.                    |
| Status Check         | HTTP Request                 | Checks video generation status          | Wait 60 Seconds       | Status Router             |                                                                                                   |
| Status Router        | Switch                       | Routes based on generation status       | Status Check          | Retrieve Video, Wait 60 Seconds |                                                                                                   |
| Retrieve Video       | HTTP Request                 | Retrieves final video details            | Status Router (Done)  | Video Redirect            |                                                                                                   |
| Video Redirect       | Form (completion)            | Redirects user to final video URL        | Retrieve Video        | (End)                     |                                                                                                   |
| Note: Mode Router    | Sticky Note                  | Documentation for Input Mode Router     |                       |                           | ## 🔀 Input Mode Router Branches to image or text-to-video with GPT-5 refinement.                  |
| Note: Image Upload   | Sticky Note                  | Documentation for Temp Image Upload     |                       |                           | ## 🖼️ Temp Image Upload Uploads reference image to tmpfiles.org with direct API URL adjustment.   |
| Note: Prompt Refiner | Sticky Note                  | Documentation for Prompt Refinement     |                       |                           | ## 🤖 Prompt Refiner Uses GPT-5 for cinematic prompt enhancement; outputs structured JSON.        |
| Note: JSON Parser    | Sticky Note                  | Documentation for JSON Output Parsing   |                       |                           | ## 🔍 JSON Output Parser Validates GPT-5 output for schema compliance.                            |
| Note: Text-to-Video  | Sticky Note                  | Documentation for Text-to-Video API call|                       |                           | ## 🎥 Text-to-Video Call Sends refined prompt to fal.ai text-to-video or pro endpoint.            |
| Note: Image-to-Video | Sticky Note                  | Documentation for Image-to-Video API call|                       |                           | ## 🖼️ Image-to-Video Call Sends raw prompt + image URL to fal.ai image-to-video or pro endpoint.  |
| Note: Form Trigger1  | Sticky Note                  | Documentation for Video Input Form       |                       |                           | ## 📝 Video Input Form Captures prompt, ratio, model, duration, optional image; validates input.  |
| Note: Polling Loop1  | Sticky Note                  | Documentation for polling loop           |                       |                           | ## ⏳ Status Polling Loop Waits 60s, checks status, loops until COMPLETED status is received.     |
| Overview Note8       | Sticky Note                  | Workflow overview, prerequisites, setup |                       |                           | # 🎬 Sora 2 Video Generator via Fal with GPT-5 Refinement — full project overview and instructions.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Name: Video Input Form  
   - Configure fields:  
     - Prompt (text, required)  
     - Aspect Ratio (dropdown, required): options "9:16 (vertical)", "16:9 (Horizontal)"  
     - Model (checkbox, required, limit to one): options "sora-2", "sora-2-pro"  
     - Length (checkbox, required, limit to one): options "4s", "8s", "12s"  
     - Image (file upload, optional): accept `.jpg, .jpeg, .png`  
   - Set webhook activation on workflow start.

2. **Add a Switch Node**  
   - Name: Input Mode Router  
   - Condition: Check if `Image.filename` is not empty (image uploaded)  
   - Outputs:  
     - "Image to Video" if image present  
     - "Text to Video" if no image  

3. **Image-to-Video Branch:**  
   3.1 Create an HTTP Request node  
       - Name: Temp Image Upload  
       - Method: POST  
       - URL: `https://tmpfiles.org/api/v1/upload`  
       - Content Type: multipart/form-data  
       - Body Parameter: file field set to `Image` from form data  
   3.2 Create an HTTP Request node  
       - Name: Image-to-Video Call  
       - Method: POST  
       - URL: `https://queue.fal.run/fal-ai/sora-2/image-to-video` (append `/pro` if model is sora-2-pro)  
       - JSON Body:  
         ```json
         {
           "prompt": "{{ raw user prompt without line breaks }}",
           "resolution": "auto",
           "aspect_ratio": "{{ aspect ratio cleaned from form field }}",
           "duration": {{ duration in seconds }},
           "image_url": "{{ tmpfiles uploaded URL adjusted by replacing '.org/' with '.org/dl/' }}"
         }
         ```  
       - Authentication: HTTP Header Auth with fal.ai API key  

4. **Text-to-Video Branch:**  
   4.1 Create a LangChain Chain LLM node  
       - Name: Prompt Refiner  
       - Model: GPT-5 via OpenAI Credentials  
       - Input: User prompt, aspect ratio, duration from form  
       - System prompt instructs GPT-5 to convert input into detailed cinematic prompt with JSON output schema containing prompt, aspect_ratio, duration  
   4.2 Create a LangChain Structured Output Parser node  
       - Name: JSON Output Parser  
       - Schema: Object with required fields prompt (string 50-4000 chars), aspect_ratio ("16:9" or "9:16"), duration (4,8,12)  
       - Input: GPT-5 raw output  
   4.3 Create an HTTP Request node  
       - Name: Text-to-Video Call  
       - Method: POST  
       - URL: `https://queue.fal.run/fal-ai/sora-2/text-to-video` (append `/pro` if model is sora-2-pro)  
       - JSON Body:  
         ```json
         {
           "prompt": "{{ refined prompt without line breaks }}",
           "resolution": "720p",
           "aspect_ratio": "{{ aspect_ratio }}",
           "duration": {{ duration }}
         }
         ```  
       - Authentication: HTTP Header Auth with fal.ai API key  

5. **Polling Loop for Both Branches:**  
   5.1 Add a Wait node  
       - Name: Wait 60 Seconds  
       - Duration: 60 seconds  
   5.2 Add HTTP Request node  
       - Name: Status Check  
       - Method: GET  
       - URL: `https://queue.fal.run/fal-ai/sora-2/requests/{{ request_id }}/status`  
       - Authentication: fal.ai API key  
   5.3 Add Switch node  
       - Name: Status Router  
       - Condition: If status == "COMPLETED" route to next step; else loop back to Wait 60 Seconds  

6. **Retrieve Final Video:**  
   6.1 Add HTTP Request node  
       - Name: Retrieve Video  
       - Method: GET  
       - URL: `https://queue.fal.run/fal-ai/sora-2/requests/{{ request_id }}`  
       - Authentication: fal.ai API key  
   6.2 Add Form node (completion type)  
       - Name: Video Redirect  
       - Operation: Completion  
       - Redirect URL: `{{ video.url }}` from Retrieve Video response  
       - Respond with redirect to user  

7. **Credentials Setup:**  
   - **fal.ai API:** Create HTTP Header Auth credential with header "Authorization" and value "Key [Your API Key]"  
   - **OpenAI API:** Create OpenAI API credential with GPT-5 model enabled  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow enables cinematic AI video generation using fal.ai’s Sora 2 endpoints with GPT-5 prompt enhancement.| Overview Note8 sticky note in workflow. |
| fal.ai API key must have "sora-2" permissions enabled; ensure key is kept secure.| Overview Note8 prerequisites section. |
| GPT-5 prompt refiner uses a complex system prompt for cinematic storytelling enhancement; output strict JSON format.| Note: Prompt Refiner sticky note. |
| tmpfiles.org is used as a temporary image host; URLs are adjusted from `/api/v1/upload` response for direct API consumption.| Note: Image Upload sticky note. |
| Polling loop waits 60 seconds between status checks to avoid rate limits; adjust timing if generating longer clips.| Note: Polling Loop1 sticky note. |
| Form trigger validates mandatory inputs and supports optional image uploads up to 10MB; large images may fail and cause upload errors.| Note: Form Trigger1 sticky note. |
| fal.ai text-to-video endpoint supports 720p resolution only; image-to-video uses auto resolution.| Notes: Text-to-Video and Image-to-Video sticky notes. |

---

**Disclaimer:** The text provided stems solely from an automated workflow created with n8n, an integration and automation platform. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.