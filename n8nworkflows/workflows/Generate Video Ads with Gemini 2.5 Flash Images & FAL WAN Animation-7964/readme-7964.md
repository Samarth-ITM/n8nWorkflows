Generate Video Ads with Gemini 2.5 Flash Images & FAL WAN Animation

https://n8nworkflows.xyz/workflows/generate-video-ads-with-gemini-2-5-flash-images---fal-wan-animation-7964


# Generate Video Ads with Gemini 2.5 Flash Images & FAL WAN Animation

### 1. Workflow Overview

This workflow automates the process of generating premium vertical video advertisements from a single product image and description. It leverages AI-powered image editing and animation services to create a 4-frame visual storyboard with subtle edits and animations, composes these animations into a single video, adds a fitting ambient soundtrack, and finally uploads the completed video to social platforms.

Logical blocks in the workflow:

- **1.1 Input Reception and Initialization**: Receives product image and description via a web form, sets API variables.
- **1.2 Storyboard Generation**: Uses Google Gemini 2.5 chat model to generate a detailed storyboard plan with image edit prompts and animation instructions.
- **1.3 Image Editing with Gemini 2.5 Flash**: Creates 4 subtly edited images based on storyboard prompts using Gemini 2.5 Flash image editing API.
- **1.4 Image Upload and Preparation**: Uploads edited images to imgbb for accessible URLs.
- **1.5 Image-to-Video Animation with FAL WAN**: Converts each edited image into a short animated video clip with FAL WAN image-to-video AI model.
- **1.6 Animation Monitoring and Polling**: Periodically checks animation job status until completion.
- **1.7 Video Composition and Finalization**: Sequences the four video clips into one final video using FFMPEG API, waits for processing, and downloads the final video.
- **1.8 Sound Generation and Integration**: Generates ambient sounds matching the storyboard environment and adds them to the video.
- **1.9 Upload to Social Platforms**: Uploads the final video to TikTok, Instagram Reels, and YouTube with appropriate metadata.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview**: Starts the workflow by receiving user input (photo and product description) through a form, then sets key API variables needed downstream.
- **Nodes Involved**: Photo Upload Form, Set APIs Vars, Merge Vars + Photo1
- **Node Details**:

  - **Photo Upload Form**  
    Type: Form Trigger  
    Role: Entry point, collects a single photo file and product description text from the user.  
    Config: Path `generate-ad`, form fields for one photo (required) and text description.  
    Inputs: External webhook trigger  
    Outputs: JSON with binary photo and description text  
    Edge cases: Missing photo or description will prevent workflow proceed.

  - **Set APIs Vars**  
    Type: Set  
    Role: Defines constants and credentials such as number of images (1), image size (1024x1024), OpenAI model name, image format, imgbb API key placeholder.  
    Config: Hardcoded values, imgbb_api_key must be replaced with real token.  
    Inputs: JSON from previous node  
    Outputs: JSON with API configuration variables  
    Edge cases: Missing or invalid API keys will break subsequent requests.

  - **Merge Vars + Photo1**  
    Type: Merge (combine mode)  
    Role: Combines API variables and uploaded photo data into one JSON object for downstream use.  
    Inputs: From Set APIs Vars and Photo Upload Form  
    Outputs: Combined JSON with photo binary and API vars  
    Edge cases: Input mismatch can cause merge errors.

---

#### 1.2 Storyboard Generation

- **Overview**: Uses Google Gemini 2.5 language model to generate a structured 4-frame storyboard JSON including visual edit prompts and animation instructions based on product description.
- **Nodes Involved**: Google Gemini Chat Model, Storyboard Agent, Structured Output Parser2, Set Storyboard Vars
- **Node Details**:

  - **Google Gemini Chat Model**  
    Type: AI Chat Language Model (Google Gemini 2.5 Pro)  
    Role: Processes prompt with product description and storytelling rules to generate storyboard JSON.  
    Config: Uses Google PALM API credentials.  
    Inputs: Product description text from form  
    Outputs: Raw AI response JSON string  
    Edge cases: API quota limits, malformed responses.

  - **Storyboard Agent**  
    Type: Langchain Agent  
    Role: Enforces prompt rules, parses and structures AI output into a compact JSON for further use.  
    Config: Strict JSON output with keys like title, prompt1-4, i2v_prompt1-4, environment, sound.  
    Inputs: AI model output  
    Outputs: Parsed JSON  
    Edge cases: Parsing errors if AI returns invalid JSON.

  - **Structured Output Parser2**  
    Type: Langchain Structured Output Parser  
    Role: Validates and extracts structured data from AI output example schema.  
    Inputs: AI output from Storyboard Agent  
    Outputs: Validated JSON with storyboard details  
    Edge cases: Schema mismatch or missing fields.

  - **Set Storyboard Vars**  
    Type: Set  
    Role: Maps extracted storyboard JSON fields into workflow variables for image generation and animation prompts.  
    Inputs: Parsed storyboard JSON  
    Outputs: Variables for prompts, titles, environment, sounds  
    Edge cases: Missing keys cause downstream failures.

---

#### 1.3 Image Editing with Gemini 2.5 Flash

- **Overview**: For each of the 4 storyboard prompts, calls Gemini 2.5 Flash Image Edit API to generate subtly edited images from the original uploaded photo.
- **Nodes Involved**: Gemini 2.5 Flash - Generate Image 2, 3, 4, 5; Upload Original Image to imgbb; Separate Image Outputs 2-5; Rename to photo 2-5; Upload Image to imgbb 2-5
- **Node Details**:

  - **Upload Original Image to imgbb**  
    Type: HTTP Request  
    Role: Uploads the original uploaded photo to imgbb to get a public URL for Gemini API calls.  
    Config: Multipart form-data, uses imgbb_api_key from vars  
    Inputs: Binary photo from merge node  
    Outputs: JSON with public image URL  
    Edge cases: Upload failures, invalid API key.

  - **Gemini 2.5 Flash - Generate Image (2 to 5)**  
    Type: HTTP Request (POST)  
    Role: Calls Gemini API endpoint to generate edited images based on prompts prompt1 to prompt4 respectively.  
    Config: JSON body includes prompt from Storyboard Vars, image_url from imgbb upload, num_images=1  
    Inputs: Output URL from imgbb and prompt variables  
    Outputs: JSON with edited images array  
    Edge cases: API request failures, rate limits, invalid prompt.

  - **Separate Image Outputs 2-5**  
    Type: Split Out  
    Role: Extracts individual images from Gemini API response arrays for further processing.  
    Inputs: Gemini API JSON response  
    Outputs: Single image JSON object per item  
    Edge cases: Empty or malformed response arrays.

  - **Rename to photo 2-5**  
    Type: Code (JavaScript)  
    Role: Reformats the split out item to contain URL and binary data under keys photo2 to photo5.  
    Inputs: Single image item  
    Outputs: JSON and binary data with renamed keys  
    Edge cases: Missing binary data or URL.

  - **Upload Image to imgbb 2-5**  
    Type: HTTP Request (multipart form-data)  
    Role: Uploads each edited image binary to imgbb to retrieve stable public URLs for animation.  
    Config: Uses imgbb_api_key, image binary inputDataFieldName corresponds to photo2-5  
    Inputs: Renamed photo binary data  
    Outputs: JSON with uploaded image URLs  
    Edge cases: Upload failures, API limits.

---

#### 1.4 Image-to-Video Animation with FAL WAN

- **Overview**: Converts each edited image into a short video clip with subtle animations using FAL WAN image-to-video AI API.
- **Nodes Involved**: FAL WAN i2v (Queue) 2-5, Wait i2v 2-5, FAL WAN i2v (status) 2-5, Animation Completed? 2-5, FAL WAN i2v (Result) 2-5, Merge1, Aggregate
- **Node Details**:

  - **FAL WAN i2v (Queue) 2-5**  
    Type: HTTP Request (POST)  
    Role: Sends job request to FAL WAN API for each image URL with corresponding animation prompt from storyboard vars.  
    Config: JSON body includes image_url, prompt, num_frames=96, fps=24, resolution=480p  
    Credentials: fal.ai victor header auth  
    Inputs: Aggregated image URLs from imgbb and i2v_prompt variables  
    Outputs: JSON with request_id for animation job  
    Edge cases: API errors, invalid URLs, auth failures.

  - **Wait i2v 2-5**  
    Type: Wait  
    Role: Waits 30 seconds before polling animation job status.  
    Inputs: Follow-up from queue requests  
    Outputs: Trigger to status checking  
    Edge cases: Insufficient wait time may cause premature polling.

  - **FAL WAN i2v (status) 2-5**  
    Type: HTTP Request (GET)  
    Role: Polls animation job status endpoint using request_id from queue response.  
    Credentials: fal.ai victor header auth  
    Outputs: JSON with current status field (e.g., PENDING, COMPLETED)  
    Edge cases: API timeout, missing request_id.

  - **Animation Completed? 2-5**  
    Type: If  
    Role: Checks if animation status equals "COMPLETED".  
    Outputs:  
      - If yes: Requests animation result  
      - If no: Re-queues the image-to-video job to retry animation  
    Edge cases: Infinite retry loops if job never completes.

  - **FAL WAN i2v (Result) 2-5**  
    Type: HTTP Request (GET)  
    Role: Retrieves the completed animation video URL once status is COMPLETED.  
    Credentials: fal.ai victor header auth  
    Outputs: JSON with video URL  
    Edge cases: Missing or delayed result data.

  - **Merge1**  
    Type: Merge (numberInputs=4)  
    Role: Merges the 4 final animation video results into a single item array for sequencing.  
    Inputs: From each animation result node  
    Outputs: Array of video URLs  
    Edge cases: Missing any video breaks downstream.

  - **Aggregate**  
    Type: Aggregate All Item Data  
    Role: Aggregates multiple video results into one JSON array for final processing.  
    Inputs: From Merge1 node  
    Outputs: Single JSON item with all video data  
    Edge cases: Empty input.

---

#### 1.5 Video Composition and Finalization

- **Overview**: Composes the four animated video clips into a single sequential video, waits for processing, downloads the final video.
- **Nodes Involved**: Sequence Video (FFmpeg), Wait Final Video, Get Final Video, Download Final Video
- **Node Details**:

  - **Sequence Video (FFmpeg)**  
    Type: HTTP Request (POST)  
    Role: Calls FFMPEG API to compose video tracks sequentially with 5s duration each.  
    Config: JSON body referencing video URLs from animation results and timing keyframes.  
    Credentials: fal.ai victor header auth  
    Inputs: Aggregated video URLs  
    Outputs: JSON with response_url for composed video  
    Edge cases: API failures, invalid video URLs.

  - **Wait Final Video**  
    Type: Wait  
    Role: Waits 200 seconds to allow video composition to complete.  
    Edge cases: Too short wait causes premature requests.

  - **Get Final Video**  
    Type: HTTP Request (GET)  
    Role: Fetches the final composed video from response_url.  
    Credentials: fal.ai victor header auth  
    Outputs: JSON with final video URL  
    Edge cases: Missing or delayed video.

  - **Download Final Video**  
    Type: HTTP Request (GET)  
    Role: Downloads the final video content by requesting final video URL.  
    Outputs: Binary video data for upload  
    Edge cases: Network errors, large file handling.

---

#### 1.6 Sound Generation and Integration

- **Overview**: Generates ambient audio track based on storyboard environment description and attaches soundtrack to final video.
- **Nodes Involved**: Create Sounds, Wait for Sounds, Get Sounds
- **Node Details**:

  - **Create Sounds**  
    Type: HTTP Request (POST)  
    Role: Calls mmaudio-v2 API with prompt combining ambient description and storyboard sound text, duration 20s, associated to final video URL.  
    Credentials: fal.ai victor header auth  
    Inputs: Final video URL, storyboard sound prompt  
    Outputs: JSON with request_id for audio generation  
    Edge cases: API limits, invalid prompt.

  - **Wait for Sounds**  
    Type: Wait  
    Role: Waits 200 seconds before polling audio generation status.  
    Edge cases: Too short wait causes premature polling.

  - **Get Sounds**  
    Type: HTTP Request (GET)  
    Role: Polls mmaudio-v2 API for generated audio result using request_id.  
    Credentials: fal.ai victor header auth  
    Outputs: JSON with audio file URL or data  
    Edge cases: Missing or delayed audio result.

---

#### 1.7 Upload to Social Platforms

- **Overview**: Uploads the final video with ambient sound to TikTok, Instagram Reels, and YouTube platforms.
- **Nodes Involved**: Upload Post
- **Node Details**:

  - **Upload Post**  
    Type: UploadPost (custom n8n node)  
    Role: Uploads video binary data to configured social media accounts with title metadata.  
    Platforms: TikTok, Instagram (Reels), YouTube  
    Credentials: UploadPost API user credentials  
    Inputs: Binary video data from Download Final Video  
    Outputs: Confirmation of upload success  
    Edge cases: Upload failures, platform API limits, credential expiry.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                         | Input Node(s)                              | Output Node(s)                             | Sticky Note                                    |
|--------------------------------|----------------------------------|---------------------------------------|-------------------------------------------|--------------------------------------------|-----------------------------------------------|
| Photo Upload Form              | Form Trigger                     | Input reception: photo + description  | External webhook                          | Merge Vars + Photo1, Storyboard Agent       | Upload photo and description                   |
| Set APIs Vars                 | Set                             | Define API keys and config vars       | Photo Upload Form                        | Merge Vars + Photo1                         |                                               |
| Merge Vars + Photo1           | Merge (combine)                 | Combine photo + API vars               | Photo Upload Form, Set APIs Vars          | Upload Original Image to imgbb, Storyboard Agent |                                               |
| Google Gemini Chat Model      | AI Chat Model (Google Gemini 2.5 Pro) | Generate storyboard JSON from description | Photo Upload Form (description)           | Storyboard Agent                           |                                               |
| Storyboard Agent             | Langchain Agent                 | Parse and structure storyboard JSON   | Google Gemini Chat Model                   | Structured Output Parser2                   |                                               |
| Structured Output Parser2     | Langchain Structured Output Parser | Validate and extract storyboard JSON  | Storyboard Agent                          | Set Storyboard Vars                        |                                               |
| Set Storyboard Vars          | Set                             | Assign storyboard vars for prompts     | Structured Output Parser2                  | Upload Original Image to imgbb              |                                               |
| Upload Original Image to imgbb | HTTP Request                   | Upload original photo to imgbb         | Merge Vars + Photo1                       | Gemini 2.5 Flash - Generate Image (2-5)    |                                               |
| Gemini 2.5 Flash - Generate Image 2 | HTTP Request                   | Generate edited image frame 1          | Upload Original Image to imgbb, Set Storyboard Vars | Separate Image Outputs 2                   |                                               |
| Gemini 2.5 Flash - Generate Image 3 | HTTP Request                   | Generate edited image frame 2          | Upload Original Image to imgbb, Set Storyboard Vars | Separate Image Outputs 3                   |                                               |
| Gemini 2.5 Flash - Generate Image 4 | HTTP Request                   | Generate edited image frame 3          | Upload Original Image to imgbb, Set Storyboard Vars | Separate Image Outputs 4                   |                                               |
| Gemini 2.5 Flash - Generate Image 5 | HTTP Request                   | Generate edited image frame 4          | Upload Original Image to imgbb, Set Storyboard Vars | Separate Image Outputs 5                   |                                               |
| Separate Image Outputs 2-5    | Split Out                      | Extract single images from Gemini response | Gemini 2.5 Flash - Generate Image 2-5     | Rename to photo 2-5                        |                                               |
| Rename to photo 2-5           | Code                           | Format images with URL and binary keys | Separate Image Outputs 2-5                 | Upload Image to imgbb 2-5                  |                                               |
| Upload Image to imgbb 2-5     | HTTP Request                   | Upload edited images to imgbb          | Rename to photo 2-5                       | Aggregate                                  |                                               |
| Aggregate                    | Aggregate                      | Aggregate uploaded image data          | Upload Image to imgbb 2-5                  | FAL WAN i2v (Queue) 2-5                    |                                               |
| FAL WAN i2v (Queue) 2-5       | HTTP Request                   | Queue image-to-video animation jobs    | Aggregate                                | Wait i2v 2-5                              |                                               |
| Wait i2v 2-5                 | Wait                           | Wait before polling animation status   | FAL WAN i2v (Queue) 2-5                    | FAL WAN i2v (status) 2-5                   |                                               |
| FAL WAN i2v (status) 2-5      | HTTP Request                   | Poll animation job status               | Wait i2v 2-5                             | Animation Completed? 2-5                    |                                               |
| Animation Completed? 2-5       | If                             | Check if animation finished             | FAL WAN i2v (status) 2-5                   | FAL WAN i2v (Result) 2-5 or re-queue      |                                               |
| FAL WAN i2v (Result) 2-5      | HTTP Request                   | Get completed animation video URLs     | Animation Completed? 2-5 (true branch)    | Merge1                                     |                                               |
| Merge1                      | Merge (numberInputs=4)          | Merge 4 animation video results         | FAL WAN i2v (Result) 2-5                   | Aggregate1                                  |                                               |
| Aggregate1                   | Aggregate                      | Aggregate merged animation videos       | Merge1                                  | Sequence Video (FFmpeg)                     |                                               |
| Sequence Video (FFmpeg)       | HTTP Request                   | Compose final sequential video          | Aggregate1                               | Wait Final Video                           |                                               |
| Wait Final Video             | Wait                           | Wait for video composition to finish   | Sequence Video (FFmpeg)                    | Get Final Video                            |                                               |
| Get Final Video              | HTTP Request                   | Download final composed video URL       | Wait Final Video                         | Create Sounds                             |                                               |
| Create Sounds               | HTTP Request                   | Generate ambient audio for video        | Get Final Video                          | Wait for Sounds                           |                                               |
| Wait for Sounds             | Wait                           | Wait for audio generation completion    | Create Sounds                           | Get Sounds                                |                                               |
| Get Sounds                 | HTTP Request                   | Retrieve generated audio file            | Wait for Sounds                         | Download Final Video                      |                                               |
| Download Final Video        | HTTP Request                   | Download video binary content            | Get Sounds                              | Upload Post                              |                                               |
| Upload Post                | UploadPost (custom)             | Upload final video to social platforms | Download Final Video                    | None                                      | Platforms: TikTok, Instagram Reels, YouTube   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Photo Upload Form"**  
   - Type: Form Trigger  
   - Path: `generate-ad`  
   - Form Fields:  
     - File field "photo" (required, single file)  
     - Text area "Product description" (required)  
   - Output: Binary photo + JSON description  

2. **Create Set Node: "Set APIs Vars"**  
   - Assign variables:  
     - `number_of_images`: 1  
     - `size_of_image`: "1024x1024"  
     - `openai_image_model`: "gemini-25-flash-image"  
     - `format_image`: "webp"  
     - `imgbb_api_key`: *insert your imgbb API key here*  

3. **Create Merge Node: "Merge Vars + Photo1"**  
   - Mode: Combine by position  
   - Inputs:  
     - From "Photo Upload Form"  
     - From "Set APIs Vars"  

4. **Create HTTP Request Node: "Upload Original Image to imgbb"**  
   - Method: POST  
   - URL: `https://api.imgbb.com/1/upload`  
   - Body: multipart/form-data, with fields:  
     - `image`: binary data from photo  
     - `key`: from `imgbb_api_key` variable  
   - Retry on fail enabled  

5. **Create AI Chat Model Node: "Google Gemini Chat Model"**  
   - Model: Gemini 2.5 Pro  
   - Credentials: Google Palm API  
   - Input: Product description text from form  

6. **Create Langchain Agent Node: "Storyboard Agent"**  
   - Prompt: Provide product description as context  
   - Output: Compact JSON with 4 visual edit prompts, 4 animation prompts, environment and sound description  
   - Use system message to enforce storytelling rules  

7. **Create Langchain Structured Output Parser: "Structured Output Parser2"**  
   - Use example JSON schema matching storyboard structure  
   - Input: Output from "Storyboard Agent"  

8. **Create Set Node: "Set Storyboard Vars"**  
   - Map fields from structured output to variables:  
     - title, prompt1-4, i2v_prompt1-4, environment, sound  

9. **Create four HTTP Request Nodes: "Gemini 2.5 Flash - Generate Image 2-5"**  
   - Method: POST  
   - URL: `https://fal.run/fal-ai/gemini-25-flash-image/edit`  
   - Body JSON:  
     - `prompt`: respective prompt1 to prompt4 from storyboard vars  
     - `image_urls`: array with original image URL from imgbb upload  
     - `num_images`: from vars (1)  
   - Use fal.ai victor HTTP header auth credentials  

10. **Create four Split Out Nodes: "Separate Image Outputs 2-5"**  
    - Field to split out: "images"  
    - Input: Corresponding Gemini 2.5 Flash generate image node  

11. **Create four Code Nodes: "Rename to photo 2-5"**  
    - JS code to map each item to JSON with URL and binary under keys photo2..photo5  

12. **Create four HTTP Request Nodes: "Upload Image to imgbb 2-5"**  
    - Method: POST  
    - URL: `https://api.imgbb.com/1/upload`  
    - Body: multipart/form-data with image binary and imgbb_api_key  

13. **Create Aggregate Node: "Aggregate"**  
    - Aggregate all uploaded image data from Upload Image to imgbb 2-5 nodes  

14. **Create four HTTP Request Nodes: "FAL WAN i2v (Queue) 2-5"**  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/wan/v2.2-a14b/image-to-video`  
    - Body JSON:  
      - `image_url`: corresponding uploaded image URL from aggregate  
      - `prompt`: respective i2v_prompt1-4 from storyboard vars  
      - `num_frames`: 96, `frames_per_second`: 24, `resolution`: 480p  
    - Use fal.ai victor HTTP header auth credentials  

15. **Create four Wait Nodes: "Wait i2v 2-5"**  
    - Duration: 30 seconds  
    - Used to pace polling  

16. **Create four HTTP Request Nodes: "FAL WAN i2v (status) 2-5"**  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/wan/requests/{{ $json.request_id }}/status`  
    - Auth: fal.ai victor header auth  

17. **Create four If Nodes: "Animation Completed? 2-5"**  
    - Condition: `$json.status == "COMPLETED"`  
    - If true: proceed to result fetch  
    - If false: re-queue animation (retry)  

18. **Create four HTTP Request Nodes: "FAL WAN i2v (Result) 2-5"**  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/wan/requests/{{ $json.request_id }}`  
    - Auth: fal.ai victor header auth  

19. **Create Merge Node: "Merge1"**  
    - Number inputs: 4  
    - Inputs: Outputs of FAL WAN i2v Result nodes  

20. **Create Aggregate Node: "Aggregate1"**  
    - Aggregate all merged video results  

21. **Create HTTP Request Node: "Sequence Video (FFmpeg)"**  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/compose`  
    - Body JSON: Compose tracks sequentially with 4 video clips, each 5 seconds  
    - Auth: fal.ai victor header auth  

22. **Create Wait Node: "Wait Final Video"**  
    - Duration: 200 seconds  

23. **Create HTTP Request Node: "Get Final Video"**  
    - Method: GET  
    - URL: From compose response URL  
    - Auth: fal.ai victor header auth  

24. **Create HTTP Request Node: "Create Sounds"**  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/mmaudio-v2`  
    - Body JSON: Prompt combining storyboard sound description, duration 20s, and final video URL  
    - Auth: fal.ai victor header auth  

25. **Create Wait Node: "Wait for Sounds"**  
    - Duration: 200 seconds  

26. **Create HTTP Request Node: "Get Sounds"**  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/mmaudio-v2/requests/{{ $json.request_id }}`  

27. **Create HTTP Request Node: "Download Final Video"**  
    - Method: GET  
    - URL: Final video URL with sound included  

28. **Create UploadPost Node: "Upload Post"**  
    - Platforms: TikTok, Instagram Reels, YouTube  
    - Credentials: UploadPost API user  
    - Input: Binary video data  

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                          |
|--------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| The workflow requires active API keys for fal.ai (victor), imgbb, Google Palm API, and UploadPost integrations.   | Credentials setup in n8n                 |
| This workflow depends heavily on external AI services and may face latency due to wait and polling intervals.      | Timeouts and retry logic implemented     |
| For best results, provide high-quality product images and detailed descriptions to guide Gemini storyboard creation. | Input quality affects output quality     |
| The storyboard agent enforces strict JSON output for easy parsing and integration.                                 | Storyboard JSON schema is embedded in Structured Output Parser2 |
| The workflow outputs vertical 9:16 videos optimized for social media platforms like TikTok and Instagram Reels.   | Video resolution fixed at 480p           |
| UploadPost node supports multiple platforms and requires correct user credentials per platform.                    | See node config for platform-specific params |
| For troubleshooting API errors, check API key permissions, request limits, and network connectivity.               | Common error sources                     |

---

**Disclaimer:**  
The provided content is exclusively generated from an automated workflow built with n8n, respecting all applicable content policies and handling only lawful, public data. It contains no illegal or offensive material.