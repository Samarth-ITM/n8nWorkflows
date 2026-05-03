Turn Text into Cinematic Videos using GPT-4, Dumpling FLUX.1 Pro, and Veo 3

https://n8nworkflows.xyz/workflows/turn-text-into-cinematic-videos-using-gpt-4--dumpling-flux-1-pro--and-veo-3-8564


# Turn Text into Cinematic Videos using GPT-4, Dumpling FLUX.1 Pro, and Veo 3

### 1. Workflow Overview

This workflow automates the transformation of short text ideas into cinematic-style videos by leveraging advanced AI models and APIs. It is designed for content creators, marketers, and multimedia artists aiming to produce visually rich videos from simple textual prompts without manual video editing.

The workflow is structured into three main logical blocks:

- **1.1 Input Reception & Prompt Expansion**: Captures a user-submitted short image idea and expands it into a detailed, vivid image prompt using GPT-4.1.

- **1.2 Image Generation & Video Prompt Creation**: Generates a high-quality cinematic image from the expanded prompt via Dumpling FLUX.1 Pro AI, uploads the image to Veo 3’s API, and uses GPT-4o to create a matching short video prompt describing scene, action, dialogue, and style.

- **1.3 Video Generation, Status Checking & Logging**: Requests Veo 3 to generate an 8-second video based on the image and prompt, waits and polls for completion, then logs the results including the original idea, prompts, and video URL into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Prompt Expansion

**Overview:**  
This block begins the workflow by receiving a short textual idea for an image from the user and uses GPT-4.1 to expand it into a detailed, descriptive prompt suited for generating high-quality cinematic images.

**Nodes Involved:**  
- Capture Image Idea (Form Trigger)  
- Generate Image Prompt (GPT-4.1)

**Node Details:**  

- **Capture Image Idea**  
  - Type: Form Trigger  
  - Role: Entry point to receive user input via a web form with a required field "Image Ideas".  
  - Configuration: A form titled "Video Content ideas" with one required field for the image idea text.  
  - Inputs: User web form submission  
  - Outputs: JSON with field `Image Ideas`  
  - Edge Cases: User submits empty input (blocked by required field), webhook availability  
  - Version: 2.2  

- **Generate Image Prompt (GPT-4.1)**  
  - Type: OpenAI GPT-4.1 via Langchain node  
  - Role: Transforms short image idea text into a vivid, cinematic prompt describing environment, style, mood, lighting, perspective, and artistic details.  
  - Configuration: Uses model "gpt-4.1-mini" with a system instruction to output only a refined prompt under 3 sentences, no technical camera jargon unless enhancing the scene.  
  - Expressions: Injects user input via `={{ $json['Image Ideas'] }}` into the prompt.  
  - Inputs: Output from Capture Image Idea  
  - Outputs: JSON containing the generated prompt in `message.content`  
  - Edge Cases: API rate limits, malformed input, model downtime, unexpected output format  
  - Credentials: OpenAI API key  
  - Version: 1.8  

---

#### 2.2 Image Generation & Video Prompt Creation

**Overview:**  
Expands the refined text prompt into a cinematic image via Dumpling AI’s FLUX.1 Pro model, uploads the generated image to Veo 3’s API, and uses GPT-4o to create a detailed video prompt describing the scene, action, dialogue, and style for video generation.

**Nodes Involved:**  
- Generate Image (FLUX.1 Pro) with Dumpling AI (HTTP Request)  
- Generate Video Prompt (GPT-4o)  
- Upload Image to KIE API for Veo 3 (HTTP Request)  
- Clean Video Prompt (Remove \n) (Code)

**Node Details:**  

- **Generate Image (FLUX.1 Pro) with Dumpling AI**  
  - Type: HTTP Request  
  - Role: Sends the cinematic prompt to Dumpling AI API to create a 16:9 aspect ratio JPG image using model "FLUX.1-pro".  
  - Configuration: POST to Dumpling endpoint with JSON body including prompt, aspect ratio, output format.  
  - Expressions: Prompt taken from `{{ $json.message.content }}` (output of GPT-4.1 node)  
  - Inputs: Generated prompt from previous node  
  - Outputs: JSON including image URL(s) in `images[0].url`  
  - Credentials: HTTP Header Auth for Dumpling AI  
  - Edge Cases: API errors, invalid prompt, response latency, image generation failures  
  - Version: 4.2  

- **Generate Video Prompt (GPT-4o)**  
  - Type: OpenAI GPT-4o via Langchain node  
  - Role: Analyzes the generated image and creates a formatted video prompt for Veo 3 describing scene, action, dialogue, camera motion, style, and audio notes.  
  - Configuration: Model "chatgpt-4o-latest", input is the first image URL from Dumpling AI. System prompt enforces strict output format and constraints (one scene, one action, short dialogue under 16 words, simple camera motion).  
  - Expressions: Image URL from `{{ $json.images[0].url }}`  
  - Inputs: Output from Generate Image (FLUX.1 Pro)  
  - Outputs: JSON with content containing formatted video prompt text  
  - Credentials: OpenAI API key  
  - Edge Cases: API timeouts, incorrect format, hallucinated scene elements, missing image URLs  
  - Version: 1.8  

- **Upload Image to KIE API for Veo 3**  
  - Type: HTTP Request  
  - Role: Uploads the generated image URL to Veo 3’s KIE AI file upload API to prepare for video generation.  
  - Configuration: POST with JSON body containing fileUrl and uploadPath "images".  
  - Expressions: Uses image URL from Dumpling AI output  
  - Inputs: Output from Generate Image (FLUX.1 Pro)  
  - Outputs: JSON confirming upload success with a downloadable URL for Veo 3 use  
  - Credentials: HTTP Header Auth for Veo 3 API  
  - Edge Cases: Upload failures, invalid URLs, network errors  
  - Version: 4.2  

- **Clean Video Prompt (Remove \n)**  
  - Type: Code (JavaScript)  
  - Role: Removes newline characters from the video prompt text generated by GPT-4o to ensure it fits API requirements.  
  - Configuration: Iterates over all items from Generate Video Prompt node, replaces `\n` with empty string in `.json.content`.  
  - Inputs: Output from Upload Image node (via connection)  
  - Outputs: Cleaned video prompt text  
  - Edge Cases: Unexpected null or missing content fields  
  - Version: 2  

---

#### 2.3 Video Generation, Status Checking & Logging

**Overview:**  
Submits the cleaned video prompt and uploaded image URL to Veo 3’s video generation API, then waits and polls for completion status. Once the video is ready, logs the original idea, prompts, and final video URL into Google Sheets.

**Nodes Involved:**  
- Generate Video (Veo 3) with KIE API (HTTP Request)  
- Wait for Video Processing (80s) (Wait)  
- Check Video Status (Veo 3) (HTTP Request)  
- Video Ready? (If)  
- Retry Delay (20s) (Wait)  
- Log to Google Sheet (Google Sheets)

**Node Details:**  

- **Generate Video (Veo 3) with KIE API**  
  - Type: HTTP Request  
  - Role: Sends request to Veo 3 API to generate a short 8-second video using the cleaned video prompt and uploaded image URL.  
  - Configuration: POST with JSON body including prompt, imageUrls (array with upload download URL), model "veo3_fast", aspect ratio 16:9, fixed seed 12345, fallback and translation flags.  
  - Expressions: Prompt from cleaned video prompt, image URL from upload node.  
  - Inputs: Output from Clean Video Prompt node  
  - Outputs: JSON containing taskId for video generation job  
  - Credentials: HTTP Header Auth for Veo 3 API  
  - Edge Cases: API failures, invalid prompt/image URL, task initiation errors  
  - Version: 4.2  

- **Wait for Video Processing (80s)**  
  - Type: Wait  
  - Role: Pauses workflow for 80 seconds to allow video processing on Veo 3 backend.  
  - Configuration: Fixed 80 seconds delay  
  - Inputs: Output from Generate Video node  
  - Outputs: Passes data unchanged  
  - Edge Cases: Fixed delay might be insufficient or excessive depending on load  

- **Check Video Status (Veo 3)**  
  - Type: HTTP Request  
  - Role: Queries Veo 3 API to check if the video generation is complete using taskId.  
  - Configuration: GET request with query parameter `taskId` taken from Generate Video node output.  
  - Inputs: Output from Wait node or Retry Delay node  
  - Outputs: JSON with status code and video URL if ready  
  - Credentials: HTTP Header Auth for Veo 3 API  
  - Edge Cases: Network errors, invalid taskId, API throttling, inconsistent status responses  
  - Version: 4.2  

- **Video Ready? (If)**  
  - Type: If Condition  
  - Role: Checks if the video status code equals 200 (success) to determine if video is ready.  
  - Configuration: Condition on `$json.code == 200`  
  - Inputs: Output from Check Video Status node  
  - Outputs: True branch (video ready), False branch (retry)  
  - Edge Cases: Status codes other than 200, missing code field  

- **Retry Delay (20s)**  
  - Type: Wait  
  - Role: Waits 20 seconds before retrying the video status check.  
  - Configuration: Fixed delay 20 seconds  
  - Inputs: False branch of Video Ready? node  
  - Outputs: Loops back to Check Video Status node  
  - Edge Cases: Multiple retries may cause long delays; no max retry count implemented  

- **Log to Google Sheet**  
  - Type: Google Sheets Append  
  - Role: Records the workflow results including the original image idea, image prompt, video prompt, and final video URL into a Google Sheets document.  
  - Configuration: Appends a row with columns: Idea, Prompt USED for Image, Prompt USED for Video, Video Created (video URL).  
  - Expressions: Maps from respective nodes' outputs.  
  - Inputs: True branch of Video Ready? node  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Sheet permission errors, quota limits, malformed data entry  
  - Version: 4.7  

---

### 3. Summary Table

| Node Name                              | Node Type                       | Functional Role                                | Input Node(s)                        | Output Node(s)                          | Sticky Note                                                                                                   |
|--------------------------------------|--------------------------------|-----------------------------------------------|------------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Capture Image Idea                   | Form Trigger                   | Receives user text idea input                  | -                                  | Generate Image Prompt (GPT-4.1)       | Step 1: Expand Text Idea into Visual Prompt with GPT-4.1                                                      |
| Generate Image Prompt (GPT-4.1)      | OpenAI (Langchain)             | Expands short idea into detailed image prompt | Capture Image Idea                 | Generate Image (FLUX.1 Pro)           | Step 1: Expand Text Idea into Visual Prompt with GPT-4.1                                                      |
| Generate Image (FLUX.1 Pro) with Dumpling AI | HTTP Request                  | Generates cinematic image from prompt          | Generate Image Prompt (GPT-4.1)    | Generate Video Prompt (GPT-4o)         | Step 2: Create Image & Generate Video Prompt                                                                   |
| Generate Video Prompt (GPT-4o)       | OpenAI (Langchain)             | Creates formatted video prompt from image      | Generate Image (FLUX.1 Pro)        | Upload Image to KIE API for Veo 3      | Step 2: Create Image & Generate Video Prompt                                                                   |
| Upload Image to KIE API for Veo 3    | HTTP Request                  | Uploads image to Veo 3 for video generation     | Generate Video Prompt (GPT-4o)     | Clean Video Prompt (Remove \n)          | Step 2: Create Image & Generate Video Prompt                                                                   |
| Clean Video Prompt (Remove \n)       | Code                          | Removes newline characters from video prompt   | Upload Image to KIE API for Veo 3  | Generate Video (Veo 3) with KIE API    | Step 2: Create Image & Generate Video Prompt                                                                   |
| Generate Video (Veo 3) with KIE API  | HTTP Request                  | Requests video generation from Veo 3            | Clean Video Prompt (Remove \n)     | Wait for Video Processing (80s)        | Step 3: Generate Video and Log Output                                                                           |
| Wait for Video Processing (80s)      | Wait                          | Waits 80 seconds for video processing           | Generate Video (Veo 3) with KIE API | Check Video Status (Veo 3)              | Step 3: Generate Video and Log Output                                                                           |
| Check Video Status (Veo 3)            | HTTP Request                  | Checks if generated video is ready              | Wait for Video Processing (80s), Retry Delay (20s) | Video Ready?                          | Step 3: Generate Video and Log Output                                                                           |
| Video Ready?                        | If Condition                   | Determines if video is ready (code == 200)      | Check Video Status (Veo 3)          | Log to Google Sheet (True) / Retry Delay (False) | Step 3: Generate Video and Log Output                                                                           |
| Retry Delay (20s)                   | Wait                          | Waits 20 seconds before retrying video status   | Video Ready? (False branch)         | Check Video Status (Veo 3)              | Step 3: Generate Video and Log Output                                                                           |
| Log to Google Sheet                 | Google Sheets Append          | Logs idea, prompts, and video URL to spreadsheet | Video Ready? (True branch)          | -                                     | Step 3: Generate Video and Log Output                                                                           |
| Sticky Note                        | Sticky Note                   | Step 1 explanation                              | -                                  | -                                     | Step 1: Expand Text Idea into Visual Prompt                                                                      |
| Sticky Note1                       | Sticky Note                   | Step 2 explanation                              | -                                  | -                                     | Step 2: Create Image & Generate Video Prompt                                                                     |
| Sticky Note2                       | Sticky Note                   | Step 3 explanation                              | -                                  | -                                     | Step 3: Generate Video and Log Output                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node - "Capture Image Idea"**  
   - Type: Form Trigger  
   - Configure form title: "Video Content ideas"  
   - Add one required text field labeled "Image Ideas"  
   - Enable webhook to accept submissions  

2. **Add OpenAI Node - "Generate Image Prompt (GPT-4.1)"**  
   - Type: OpenAI (Langchain)  
   - Model: "gpt-4.1-mini"  
   - System prompt: Instruct to expand short image ideas into vivid, cinematic prompts (include environment, style, mood, lighting, perspective, artistic details, max 3 sentences, no camera jargon unless enhancing)  
   - User prompt: Inject `={{ $json['Image Ideas'] }}`  
   - Credentials: Link valid OpenAI API key  
   - Connect: Output of "Capture Image Idea" to this node  

3. **Add HTTP Request Node - "Generate Image (FLUX.1 Pro) with Dumpling AI"**  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/generate-ai-image`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "model": "FLUX.1-pro",
       "input": {
         "prompt": "{{ $json.message.content }}",
         "aspect_ratio": "16:9",
         "output_format": "jpg"
       }
     }
     ```  
   - Authentication: HTTP Header Auth with Dumpling AI credentials  
   - Connect: Output of "Generate Image Prompt (GPT-4.1)"  

4. **Add OpenAI Node - "Generate Video Prompt (GPT-4o)"**  
   - Type: OpenAI (Langchain)  
   - Model: "chatgpt-4o-latest"  
   - Text prompt: Provide a detailed instruction to analyze the first image URL and generate a formatted video prompt including scene, action, dialogue, camera motion, style, audio notes (strict format, one scene, one action, dialogue <16 words). Use the image URL: `={{ $json.images[0].url }}`  
   - Credentials: OpenAI API key  
   - Connect: Output of "Generate Image (FLUX.1 Pro) with Dumpling AI"  

5. **Add HTTP Request Node - "Upload Image to KIE API for Veo 3"**  
   - Method: POST  
   - URL: `https://kieai.redpandaai.co/api/file-url-upload`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "fileUrl": "{{ $('Generate Image (FLUX.1 Pro) with Dumpling AI').item.json.images[0].url }}",
       "uploadPath": "images"
     }
     ```  
   - Authentication: HTTP Header Auth with Veo 3 credentials  
   - Connect: Output of "Generate Video Prompt (GPT-4o)"  

6. **Add Code Node - "Clean Video Prompt (Remove \n)"**  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const analyzeImageItems = $("Generate Video Prompt (GPT-4o)").all();
     const cleanedItems = analyzeImageItems.map((item) => {
       item.json.content = item.json.content.replace(/\n/g, "");
       return item;
     });
     return cleanedItems;
     ```  
   - Connect: Output of "Upload Image to KIE API for Veo 3"  

7. **Add HTTP Request Node - "Generate Video (Veo 3) with KIE API"**  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/veo/generate`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "prompt": "{{ $json.content }}",
       "imageUrls": ["{{ $('Upload Image to KIE API for Veo 3').item.json.data.downloadUrl }}"],
       "model": "veo3_fast",
       "aspectRatio": "16:9",
       "seeds": 12345,
       "enableFallback": false,
       "enableTranslation": true
     }
     ```  
   - Authentication: HTTP Header Auth with Veo 3 credentials  
   - Connect: Output of "Clean Video Prompt (Remove \n)"  

8. **Add Wait Node - "Wait for Video Processing (80s)"**  
   - Duration: 80 seconds  
   - Connect: Output of "Generate Video (Veo 3) with KIE API"  

9. **Add HTTP Request Node - "Check Video Status (Veo 3)"**  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/veo/get-1080p-video`  
   - Query Parameter: `taskId` = `={{ $('Generate Video (Veo 3) with KIE API').item.json.data.taskId }}`  
   - Authentication: HTTP Header Auth with Veo 3 credentials  
   - Connect: Output of "Wait for Video Processing (80s)" and also from "Retry Delay (20s)"  

10. **Add If Node - "Video Ready?"**  
    - Condition: Check if `$json.code == 200`  
    - Connect: Output of "Check Video Status (Veo 3)"  
    - True branch: Proceed to "Log to Google Sheet"  
    - False branch: Proceed to "Retry Delay (20s)"  

11. **Add Wait Node - "Retry Delay (20s)"**  
    - Duration: 20 seconds  
    - Connect: False branch of "Video Ready?" node  
    - Output: Loop back to "Check Video Status (Veo 3)"  

12. **Add Google Sheets Node - "Log to Google Sheet"**  
    - Operation: Append  
    - Sheet: Use target Google Sheet ID and sheet name (e.g., gid=0)  
    - Columns:  
      - Idea: `={{ $('Capture Image Idea').item.json['Image Ideas'] }}`  
      - Prompt USED for Image: `={{ $('Generate Image Prompt (GPT-4.1)').item.json.message.content }}`  
      - Prompt USED for Video: `={{ $('Generate Video Prompt (GPT-4o)').item.json.content }}`  
      - Video Created: `={{ $json.data.result_url }}` (from video status)  
    - Credentials: Google Sheets OAuth2  
    - Connect: True branch of "Video Ready?" node  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Step 1: Expanding a brief text idea into a vivid visual prompt is crucial for guiding AI image generation quality.    | Sticky Note associated with "Capture Image Idea" and "Generate Image Prompt (GPT-4.1)" nodes                         |
| Step 2: The video prompt format for Veo 3 requires strict adherence to one scene, one action, short dialogue, and style notes for video realism. | Sticky Note associated with image generation, video prompt nodes, and image upload nodes                             |
| Step 3: The workflow uses polling with retry logic to handle asynchronous video generation on Veo 3, logging final results for audit and manual review. | Sticky Note associated with video generation, waiting, status check, retry, and logging nodes                        |
| Dumpling FLUX.1 Pro API documentation: https://app.dumplingai.com/docs/api                                                | Reference for image generation API                                                                                   |
| Veo 3 KIE AI API documentation: https://kieai.redpandaai.co/docs                                                        | Reference for video generation and file upload APIs                                                                  |
| OpenAI GPT-4.1 and GPT-4o Models: Use appropriate API keys and monitor quotas to avoid rate limiting and interruptions | OpenAI official documentation: https://platform.openai.com/docs/models/gpt-4                                          |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.