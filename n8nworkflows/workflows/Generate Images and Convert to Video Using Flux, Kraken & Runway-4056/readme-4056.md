Generate Images and Convert to Video Using Flux, Kraken & Runway

https://n8nworkflows.xyz/workflows/generate-images-and-convert-to-video-using-flux--kraken---runway-4056


# Generate Images and Convert to Video Using Flux, Kraken & Runway

### 1. Workflow Overview

This workflow automates the process of generating AI-based images, hosting them publicly, and converting them into videos using AI services. It targets creators, automation enthusiasts, and indie hackers who want to produce image-based videos cost-effectively and with minimal manual intervention.

The workflow is logically organized into the following blocks:

- **1.1 Input Trigger:** Manual or scheduled initiation of the workflow.
- **1.2 Image Generation:** Producing AI-generated images using Flux via two possible API options (official Blackforest Labs API or RapidAPI).
- **1.3 Image Retrieval and Hosting:** Downloading the generated image and uploading it to Kraken.io to obtain a publicly accessible URL.
- **1.4 Video Generation:** Using RunwayML APIs (official or via RapidAPI) to convert the hosted image into a video.
- **1.5 Video Retrieval and Polling:** Polling for video generation status and downloading the final video URL once ready.
- **1.6 Wait and Retry Logic:** Implemented wait nodes to handle asynchronous API processing and polling intervals.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

**Overview:**  
This block initiates the workflow manually or by schedule. It serves as the entry point for the image-to-video generation process.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow on user command.  
  - *Configuration:* No parameters; passive trigger node.  
  - *Inputs:* None  
  - *Outputs:* Connects to `generate image (flux)` node.  
  - *Failure Modes:* None (manual trigger)  
  - *Sub-workflow:* None  

---

#### 2.2 Image Generation

**Overview:**  
Generates an AI image using Flux service with two options: official Blackforest Labs API or Flux via RapidAPI. The user can select either method for image creation.

**Nodes Involved:**  
- generate image (flux)  
- generate image (flux-rapid-api)  
- Wait  
- get_image_url  
- get_image  
- Get Image3

**Node Details:**

- **generate image (flux)**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to Blackforest Labs Flux API to generate an image based on a text prompt and configuration (resolution, seed, safety tolerance).  
  - *Config:* Uses JSON body with customizable prompt, width 768, height 1024, seed 42, output format JPEG. Header includes `x-key` API key placeholder.  
  - *Input:* From manual trigger  
  - *Output:* Image generation ID sent to `Wait` node  
  - *Failure Modes:* Authentication error (invalid key), API errors, malformed JSON, network issues.  
  - *Version:* n8n v4.2 HTTP Request node  

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses workflow for 15 seconds to allow image generation to complete asynchronously.  
  - *Config:* Wait for 15 seconds.  
  - *Input:* From `generate image (flux)`  
  - *Output:* Triggers `get_image_url` node.  
  - *Failure Modes:* None expected; timer node.

- **get_image_url**  
  - *Type:* HTTP Request  
  - *Role:* Polls Blackforest Labs API to retrieve generated image URL using the ID from `generate image (flux)`.  
  - *Config:* GET request with dynamic URL using the image ID. Includes `x-key` header.  
  - *Input:* From `Wait`  
  - *Output:* Passes image URL to `get_image`  
  - *Failure Modes:* Timeout if image not ready, auth error, network errors.

- **get_image**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the actual image file from the URL obtained.  
  - *Config:* GET request to image URL, expects full HTTP response as file. Includes `auth` header placeholder.  
  - *Input:* From `get_image_url`  
  - *Output:* Sends image file data to `upload to kraken` for hosting.  
  - *Failure Modes:* URL invalid, download failure, auth error.

- **generate image (flux-rapid-api)**  
  - *Type:* HTTP Request  
  - *Role:* Alternative image generation via Flux RapidAPI. POST request with prompt, style, and size parameters.  
  - *Config:* Headers include `x-rapidapi-host` and `x-rapidapi-key`. Body parameters include prompt, style_id, size.  
  - *Input:* Separate path, can be triggered manually or integrated.  
  - *Output:* Sends response to `Get Image3`  
  - *Failure Modes:* API key errors, rate limiting, malformed request.

- **Get Image3**  
  - *Type:* HTTP Request  
  - *Role:* Downloads generated image from RapidAPI response URL.  
  - *Config:* Uses header with RapidAPI key, URL from JSON response.  
  - *Input:* From `generate image (flux-rapid-api)`  
  - *Output:* Sends image to `upload to kraken1`  
  - *Failure Modes:* Download errors, invalid URL.

---

#### 2.3 Image Hosting Using Kraken.io

**Overview:**  
Uploads the downloaded image file to Kraken.io to obtain a public URL, which is required by RunwayML for video generation.

**Nodes Involved:**  
- upload to kraken  
- upload to kraken1

**Node Details:**

- **upload to kraken**  
  - *Type:* HTTP Request  
  - *Role:* Uploads image binary data as multipart form-data to Kraken API; receives public URL in response.  
  - *Config:* POST to `https://api.kraken.io/v1/upload` with authentication JSON in form-data field `data` (includes API key and secret). The image file is passed as `upload` binary data.  
  - *Input:* From `get_image` (official Flux flow)  
  - *Output:* Sends Kraken public URL to `image to video (runway)`  
  - *Failure Modes:* API auth failure, file upload errors, network timeout.

- **upload to kraken1**  
  - *Type:* HTTP Request  
  - *Role:* Same as `upload to kraken` but for RapidAPI Flux image flow.  
  - *Config:* Similar multipart form-data, credentials placeholders.  
  - *Input:* From `Get Image3`  
  - *Output:* To `image to video (runway-rapid-api)`  
  - *Failure Modes:* Same as above.

---

#### 2.4 Video Generation Using RunwayML

**Overview:**  
Converts the hosted image into a video using RunwayML’s AI video generation APIs. Supports both official API and RapidAPI.

**Nodes Involved:**  
- image to video (runway)  
- image to video (runway-rapid-api)  
- Download Video  
- Get Video Generation Status1  
- Confirm Generation Status  
- 1 minute3  
- 1 minute2

**Node Details:**

- **image to video (runway)**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to official Runway API’s `/image_to_video` endpoint with Kraken public image URL and video generation parameters (model, ratio, duration, prompt text).  
  - *Config:* Headers include `X-Runway-Version` and `Authorization` Bearer token. Body parameters specify image URL, model "gen3a_turbo", ratio 768:1280, duration 10 seconds, and prompt.  
  - *Input:* From `upload to kraken`  
  - *Output:* Proceeds to `1 minute2` wait node for asynchronous processing.  
  - *Failure Modes:* Auth errors, API rate limits, invalid parameters.

- **image to video (runway-rapid-api)**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to RapidAPI endpoint for Runway video generation using the Kraken image URL and additional parameters like text prompt, model type, motion, seed, time, etc.  
  - *Config:* Headers with RapidAPI host and key, body parameters customized for video generation.  
  - *Input:* From `upload to kraken1`  
  - *Output:* Connects to `1 minute3` wait node.  
  - *Failure Modes:* API key errors, network issues.

- **1 minute2**  
  - *Type:* Wait  
  - *Role:* Waits 1 minute to allow Runway official API video generation to complete.  
  - *Input:* From `image to video (runway)` or from `Download Video` node on retry.  
  - *Output:* Triggers `Download Video` node.  
  - *Failure Modes:* Potential delay in processing requiring longer waits.

- **Download Video**  
  - *Type:* HTTP Request  
  - *Role:* Polls Runway official API to get the status and URL of the generated video using the task ID.  
  - *Config:* GET request with `X-Runway-Version` and Bearer `Authorization` headers, URL constructed with the task ID.  
  - *Input:* From `1 minute2`  
  - *Output:* On failure or incomplete status, triggers `1 minute2` again (retry loop). On success, ends or continues downstream automation.  
  - *Failure Modes:* API errors, invalid ID, network.

- **Get Video Generation Status1**  
  - *Type:* HTTP Request  
  - *Role:* Polls Runway RapidAPI endpoint to check if video generation is complete by UUID.  
  - *Config:* GET with query parameter `uuid` and RapidAPI headers.  
  - *Input:* From `1 minute3` wait node  
  - *Output:* To `Confirm Generation Status` switch node  
  - *Failure Modes:* API timeout, invalid UUID.

- **Confirm Generation Status**  
  - *Type:* Switch  
  - *Role:* Checks if the returned status contains "success" to determine if video is ready.  
  - *Config:* String contains check on `$json.status`. If contains "success", outputs to "completed", else "pending".  
  - *Input:* From `Get Video Generation Status1`  
  - *Output:*  
    - "completed" path ends or continues downstream  
    - "pending" path loops back to `1 minute3` wait node for retry after 1 minute  
  - *Failure Modes:* Incorrect status parsing, empty responses.

- **1 minute3**  
  - *Type:* Wait  
  - *Role:* Waits 1 minute before retrying video status check.  
  - *Input:* From "pending" output of `Confirm Generation Status` or from `image to video (runway-rapid-api)`  
  - *Output:* Triggers `Get Video Generation Status1` for next poll.  
  - *Failure Modes:* None expected.

---

### 3. Summary Table

| Node Name                      | Node Type         | Functional Role                         | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                  |
|-------------------------------|-------------------|---------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger    | Workflow start trigger                 | None                         | generate image (flux)          |                                                                                                              |
| generate image (flux)          | HTTP Request      | Generate image via Flux (Blackforest) | When clicking ‘Test workflow’| Wait                          | ## Generate Image: Using Flux (Blackforest labs Option)                                                      |
| Wait                          | Wait              | Pause for image generation             | generate image (flux)         | get_image_url                 |                                                                                                              |
| get_image_url                 | HTTP Request      | Retrieve generated image URL           | Wait                         | get_image                     |                                                                                                              |
| get_image                     | HTTP Request      | Download the generated image           | get_image_url                | upload to kraken              |                                                                                                              |
| upload to kraken              | HTTP Request      | Upload image to Kraken.io for hosting  | get_image                    | image to video (runway)       |                                                                                                              |
| image to video (runway)       | HTTP Request      | Generate video via Runway official API | upload to kraken             | 1 minute2                    | ## Image to Video: Using Runway (official api)                                                              |
| 1 minute2                    | Wait              | Wait for video generation completion   | image to video (runway), Download Video | Download Video        |                                                                                                              |
| Download Video               | HTTP Request      | Poll Runway official API for video URL | 1 minute2                   | 1 minute2 (retry)             |                                                                                                              |
| generate image (flux-rapid-api)| HTTP Request     | Generate image via Flux RapidAPI       | (alternate entry)            | Get Image3                   | ## Flux (Rapid API Endpoint)                                                                                   |
| Get Image3                   | HTTP Request      | Download generated image from RapidAPI | generate image (flux-rapid-api)| upload to kraken1          |                                                                                                              |
| upload to kraken1            | HTTP Request      | Upload RapidAPI image to Kraken.io     | Get Image3                   | image to video (runway-rapid-api) |                                                                                                              |
| image to video (runway-rapid-api) | HTTP Request | Generate video via Runway RapidAPI     | upload to kraken1            | 1 minute3                    | ## Image to Video: Using Runway (Rapid API)                                                                  |
| 1 minute3                   | Wait              | Wait before polling RapidAPI video status | image to video (runway-rapid-api), Confirm Generation Status (pending) | Get Video Generation Status1 |                                                                                                              |
| Get Video Generation Status1 | HTTP Request      | Poll Runway RapidAPI for video status  | 1 minute3                   | Confirm Generation Status     |                                                                                                              |
| Confirm Generation Status    | Switch            | Check if video generation succeeded    | Get Video Generation Status1 | 1 minute3 (pending), End (completed) |                                                                                                              |
| Sticky Note                 | Sticky Note       | Note for Flux Blackforest API usage    | None                        | None                         | ## Generate Image: Using Flux (Blackforest labs Option)                                                      |
| Sticky Note1                | Sticky Note       | Note for Flux Rapid API usage           | None                        | None                         | ## Flux (Rapid API Endpoint)                                                                                   |
| Sticky Note2                | Sticky Note       | Note for Runway RapidAPI video generation | None                        | None                         | ## Image to Video: Using Runway (Rapid API)                                                                  |
| Sticky Note10               | Sticky Note       | Note for Runway official API video generation | None                        | None                         | ## Image to Video: Using Runway (official api)                                                              |
| Sticky Note3                | Sticky Note       | General note on using RapidAPI endpoints | None                        | None                         | ## Using Rapid API Endpoints                                                                                   |
| Sticky Note4                | Sticky Note       | General note on using official APIs    | None                        | None                         | ## Using official APIs                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.  

2. **Add HTTP Request Node: `generate image (flux)`**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.us1.bfl.ai/v1/flux-pro-1.1`  
   - Headers: Add `x-key` with your Blackforest API key (string, e.g., `"your-key"`)  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "prompt": "your_text_to_image_prompt",
       "width": 768,
       "height": 1024,
       "prompt_upsampling": false,
       "seed": 42,
       "safety_tolerance": 2,
       "output_format": "jpeg"
     }
     ```  
   - Connect Manual Trigger → `generate image (flux)`

3. **Add Wait Node: `Wait`**  
   - Type: Wait  
   - Duration: 15 seconds  
   - Connect `generate image (flux)` → `Wait`

4. **Add HTTP Request Node: `get_image_url`**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression to get result ID:  
     `https://api.us1.bfl.ai/v1/get_result?id={{ $json["id"] }}` (replace `$json["id"]` with the output from `generate image (flux)`)  
   - Headers: `x-key` with your Blackforest API key  
   - Connect `Wait` → `get_image_url`

5. **Add HTTP Request Node: `get_image`**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression `={{ $json.result.sample }}` to get image URL from previous node  
   - Response Format: File (full response)  
   - Headers: `auth` with your API key (if required)  
   - Connect `get_image_url` → `get_image`

6. **Add HTTP Request Node: `upload to kraken`**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.kraken.io/v1/upload`  
   - Content Type: `multipart/form-data`  
   - Body Parameters:  
     - `data` (string): JSON string with Kraken auth and wait flag:  
       `{"auth":{"api_key":"your_api_key","api_secret":"your_api_secret"},"wait":true}`  
     - `upload`: Set as binary data input (image from `get_image`)  
   - Connect `get_image` → `upload to kraken`

7. **Add HTTP Request Node: `image to video (runway)`**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.dev.runwayml.com/v1/image_to_video`  
   - Headers:  
     - `X-Runway-Version`: `2024-11-06`  
     - `Authorization`: Bearer token (`"Your-bearer-key"`)  
   - Body Parameters:  
     - `promptImage`: Expression from `upload to kraken` output (public URL)  
     - `model`: `"gen3a_turbo"`  
     - `ratio`: `"768:1280"`  
     - `duration`: `"10"`  
     - `promptText`: Text prompt for video generation  
   - Connect `upload to kraken` → `image to video (runway)`

8. **Add Wait Node: `1 minute2`**  
   - Type: Wait  
   - Duration: 1 minute  
   - Connect `image to video (runway)` → `1 minute2`

9. **Add HTTP Request Node: `Download Video`**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression to fetch video status by ID:  
     `https://api.dev.runwayml.com/v1/tasks/{{ $json.id }}`  
   - Headers:  
     - `X-Runway-Version`: `2024-11-06`  
     - `Authorization`: Bearer token (`"your-bearer-key"`)  
   - On Error: Continue (to allow retries)  
   - Connect `1 minute2` → `Download Video`

10. **Connect `Download Video` node’s success output back to `1 minute2`**  
    - This creates a retry loop until video is ready.

---

**Alternate Flux via RapidAPI path:**

11. **Add HTTP Request Node: `generate image (flux-rapid-api)`**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://ai-text-to-image-generator-flux-free-api.p.rapidapi.com/aaaaaaaaaaaaaaaaaiimagegenerator/quick.php`  
    - Headers:  
      - `x-rapidapi-host`: `ai-text-to-image-generator-flux-free-api.p.rapidapi.com`  
      - `x-rapidapi-key`: Your RapidAPI key  
    - Body Parameters:  
      - `prompt`: Your text prompt  
      - `style_id`: `"1"`  
      - `size`: `"1-1"`  
    - Connect manual trigger or other entry point as desired.

12. **Add HTTP Request Node: `Get Image3`**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: Use expression from previous node’s response: `{{$json.final_result[0].origin}}`  
    - Headers: `x-rapidapi-key` with your RapidAPI key  
    - Connect `generate image (flux-rapid-api)` → `Get Image3`

13. **Add HTTP Request Node: `upload to kraken1`**  
    - Same configuration as `upload to kraken` but connected to `Get Image3` output.  
    - Connect `Get Image3` → `upload to kraken1`

14. **Add HTTP Request Node: `image to video (runway-rapid-api)`**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://runwayml.p.rapidapi.com/generate/imageDescription`  
    - Headers:  
      - `x-rapidapi-host`: `runwayml.p.rapidapi.com`  
      - `x-rapidapi-key`: Your RapidAPI key  
    - Body Parameters:  
      - `text_prompt`: Your video prompt  
      - `img_prompt`: Kraken public image URL from `upload to kraken1` output  
      - Additional parameters: `model`, `image_as_end_frame`, `flip`, `motion`, `seed`, `time` as per your requirements.  
    - Connect `upload to kraken1` → `image to video (runway-rapid-api)`

15. **Add Wait Node: `1 minute3`**  
    - Type: Wait  
    - Duration: 1 minute  
    - Connect `image to video (runway-rapid-api)` → `1 minute3`

16. **Add HTTP Request Node: `Get Video Generation Status1`**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://runwayml.p.rapidapi.com/status` with query param `uuid={{ $json.uuid }}`  
    - Headers:  
      - `x-rapidapi-host`: `runwayml.p.rapidapi.com`  
      - `x-rapidapi-key`: Your RapidAPI key  
    - Connect `1 minute3` → `Get Video Generation Status1`

17. **Add Switch Node: `Confirm Generation Status`**  
    - Type: Switch  
    - Condition: Check if `$json.status` contains `"success"` (case-sensitive)  
    - Output "completed": End or proceed downstream  
    - Output "pending": Loop back to `1 minute3` wait node for retry  
    - Connect `Get Video Generation Status1` → `Confirm Generation Status`  
    - Connect `Confirm Generation Status` "pending" → `1 minute3`

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                      |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| You can schedule this workflow to run daily or weekly for automated video generation.               | Scheduling in n8n                                                |
| Combine with Google Sheets or Notion to process bulk prompts for batch video creation.              | Use n8n integrations with Sheets or Notion                         |
| Use consistent visual styles for brand identity in generated videos.                                | Creative branding advice                                            |
| Workflow is cost-effective and suitable for indie projects or experimental AI content generation.  | Target audience note                                               |
| Flux API Docs: [https://docs.bfl.ml/](https://docs.bfl.ml/)                                         | Official Flux API documentation                                    |
| Flux RapidAPI: [https://rapidapi.com/poorav925/api/ai-text-to-image-generator-flux-free-api/](https://rapidapi.com/poorav925/api/ai-text-to-image-generator-flux-free-api/) | RapidAPI Flux API documentation                                    |
| RunwayML Official API Docs: [https://dev.runwayml.com/](https://dev.runwayml.com/)                  | RunwayML developer documentation                                  |
| Runway RapidAPI Docs: [https://rapidapi.com/fortunehoppers/api/runwayml/](https://rapidapi.com/fortunehoppers/api/runwayml/) | Runway RapidAPI documentation                                     |
| Kraken.io API Credentials: [https://kraken.io/account/api-credentials](https://kraken.io/account/api-credentials) | Kraken.io API key management                                      |
| Contact for assistance or custom workflow development: Twitter [@juppfy](https://x.com/juppfy), Email: joseph@uppfy.com | Support and consultancy contact                                   |

---

**Disclaimer:** The text and workflow are created exclusively with n8n, respecting all applicable content policies, and processing only legal and public data.