Recreate Instagram Reels with Gemini 2.0 Analysis & Minimax Video Generation

https://n8nworkflows.xyz/workflows/recreate-instagram-reels-with-gemini-2-0-analysis---minimax-video-generation-9863


# Recreate Instagram Reels with Gemini 2.0 Analysis & Minimax Video Generation

### 1. Workflow Overview

This workflow automates the process of recreating Instagram Reels using advanced AI analysis and video generation. Starting from a chat-triggered Instagram Reel URL input, it downloads the video, sends it to Google’s Gemini 2.0 AI model for a detailed frame-by-frame analysis, then uses the Minimax Video Generation model (hosted on Replicate) to generate a new video based on that analysis. The workflow includes robust validation, retry logic, and asynchronous polling to handle the video generation time.

**Target use cases:**  
- Content creators who want to generate AI-based variations of Instagram Reels  
- Social media managers automating video content creation pipelines  
- AI enthusiasts exploring video interpretation and generation

**Logical blocks:**  
- **1.1 Input Reception:** Receives the Instagram Reel URL via chat trigger.  
- **1.2 Download Instagram Reel:** Fetches Instagram Reel metadata and extracts the direct video URL.  
- **1.3 Validation & Download Video:** Validates the URL and downloads the video as binary data.  
- **1.4 Upload to AI & Video Analysis:** Uploads the video to Google AI and analyzes it using Gemini 2.0.  
- **1.5 AI Video Generation:** Sends the detailed analysis as a prompt to Minimax video generation model on Replicate.  
- **1.6 Polling for Completion:** Waits and periodically checks the status until video generation completes.  
- **1.7 Fetch Final Video:** Downloads the AI-generated video output for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Waits for a user to send an Instagram Reel URL via chat message to trigger the workflow.

**Nodes Involved:**  
- When chat message received

**Node Details:**  
- **When chat message received**  
  - *Type:* Chat Trigger (LangChain)  
  - *Role:* Entry point listening for chat messages containing Instagram Reel URLs.  
  - *Configuration:* Public webhook, no special options. Captures user input in `{{ $json.chatInput }}`.  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Passes input to "Reel Downloader" node.  
  - *Failure cases:* Network issues, webhook misconfiguration, malformed input (non-URL).  
  - *Notes:* Webhook ID auto-generated, no manual config needed.

---

#### 2.2 Download Instagram Reel

**Overview:**  
Queries RapidAPI’s Instagram Reels Downloader API to fetch metadata and obtain the direct video URL.

**Nodes Involved:**  
- Reel Downloader  
- Formatter  
- Checks Null/ Not

**Node Details:**  
- **Reel Downloader**  
  - *Type:* HTTP Request  
  - *Role:* Calls RapidAPI to get Instagram Reel metadata and video URL.  
  - *Configuration:*  
    - URL: `https://instagram-reels-downloader-api.p.rapidapi.com/download`  
    - Query param: `url={{ $json.chatInput }}` (user input URL)  
    - Headers: `x-rapidapi-host` fixed, `x-rapidapi-key` must be replaced with valid RapidAPI key.  
  - *Inputs:* From chat trigger  
  - *Outputs:* JSON containing `data.medias[0].url` with video URL.  
  - *Failures:* API key missing/invalid, rate limits, network errors, invalid input URL.  
- **Formatter**  
  - *Type:* Set  
  - *Role:* Extracts the video URL from API response and assigns it to variable `Instagram Reel`.  
  - *Configuration:* Assigns `Instagram Reel` = `{{ $json.data.medias[0].url }}`  
  - *Inputs:* Reel Downloader output  
  - *Outputs:* JSON with clean variable for next node  
- **Checks Null/ Not**  
  - *Type:* If  
  - *Role:* Validates if the video URL is null or not to decide flow.  
  - *Configuration:* Checks if `{{ $json['Instagram Reel'] }}` equals `"null"` string.  
  - *Inputs:* Formatter output  
  - *Outputs:*  
    - True branch: Loop back to Reel Downloader (retry)  
    - False branch: Continue to Download Reel node  
  - *Failures:* Logic errors if null check fails, infinite loop risk if API persistently returns null.

---

#### 2.3 Validation & Download Video

**Overview:**  
Downloads the Instagram Reel video file using the validated URL as binary data for further processing.

**Nodes Involved:**  
- Download Reel

**Node Details:**  
- **Download Reel**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the actual video file (binary data).  
  - *Configuration:*  
    - URL: `{{ $json["Instagram Reel"] }}` (clean video URL)  
    - Method: GET  
    - No special headers or auth required.  
  - *Inputs:* Output from Checks Null/ Not (false branch)  
  - *Outputs:* Binary video data (MP4)  
  - *Failures:* Network issues, broken URLs, large file download errors.

---

#### 2.4 Upload to AI & Video Analysis

**Overview:**  
Uploads the binary video file to Google’s Gemini AI platform and requests detailed frame-by-frame video analysis.

**Nodes Involved:**  
- Upload the video to model  
- Analyse the Video

**Node Details:**  
- **Upload the video to model**  
  - *Type:* HTTP Request  
  - *Role:* Uploads video as binary data to Google AI Storage API.  
  - *Configuration:*  
    - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files?key=YOUR_GOOGLE_API_KEY_HERE` (replace API key)  
    - Method: POST  
    - Headers: Google-specific upload commands (start, upload, finalize), content-type `video/mp4`  
    - Body: Binary data from previous node  
  - *Inputs:* Binary video data from Download Reel  
  - *Outputs:* JSON with `file.uri` for uploaded video  
  - *Failures:* API key invalid, upload errors, network timeouts.  
- **Analyse the Video**  
  - *Type:* HTTP Request  
  - *Role:* Sends uploaded video URI to Gemini 2.0 Flash model for detailed video description.  
  - *Configuration:*  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=YOUR_GOOGLE_API_KEY_HERE` (replace API key)  
    - Method: POST  
    - Body: JSON with `file_data` referencing uploaded video URI and detailed instructions for Gemini to analyze every frame visually and audibly with strict formatting rules.  
    - Headers: `Content-Type: application/js` (likely a typo, intended `application/json`)  
  - *Inputs:* Output from upload node (including `file.uri`)  
  - *Outputs:* JSON containing detailed textual description of video in continuous lowercase text, no punctuation.  
  - *Failures:* API key issues, malformed request, model unavailability, response parse errors.

---

#### 2.5 AI Video Generation

**Overview:**  
Uses the textual video analysis as a prompt to generate a new video using Replicate’s Minimax Video-01 model asynchronously.

**Nodes Involved:**  
- Create the Video

**Node Details:**  
- **Create the Video**  
  - *Type:* HTTP Request  
  - *Role:* Sends Gemini’s detailed text prompt to Minimax video generation API.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/models/minimax/video-01/predictions`  
    - Method: POST  
    - Body JSON: `{ "input": { "prompt": "{{ $json.candidates[0].content.parts[0].text }}" } }` (uses Gemini output)  
    - Headers: Authorization Bearer token (`YOUR_REPLICATE_API_KEY_HERE`), `Prefer: wait` to wait for prediction  
  - *Inputs:* Gemini analysis output  
  - *Outputs:* JSON with prediction ID for video generation process  
  - *Failures:* Auth failures, API rate limiting, malformed prompt, network errors.

---

#### 2.6 Polling for Completion

**Overview:**  
Waits a fixed duration and polls Replicate API for video generation status until the video is ready.

**Nodes Involved:**  
- Pause  
- Get Status of the Video  
- Checks if the video generation is complete  
- Wait (loop node)

**Node Details:**  
- **Pause**  
  - *Type:* Wait  
  - *Role:* Pauses workflow execution for 2 minutes to allow video generation processing.  
  - *Configuration:* `amount` = 2 (minutes)  
  - *Inputs:* Output from Create the Video node  
  - *Outputs:* To Get Status of the Video node  
- **Get Status of the Video**  
  - *Type:* HTTP Request  
  - *Role:* Checks current status of video generation using prediction ID.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions/{{ $json["id"] }}`  
    - Method: GET  
    - Headers: Authorization Bearer token (`YOUR_REPLICATE_API_KEY_HERE`)  
  - *Inputs:* From Pause or Wait node  
  - *Outputs:* JSON with `status` field (`succeeded`, `processing`, or `failed`)  
- **Checks if the video generation is complete**  
  - *Type:* If  
  - *Role:* Branches workflow based on video generation status.  
  - *Configuration:* Checks if `{{ $json.status }}` equals `"succeeded"`  
  - *Outputs:*  
    - True branch: Continue to fetch video output  
    - False branch: Loop back to Wait node for another polling cycle  
- **Wait**  
  - *Type:* Wait  
  - *Role:* Acts as loop delay before polling again.  
  - *Inputs:* False branch from Checks if complete node  
  - *Outputs:* Back to Get Status of the Video  
  - *Failures:* Infinite loop risk if video generation never completes or fails.

---

#### 2.7 Fetch Final Video

**Overview:**  
Downloads the final AI-generated video file after successful generation.

**Nodes Involved:**  
- Fetches the Video Output

**Node Details:**  
- **Fetches the Video Output**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the generated video from the URL provided by Replicate API output.  
  - *Configuration:*  
    - URL: `{{ $json.output }}` (direct video URL)  
    - Method: GET  
    - Headers: Authorization Bearer token (`YOUR_REPLICATE_API_KEY_HERE`)  
  - *Inputs:* True branch from Checks if complete node  
  - *Outputs:* Binary video data (MP4) ready for further processing or download  
  - *Failures:* Auth errors, broken URLs, network issues.

---

### 3. Summary Table

| Node Name                         | Node Type                    | Functional Role                         | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                                   |
|----------------------------------|------------------------------|---------------------------------------|----------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When chat message received        | Chat Trigger (LangChain)      | Receives Instagram Reel URL via chat  | None                       | Reel Downloader            | 🎯 STEP 1: Input Trigger. Captures URL in `{{ $json.chatInput }}`. No config needed.                                           |
| Reel Downloader                  | HTTP Request                 | Calls RapidAPI to get Reel metadata   | When chat message received | Formatter                 | 📥 STEP 2: Download Instagram Reel. Replace `YOUR_RAPIDAPI_KEY_HERE`. Returns video URL in `data.medias[0].url`.              |
| Formatter                       | Set                         | Extracts and stores video URL          | Reel Downloader            | Checks Null/ Not          | 🔄 STEP 3: Format data. Assigns `Instagram Reel` variable from API response. No config needed.                                 |
| Checks Null/ Not                | If                          | Validates video URL presence           | Formatter                  | Reel Downloader (retry) / Download Reel | ✅ STEP 4: Validation check. Loops on null URL to retry, else continue. No config needed.                                      |
| Download Reel                  | HTTP Request                 | Downloads Instagram Reel video binary  | Checks Null/ Not           | Upload the video to model  | 🎥 STEP 5: Download video file as binary. No config needed.                                                                    |
| Upload the video to model        | HTTP Request                 | Uploads binary video to Google AI      | Download Reel              | Analyse the Video          | ☁️ STEP 6: Upload to Google AI. Replace `YOUR_GOOGLE_API_KEY_HERE`. Header config for upload commands.                          |
| Analyse the Video                | HTTP Request                 | Analyzes video with Gemini 2.0 AI      | Upload the video to model  | Create the Video           | 🤖 STEP 7: AI Video Analysis. Replace `YOUR_GOOGLE_API_KEY_HERE`. Outputs detailed text description of video.                   |
| Create the Video                | HTTP Request                 | Sends text prompt to Minimax Video Gen | Analyse the Video          | Pause                     | 🎬 STEP 8: Generate new video. Replace `YOUR_REPLICATE_API_KEY_HERE`. Returns prediction ID.                                   |
| Pause                          | Wait                        | Waits 2 minutes for video generation   | Create the Video           | Get Status of the Video    | ⏸️ STEP 9: Wait period to allow generation. Adjustable amount parameter.                                                       |
| Get Status of the Video         | HTTP Request                 | Polls Replicate API for generation status | Pause / Wait              | Checks if the video generation is complete | 🔍 STEP 10: Check status. Replace `YOUR_REPLICATE_API_KEY_HERE`. Returns status.                                               |
| Checks if the video generation is complete | If                          | Branches based on generation status    | Get Status of the Video    | Fetches the Video Output / Wait | ✅ STEP 11: Status decision. Loops to Wait if processing, else fetch video.                                                    |
| Wait                          | Wait                        | Delay node for polling loop             | Checks if the video generation is complete (false branch) | Get Status of the Video | 🔁 Wait Loop Node. Polls until video generation complete.                                                                       |
| Fetches the Video Output        | HTTP Request                 | Downloads the AI-generated video        | Checks if the video generation is complete (true branch) | None                      | 🎉 STEP 12: Download generated video. Replace `YOUR_REPLICATE_API_KEY_HERE`. Output is binary video file ready for use.        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: `When chat message received`  
   - Set to public webhook, no other options.  
   - This node listens for user messages containing Instagram Reel URLs.

2. **Create HTTP Request Node to Download Instagram Reel Metadata**  
   - Name: `Reel Downloader`  
   - Method: GET  
   - URL: `https://instagram-reels-downloader-api.p.rapidapi.com/download`  
   - Query Parameters: `url={{ $json.chatInput }}`  
   - Headers:  
     - `x-rapidapi-host`: `instagram-reels-downloader-api.p.rapidapi.com`  
     - `x-rapidapi-key`: Replace with your RapidAPI key  
   - Max Retries: 2  
   - Retry on Fail: Enabled

3. **Create Set Node to Extract Video URL**  
   - Name: `Formatter`  
   - Add assignment:  
     - Variable: `Instagram Reel`  
     - Value: `{{ $json.data.medias[0].url }}`  
   - This simplifies accessing the video URL in next steps.

4. **Create If Node for Validation**  
   - Name: `Checks Null/ Not`  
   - Condition: Check if `{{ $json['Instagram Reel'] }}` equals string `"null"`  
   - True branch: Connect back to `Reel Downloader` (retry)  
   - False branch: Continue to next node

5. **Create HTTP Request Node to Download Video**  
   - Name: `Download Reel`  
   - Method: GET  
   - URL: `{{ $json["Instagram Reel"] }}`  
   - Expect binary data output (video file)

6. **Create HTTP Request Node to Upload Video to Google AI**  
   - Name: `Upload the video to model`  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files?key=YOUR_GOOGLE_API_KEY_HERE` (replace API key)  
   - Headers:  
     - `X-Goog-Upload-Command`: `start, upload, finalize`  
     - `X-Goog-Upload-Header-Content-Length`: approximate size (optional)  
     - `X-Goog-Upload-Header-Content-Type`: `video/mp4`  
     - `Content-Type`: `video/mp4`  
   - Body: Binary data from previous node  
   - Send as binary data

7. **Create HTTP Request Node for Gemini Video Analysis**  
   - Name: `Analyse the Video`  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=YOUR_GOOGLE_API_KEY_HERE` (replace API key)  
   - Body (JSON):  
     ```json
     {
       "contents": [
         {
           "parts": [
             {
               "file_data": {
                 "mime_type": "video/mp4",
                 "file_uri": "{{ $json.file.uri }}"
               }
             },
             {
               "text": "You are a highly advanced video interpreter model Your role is to observe and analyze the full input video in detail and produce a complete and literal visual and audio explanation of the entire video sequence frame by frame The context is that your output will be passed as input to a downstream AI video generation model which will recreate the same video using only the output you generate Therefore your output must serve as a perfect structural and sensory blueprint for that video to be reproduced without any changes deviations or loss of fidelity Your task is to describe the video from start to end with extreme precision including every single visual auditory spatial and emotional detail you must include what appears in each frame the characters and subjects involved their movements gestures facial expressions eye direction clothing background objects camera movement zoom lighting angle shadows time of day environment sounds background audio tone and transitions between scenes You must describe it literally and completely Do not skip any detail or infer any abstract meaning The output must be returned as a single uninterrupted block of plain lowercase human readable english text without any formatting punctuation quotation marks commas periods hyphens special characters or line breaks This is required because your output will be embedded as a string inside a JSON field used for AI model chaining This is not a summary Your output must act as an instruction set for recreating the entire video exactly as it appears Do not interpret or shorten any part Write with complete continuity as if the generation model will reassemble the visuals and audio step by step from your description"
             }
           ]
         }
       ]
     }
     ```
   - Set Headers: `Content-Type: application/json` (corrected from original)  
   - Retry on Fail: Enabled (2 retries)

8. **Create HTTP Request Node to Generate Video with Minimax**  
   - Name: `Create the Video`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/models/minimax/video-01/predictions`  
   - Headers:  
     - `Authorization`: `Bearer YOUR_REPLICATE_API_KEY_HERE`  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "input": {
         "prompt": "{{ $json.candidates[0].content.parts[0].text }}"
       }
     }
     ```

9. **Create Wait Node for 2 Minutes**  
   - Name: `Pause`  
   - Wait time: 2 minutes (adjustable)

10. **Create HTTP Request Node to Check Video Generation Status**  
    - Name: `Get Status of the Video`  
    - Method: GET  
    - URL: `https://api.replicate.com/v1/predictions/{{ $json["id"] }}`  
    - Headers:  
      - `Authorization`: `Bearer YOUR_REPLICATE_API_KEY_HERE`

11. **Create If Node to Check Completion**  
    - Name: `Checks if the video generation is complete`  
    - Condition: Check if `{{ $json.status }}` equals `"succeeded"`  
    - True branch: Continue to fetch output  
    - False branch: Connect to Wait node to loop again

12. **Create Wait Node for Polling Loop**  
    - Name: `Wait`  
    - Wait time: 2 minutes (same as Pause)  
    - Connect false branch of completion check to this node  
    - Connect this node back to `Get Status of the Video` node

13. **Create HTTP Request Node to Fetch Final Video**  
    - Name: `Fetches the Video Output`  
    - Method: GET  
    - URL: `{{ $json.output }}`  
    - Headers:  
      - `Authorization`: `Bearer YOUR_REPLICATE_API_KEY_HERE`  
    - Output binary video file (MP4) ready for download or further processing

14. **Connect all nodes as per flow:**  
    - Chat Trigger → Reel Downloader → Formatter → Checks Null/ Not  
    - Checks Null/ Not (true) → Reel Downloader (retry)  
    - Checks Null/ Not (false) → Download Reel → Upload video → Analyse video → Create the Video → Pause → Get Status → Checks if complete  
    - Checks if complete (true) → Fetches the Video Output  
    - Checks if complete (false) → Wait → Get Status (loop)

15. **Replace all placeholder API keys with actual values:**  
    - RapidAPI key in Reel Downloader  
    - Google API key in Upload and Analyse nodes  
    - Replicate API token in Create the Video, Get Status, and Fetches the Video Output nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow automates Instagram Reel download, AI analysis using Google Gemini, and AI video regeneration via Replicate. Perfect for social media content creators seeking automated video variations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Workflow Overview and Use Case                                                                                      |
| Setup requires API keys: RapidAPI for Instagram Reels Downloader, Google AI Studio API key with Gemini enabled, and Replicate API token.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | RapidAPI: https://rapidapi.com<br>Google AI Studio: https://aistudio.google.com/apikey<br>Replicate: https://replicate.com |
| Video generation can take 2-5 minutes; the workflow uses polling and wait nodes to handle asynchronous completion. Adjust wait times as necessary.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Polling and wait mechanism in workflow                                                                               |
| The Gemini AI output must be an uninterrupted lowercase text block with no punctuation or line breaks to serve as a precise video generation prompt.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Gemini AI analysis node content and instructions                                                                    |
| Replace all placeholder keys (`YOUR_RAPIDAPI_KEY_HERE`, `YOUR_GOOGLE_API_KEY_HERE`, `YOUR_REPLICATE_API_KEY_HERE`) before activating workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Critical configuration step                                                                                          |
| Pro Tip: After final video fetch, add nodes to upload to cloud storage, send via email/webhook, or post to social media as per your workflow needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Suggested enhancements post-video generation                                                                         |
| Sticky notes in workflow provide detailed step-by-step guidance and are positioned near relevant nodes for easy reference.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Workflow sticky notes for user guidance                                                                              |

---

**Disclaimer:** This document is an analysis and documentation of a legal, automated n8n workflow. No illegal, offensive, or protected content is included. All data handled complies with applicable content policies.