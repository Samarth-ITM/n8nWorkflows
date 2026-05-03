Generate AI Avatar Videos from Text with HeyGen and Upload to YouTube

https://n8nworkflows.xyz/workflows/generate-ai-avatar-videos-from-text-with-heygen-and-upload-to-youtube-8622


# Generate AI Avatar Videos from Text with HeyGen and Upload to YouTube

### 1. Workflow Overview

This workflow automates the process of generating AI avatar videos from provided text scripts using the HeyGen platform, then uploads the resulting video to YouTube. It is designed for use cases such as content creators, marketers, or educators who want to convert text-based scripts into engaging video content with AI avatars and voices.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Receives incoming HTTP POST requests with text and metadata, validates and sanitizes the input.
- **1.2 Parameter Initialization:** Sets up required parameters and secrets for the HeyGen API call including avatar and voice IDs.
- **1.3 HeyGen Video Generation Request:** Submits the video generation job to HeyGen.
- **1.4 Polling and Status Checking:** Waits and polls HeyGen API until video processing is complete.
- **1.5 Download and Upload:** Downloads the generated video and uploads it to YouTube.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block handles the inbound HTTP POST request, validates the secret token, parses the JSON payload, sanitizes the text script to remove unwanted characters, and prepares the data for further processing.

**Nodes Involved:**  
- Sticky - Webhook Contract & Security  
- WebHook  
- Code  
- Sticky - Normalize Text for TTS

**Node Details:**  

- **Sticky - Webhook Contract & Security**  
  - Type: Sticky Note (documentation only)  
  - Role: Documents that the webhook expects a secret validation, JSON parsing, and immediate 200 response.  
  - No inputs or outputs; purely informational.  
  - Edge Cases: N/A  

- **WebHook**  
  - Type: Webhook Trigger  
  - Role: Receives external POST requests at path `52048302-06bd-4eae-9b004c5ed17d`.  
  - Config: HTTP method POST, no special options.  
  - Outputs JSON body of incoming request.  
  - Edge Cases: Invalid HTTP method, malformed JSON, missing secret validation (not explicitly coded here).  

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Sanitizes `body.content` by removing newline and carriage return characters; passes `title` as is.  
  - Key Expression: `item.json.body.content.replaceAll("\n", "").replaceAll("\r", "")`  
  - Input: JSON from WebHook node  
  - Output: JSON with sanitized `content` and original `title`  
  - Edge Cases: Missing `body` or `content` keys causes runtime error; no null checks.  

- **Sticky - Normalize Text for TTS**  
  - Type: Sticky Note  
  - Role: Documents that the script is sanitized for HeyGen compatibility.  
  - No inputs or outputs.  

---

#### 2.2 Parameter Initialization

**Overview:**  
Initializes variables and secrets such as API keys, avatar IDs, voice IDs, and default parameters required for the HeyGen API call.

**Nodes Involved:**  
- Sticky - Parameter Staging (Secrets & IDs)  
- Setup Heygen Parameters (disabled)

**Node Details:**  

- **Sticky - Parameter Staging (Secrets & IDs)**  
  - Type: Sticky Note  
  - Role: Explains initialization of variables like `script`, `voice_id`, `avatar_id`, `aspect`, and default value of `sub=false`.  
  - No inputs or outputs.  

- **Setup Heygen Parameters** (Disabled)  
  - Type: Set (disabled)  
  - Role: Intended to set API key, voice/ avatar IDs, and pass title/content. However, this node is disabled, so parameters must be set elsewhere or manually.  
  - Parameters: Sets fields `x-api-key`, `voice_id`, `avatar_id`, along with `title` and `content` from previous node.  
  - Edge Cases: Disabled state means parameters might be injected dynamically elsewhere or require manual configuration.

---

#### 2.3 HeyGen Video Generation Request

**Overview:**  
Submits the video generation request to HeyGen’s API with the avatar and voice parameters, requesting a video with specified dimensions.

**Nodes Involved:**  
- Sticky - Create HeyGen Job (v2)  
- Create Avatar Video (HeyGen)

**Node Details:**  

- **Sticky - Create HeyGen Job (v2)**  
  - Type: Sticky Note  
  - Role: Documents the API endpoint and parameters used to submit video generation request.  
  - No inputs or outputs.  

- **Create Avatar Video (HeyGen)**  
  - Type: HTTP Request  
  - Role: Sends POST request to `https://api.heygen.com/v2/video/generate` with JSON body specifying avatar, voice, text content, and video dimension (1280x720).  
  - Headers: `X-Api-Key` taken dynamically from JSON input.  
  - Inputs: JSON fields `x-api-key`, `avatar_id`, `voice_id`, `content`.  
  - Outputs: JSON response containing `video_id` and status data.  
  - Edge Cases: API authentication errors, invalid parameters, network timeouts.

---

#### 2.4 Polling and Status Checking

**Overview:**  
Implements a polling loop to check the status of the video generation job at HeyGen and waits until the processing is finished before proceeding.

**Nodes Involved:**  
- Sticky - Polling Cycle  
- Wait for Video (HeyGen)  
- Sticky - Polling Cycle1  
- Get Avatar Video Status (HeyGen)  
- If Processing Completed

**Node Details:**  

- **Sticky - Polling Cycle**  
  - Type: Sticky Note  
  - Role: Notes the use of delay to avoid rate limits and adapt wait time for longer videos.  

- **Wait for Video (HeyGen)**  
  - Type: Wait  
  - Role: Pauses workflow for 0.1 minutes (~6 seconds) before polling again.  
  - Inputs: Triggered after video generation request or from If node retry path.  
  - Edge Cases: Too short delay may hit rate limits; too long delays slow workflow.  

- **Sticky - Polling Cycle1**  
  - Type: Sticky Note  
  - Role: Documents that this block retrieves the generated video status from HeyGen.  

- **Get Avatar Video Status (HeyGen)**  
  - Type: HTTP Request  
  - Role: Calls HeyGen API `https://api.heygen.com/v1/video_status.get` with query parameter `video_id` to get current processing status.  
  - Headers: Uses same API key as creation request.  
  - Inputs: Reads `video_id` from previous creation node output.  
  - Outputs: JSON with status field.  
  - Edge Cases: API errors, invalid video_id, network issues.  

- **If Processing Completed**  
  - Type: If  
  - Role: Checks if `status` field returned by HeyGen is NOT equal to `"processing"`.  
  - True branch: Processing finished, proceed to download.  
  - False branch: Still processing, loop back to wait and poll again.  
  - Edge Cases: Unexpected status values, JSON path errors.

---

#### 2.5 Download and Upload

**Overview:**  
Downloads the generated video file once ready and uploads it to YouTube using configured OAuth2 credentials.

**Nodes Involved:**  
- Sticky - Download Result (Binary)  
- Download Video  
- Sticky - YouTube Upload (Binary Property & Metadata)  
- Upload a video

**Node Details:**  

- **Sticky - Download Result (Binary)**  
  - Type: Sticky Note  
  - Role: Explains that if the video is ready (status != processing), the workflow attempts to download the video file. Failure to download indicates an error in HeyGen video generation.  

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads the video binary from URL provided in `data.video_url`.  
  - Inputs: URL from HeyGen status response.  
  - Outputs: Binary video data attached to the workflow item.  
  - Edge Cases: Download failure, invalid URL, network errors.  

- **Sticky - YouTube Upload (Binary Property & Metadata)**  
  - Type: Sticky Note  
  - Role: Documents that the next node uploads the binary video data to YouTube with metadata like title.  

- **Upload a video**  
  - Type: YouTube Node  
  - Role: Uploads the binary video to YouTube under category "Education" (categoryId 28), region US.  
  - Credentials: Uses OAuth2 credential named "YouTube account".  
  - Parameters: Title fetched dynamically from `Setup Heygen Parameters` node.  
  - Edge Cases: OAuth token expiration, upload failures, quota limits.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                          |
|-------------------------------|---------------------|----------------------------------------|-----------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Sticky - Webhook Contract & Security | Sticky Note        | Documentation of webhook contract       | None                        | None                           | Inbound trigger for external start. Validate secret; parse JSON; return 200 immediately.           |
| WebHook                       | Webhook             | Receive external POST requests          | None                        | Code                           |                                                                                                    |
| Code                         | Code                | Sanitize text script content            | WebHook                     | Setup Heygen Parameters         |                                                                                                    |
| Sticky - Normalize Text for TTS | Sticky Note        | Document script sanitization             | None                        | None                           | Sanitize the video script to ensure HeyGen can use that correctly.                                 |
| Sticky - Parameter Staging (Secrets & IDs) | Sticky Note        | Document secret and parameter setup      | None                        | None                           | Init vars (script, voice_id, avatar_id, aspect, sub=false); set defaults.                          |
| Setup Heygen Parameters       | Set (disabled)      | Set API key, voice/avatar IDs, title, content | Code                        | Create Avatar Video (HeyGen)   |                                                                                                    |
| Sticky - Create HeyGen Job (v2) | Sticky Note        | Document HeyGen video generation request | None                        | None                           | Submit video generation request to HeyGen using API: POST /v2/video/generate with params.          |
| Create Avatar Video (HeyGen)  | HTTP Request        | Submit video generation to HeyGen API   | Setup Heygen Parameters      | Wait for Video (HeyGen)         |                                                                                                    |
| Sticky - Polling Cycle        | Sticky Note         | Document polling delay strategy          | None                        | None                           | Delay polling to avoid rate limits; increase wait for long video generations                       |
| Wait for Video (HeyGen)       | Wait                | Delay before polling status again        | Create Avatar Video (HeyGen), If Processing Completed (false branch) | Get Avatar Video Status (HeyGen) |                                                                                                    |
| Sticky - Polling Cycle1       | Sticky Note         | Document status retrieval from HeyGen    | None                        | None                           | Get status of the generated video from HeyGen.                                                     |
| Get Avatar Video Status (HeyGen) | HTTP Request        | Query HeyGen for video status             | Wait for Video (HeyGen)      | If Processing Completed         |                                                                                                    |
| If Processing Completed       | If                  | Check if video processing is finished    | Get Avatar Video Status (HeyGen) | Download Video (true), Wait for Video (HeyGen) (false) |                                                                                                    |
| Sticky - Download Result (Binary) | Sticky Note        | Document download step and error handling | None                        | None                           | If status is not 'processing' -> Try to download; if download fails it means the video generation in HeyGen errored for whatever reason. |
| Download Video               | HTTP Request        | Download generated video binary           | If Processing Completed     | Upload a video                 |                                                                                                    |
| Sticky - YouTube Upload (Binary Property & Metadata) | Sticky Note        | Document YouTube upload step              | None                        | None                           | Upload the video (downloaded data) to Youtube.                                                     |
| Upload a video                | YouTube             | Upload video to YouTube channel           | Download Video              | None                           |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `52048302-06bd-4eae-9b004c5ed17d`  
   - Purpose: Receive external requests with JSON body containing video `title` and `content`.

2. **Add a Code node connected to the Webhook**  
   - Purpose: Sanitize text content by removing newlines and carriage returns.  
   - JavaScript code:  
     ```javascript
     const item = $input.item;
     return { json: { content: item.json.body.content.replaceAll("\n", "").replaceAll("\r", ""), title: item.json.body.title } };
     ```  
   - Input: JSON from Webhook node.

3. **Set up secret and parameter initialization**  
   - Although the original Set node is disabled, create a `Set` node to define the following fields:  
     - `x-api-key`: Your HeyGen API key (string, secure storage recommended)  
     - `voice_id`: HeyGen voice ID to use (string)  
     - `avatar_id`: HeyGen avatar ID to use (string)  
     - `title`: `={{ $json.title }}` (from previous node)  
     - `content`: `={{ $json.content }}` (sanitized text)  
   - Connect the Code node to this Set node.  

4. **Add HTTP Request node to create HeyGen video job**  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Method: POST  
   - Headers: `X-Api-Key` set from `x-api-key` field  
   - Body (JSON):  
     ```json
     {
       "video_inputs": [
         {
           "character": {
             "type": "avatar",
             "avatar_id": "{{ $json.avatar_id }}",
             "avatar_style": "normal"
           },
           "voice": {
             "type": "text",
             "input_text": "{{ $json.content }}",
             "voice_id": "{{ $json.voice_id }}",
             "speed": 1.1
           }
         }
       ],
       "dimension": {
         "width": 1280,
         "height": 720
       }
     }
     ```  
   - Connect Set node to this HTTP Request node.

5. **Add a Wait node for polling delay**  
   - Set to 0.1 minutes (6 seconds) or adjust as needed.  
   - Connect HTTP Request node to this Wait node.

6. **Add HTTP Request node to get video status**  
   - URL: `https://api.heygen.com/v1/video_status.get`  
   - Method: GET  
   - Query Parameters: `video_id` from the previous creation node's response JSON path `data.video_id`  
   - Header: `X-Api-Key` same as before  
   - Connect Wait node to this HTTP Request node.

7. **Add an If node to check processing status**  
   - Condition: Check if `{{$json.data.status}}` is NOT equal to `"processing"`  
   - True branch: Video ready  
   - False branch: Video still processing

8. **On False branch, loop back to Wait node**  
   - Connect False output of If node back to Wait node for continuous polling.

9. **On True branch, add HTTP Request node to download video**  
   - URL: `{{$json.data.video_url}}` (from status API response)  
   - Method: GET  
   - This node downloads the video binary data.

10. **Add YouTube node to upload video**  
    - Resource: Video  
    - Operation: Upload  
    - Title: Use `title` parameter from Set node  
    - CategoryId: 28 (Education)  
    - RegionCode: US  
    - Credentials: Configure YouTube OAuth2 credentials with proper scopes  
    - Connect Download Video node to YouTube upload node.

11. **Add Sticky Notes at appropriate points** for documentation and clarity as per original workflow, e.g.:  
    - Describe webhook contract and security  
    - Explain text normalization  
    - Document parameter staging  
    - Outline HeyGen job creation  
    - Describe polling strategy  
    - Clarify download and upload steps

12. **Test the entire workflow** by sending POST requests with JSON body containing `title` and `content` fields, ensuring the video is generated and uploaded correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| HeyGen API documentation for video generation and status retrieval: https://docs.heygen.com/api-reference    | Refer to for detailed API usage and parameters                                                  |
| YouTube API upload quotas and OAuth2 setup details: https://developers.google.com/youtube/v3/guides/uploading-videos | For credential setup and quota considerations                                                    |
| Be mindful of rate limits on HeyGen API; adjust polling delay accordingly                                     | Recommended to avoid hitting rate limits and incurring throttling                               |
| Use secure credential storage in n8n for API keys and OAuth2 tokens                                          | Protect sensitive data from exposure                                                            |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.