Create & Share Lip-Synced Avatar Videos with Infinitalk AI & Upload to TikTok/YouTube

https://n8nworkflows.xyz/workflows/create---share-lip-synced-avatar-videos-with-infinitalk-ai---upload-to-tiktok-youtube-8912


# Create & Share Lip-Synced Avatar Videos with Infinitalk AI & Upload to TikTok/YouTube

### 1. Workflow Overview

This workflow automates the creation and sharing of lip-synced avatar videos using Infinitalk AI, starting from user-submitted video and audio URLs and a prompt. It integrates multiple services to generate a talking avatar video that lip-syncs naturally to the given audio, uploads the resulting video to Google Drive, generates an SEO-optimized YouTube title using OpenAI, and then publishes the video to YouTube and TikTok via the Upload-Post API.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Data Preparation:** Captures user input from a form and prepares data for video generation.
- **1.2 Video Generation via Infinitalk API:** Sends a request to create a lip-synced avatar video and polls for completion.
- **1.3 Video Retrieval and Storage:** Retrieves the completed video URL, downloads the video file, and uploads it to Google Drive.
- **1.4 AI-Generated Title Creation:** Uses OpenAI to generate an SEO-friendly YouTube title based on the prompt.
- **1.5 Video Upload to Social Platforms:** Uploads the video to YouTube and TikTok through Upload-Post API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Preparation

- **Overview:**  
  This block receives user input via a form trigger node, extracting the video URL, audio URL, and a text prompt. It organizes these inputs into a structured data format for downstream processing.

- **Nodes Involved:**  
  - On form submission1  
  - Set data  
  - Sticky Note6 (API key instruction)  
  - Sticky Note7 (API key reminder)  
  - Sticky Note8 (Upload-Post API setup instructions)  
  - Sticky Note1 (example input data)

- **Node Details:**  
  - **On form submission1**  
    - Type: Form Trigger  
    - Role: Entry point capturing user-submitted video URL, audio URL, and prompt via a web form.  
    - Config: Form titled "Video to Video" with required fields "Video url", "Audio url", and "Prompt".  
    - Outputs: JSON object containing user inputs.  
    - Edge cases: Missing required fields or malformed URLs might cause downstream failures.  
  - **Set data**  
    - Type: Set node  
    - Role: Maps form submission fields into variables `video_url`, `audio_url`, and `prompt` for easier referencing.  
    - Expressions: Uses expressions such as `={{ $('On form submission1').item.json.Prompt }}`.  
    - Input: From form submission node.  
    - Output: JSON with standardized variable names for next steps.  
    - Edge cases: Expression failures if input fields are missing.  
  - **Sticky Notes (6,7,8,1)**  
    - Provide guidance on obtaining API keys for Fal.ai and Upload-Post, usage instructions for authorization headers, and an example of input data.  
    - Context: Critical for users to configure credentials correctly.

#### 2.2 Video Generation via Infinitalk API

- **Overview:**  
  This block initiates the video creation request to Infinitalk AI, waits for processing, and polls the API until the video generation is complete.

- **Nodes Involved:**  
  - Create Video  
  - Wait 60 sec.  
  - Get status  
  - Completed? (If node)  
  - Sticky Note3 (workflow purpose description)

- **Node Details:**  
  - **Create Video**  
    - Type: HTTP Request  
    - Role: Sends POST request to Infinitalk's "video-to-video" endpoint with input video URL, audio URL, prompt, and generation parameters (num_frames=145, resolution=480p, seed=42, acceleration=regular).  
    - Authentication: Header Auth with "Authorization" set to "Key YOURAPIKEY" (Fal.run API).  
    - Input: Data from `Set data` node.  
    - Output: JSON containing a `request_id` for tracking generation status.  
    - Edge cases: API errors, invalid URLs, or auth failures.  
  - **Wait 60 sec.**  
    - Type: Wait node  
    - Role: Delays workflow to allow asynchronous video processing.  
    - Duration: 60 seconds.  
    - Input: After `Create Video` and `Get status` node based on polling loop.  
  - **Get status**  
    - Type: HTTP Request  
    - Role: Polls the Infinitalk API endpoint `/status` with the `request_id` to check processing status.  
    - Authentication: Same as `Create Video`.  
    - Output: Status string such as "COMPLETED" or "PROCESSING".  
    - Edge cases: Network timeouts, API rate limits.  
  - **Completed?**  
    - Type: If node  
    - Role: Checks if `status == "COMPLETED"`.  
    - True branch: proceeds to video retrieval.  
    - False branch: loops back to `Wait 60 sec.` for another polling cycle.  
  - **Sticky Note3**  
    - Summarizes the entire workflow purpose and key steps.

#### 2.3 Video Retrieval and Storage

- **Overview:**  
  Once the video generation is complete, this block retrieves the video URL, downloads the video file, and uploads it to a designated Google Drive folder.

- **Nodes Involved:**  
  - Get Url Video  
  - Generate title  
  - Get File Video  
  - Upload Video  
  - Sticky Note (Set YOUR_USERNAME reminder)

- **Node Details:**  
  - **Get Url Video**  
    - Type: HTTP Request  
    - Role: Fetches the video metadata including the video download URL by querying the Infinitalk API with `request_id`.  
    - Authentication: Same as previous Fal.run API calls.  
    - Output: JSON containing `video.url` and `video.file_name`.  
  - **Generate title**  
    - Type: OpenAI (LangChain) node  
    - Role: Generates a YouTube SEO-optimized video title based on the prompt using GPT-5-mini model.  
    - Input: The `prompt` from `Set data` node.  
    - Parameters: Instructions specify max 60 characters, keywords, catchy style, no clickbait, same language output.  
    - Credentials: OpenAI API key required.  
    - Output: Generated title string.  
  - **Get File Video**  
    - Type: HTTP Request  
    - Role: Downloads the actual video file from the URL obtained in `Get Url Video`.  
    - Input: URL dynamically from previous node output.  
    - Output: Binary video data for upload.  
  - **Upload Video**  
    - Type: Google Drive node  
    - Role: Uploads the downloaded video to a specific Google Drive folder.  
    - Configuration: Dynamic filename using timestamp and original file name; target folder ID specified.  
    - Credentials: Google Drive OAuth2 configured.  
    - Edge cases: Upload failures, quota limits, auth expiration.  
  - **Sticky Note**  
    - Reminder for users to set their username for upload services.

#### 2.4 Video Upload to Social Platforms

- **Overview:**  
  This block uploads the generated video, along with the AI-generated title, to YouTube and TikTok using the Upload-Post API.

- **Nodes Involved:**  
  - Upload to Youtube  
  - Upload on TikTok  
  - Sticky Note8 (Upload-Post API instructions)

- **Node Details:**  
  - **Upload to Youtube**  
    - Type: HTTP Request  
    - Role: Sends a multipart-form POST request to Upload-Post API to upload the video to YouTube platform.  
    - Inputs: Title from `Generate title`, user profile name (`YOUR_USERNAME` placeholder), binary video data.  
    - Authentication: Header Auth with API key from Upload-Post.  
    - Edge cases: API errors, invalid credentials, platform restrictions.  
  - **Upload on TikTok**  
    - Type: HTTP Request  
    - Role: Identical to YouTube upload but targets TikTok platform.  
    - Note: Free Upload-Post plan does not support TikTok uploads; requires paid plan.  
    - Edge cases: Same as above plus TikTok-specific upload limitations.  
  - **Sticky Note8**  
    - Explains Upload-Post API key acquisition, header setup, profile creation, and free plan limitations.

---

### 3. Summary Table

| Node Name         | Node Type            | Functional Role                         | Input Node(s)          | Output Node(s)              | Sticky Note                                  |
|-------------------|----------------------|---------------------------------------|------------------------|-----------------------------|----------------------------------------------|
| On form submission1| Form Trigger         | Capture user inputs (video URL, audio URL, prompt) | —                      | Set data                    |                                              |
| Set data          | Set                  | Prepare and structure input data      | On form submission1     | Create Video                |                                              |
| Create Video      | HTTP Request         | Request video generation from Infinitalk API | Set data                | Wait 60 sec.                |                                              |
| Wait 60 sec.      | Wait                 | Pause for processing time              | Create Video, Get status | Get status                  |                                              |
| Get status        | HTTP Request         | Poll Infinitalk API for generation status | Wait 60 sec.            | Completed?                  |                                              |
| Completed?        | If                   | Check if video generation is complete | Get status              | Get Url Video (true), Wait 60 sec. (false) |                                              |
| Get Url Video     | HTTP Request         | Get video URL and metadata             | Completed? (true)       | Generate title              |                                              |
| Generate title    | OpenAI (LangChain)   | Create SEO-optimized video title       | Get Url Video           | Get File Video              |                                              |
| Get File Video    | HTTP Request         | Download generated video file          | Generate title          | Upload Video, Upload to Youtube, Upload on TikTok |                                              |
| Upload Video      | Google Drive         | Upload video to Google Drive folder    | Get File Video          | —                           |                                              |
| Upload to Youtube | HTTP Request         | Upload video to YouTube via Upload-Post API | Get File Video          | —                           |                                              |
| Upload on TikTok  | HTTP Request         | Upload video to TikTok via Upload-Post API | Get File Video          | —                           |                                              |
| Sticky Note3      | Sticky Note          | Workflow purpose and summary           | —                      | —                           |                                              |
| Sticky Note6      | Sticky Note          | API Key setup instructions for Fal.ai | —                      | —                           |                                              |
| Sticky Note7      | Sticky Note          | Reminder to set API Key                 | —                      | —                           |                                              |
| Sticky Note8      | Sticky Note          | Upload-Post API key and profile setup  | —                      | —                           |                                              |
| Sticky Note1      | Sticky Note          | Example input data                      | —                      | —                           |                                              |
| Sticky Note       | Sticky Note          | Reminder to set YOUR_USERNAME          | —                      | —                           |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Configure a webhook with a form titled "Video to Video".  
   - Add required fields: "Video url" (string), "Audio url" (string), "Prompt" (string).  
   - This node will receive user inputs to start the workflow.

2. **Add a Set Node (named "Set data")**  
   - Purpose: Map form fields to variables for downstream use.  
   - Assignments:  
     - `prompt` = Expression: `{{$json["Prompt"]}}` from form trigger.  
     - `video_url` = Expression: `{{$json["Video url"]}}`.  
     - `audio_url` = Expression: `{{$json["Audio url"]}}`.  
   - Connect the form trigger node to this Set node.

3. **Create an HTTP Request Node ("Create Video")**  
   - Configure POST request to `https://queue.fal.run/fal-ai/infinitalk/video-to-video`.  
   - Authentication: HTTP Header Auth with:  
     - Header Name: `Authorization`  
     - Header Value: `Key YOURAPIKEY` (replace with your Fal.ai API key).  
   - Body (JSON):  
     ```json
     {
       "video_url": "={{ $json.video_url }}",
       "audio_url": "={{ $json.audio_url }}",
       "prompt": "={{ $json.prompt }}",
       "num_frames": 145,
       "resolution": "480p",
       "seed": 42,
       "acceleration": "regular"
     }
     ```  
   - Connect `Set data` node to this HTTP Request node.

4. **Add a Wait Node ("Wait 60 sec.")**  
   - Set to wait 60 seconds.  
   - Connect output of "Create Video" node to this node.

5. **Add an HTTP Request Node ("Get status")**  
   - Configure GET request to:  
     `https://queue.fal.run/fal-ai/infinitalk/requests/{{ $json.request_id }}/status`  
   - Use same HTTP Header Auth credentials as "Create Video".  
   - Connect "Wait 60 sec." node output here.

6. **Add an If Node ("Completed?")**  
   - Condition: Check if `status` equals `"COMPLETED"`.  
   - Input: output of "Get status" node.  
   - True branch: Connect to next step (video retrieval).  
   - False branch: Loop back to "Wait 60 sec." node for polling.

7. **Add an HTTP Request Node ("Get Url Video")**  
   - Configure GET request to:  
     `https://queue.fal.run/fal-ai/infinitalk/requests/{{ $json.request_id }}`  
   - Same authentication as previous Fal.ai nodes.  
   - Connect the true branch of "Completed?" node here.

8. **Add an OpenAI Node ("Generate title")**  
   - Use LangChain OpenAI node.  
   - Model: `gpt-5-mini`.  
   - Input message: Pass the prompt from "Set data" node.  
   - System message: Provide instructions to generate an SEO-friendly YouTube title, max 60 characters, same language, no clickbait, keyword-rich, catchy.  
   - Credentials: OpenAI API key configured.  
   - Connect "Get Url Video" node output to this node.

9. **Add an HTTP Request Node ("Get File Video")**  
   - Download the video file from URL: `={{ $('Get Url Video').item.json.video.url }}`.  
   - No authentication needed.  
   - Connect "Generate title" node output here.

10. **Add a Google Drive Node ("Upload Video")**  
    - Upload binary video data from "Get File Video".  
    - Set filename: `{{ $now.format('yyyyLLddHHmmss') }}-{{ $('Get Url Video').item.json.video.file_name }}`.  
    - Target folder: Specify Google Drive folder ID.  
    - Credentials: Google Drive OAuth2 configured.  
    - Connect "Get File Video" to this node.

11. **Add HTTP Request Node ("Upload to Youtube")**  
    - POST to `https://api.upload-post.com/api/upload`.  
    - Content Type: multipart/form-data.  
    - Body parameters:  
      - `title`: From "Generate title" node output.  
      - `user`: Your Upload-Post profile username.  
      - `platform[]`: `"youtube"`.  
      - `video`: binary data from "Get File Video".  
    - Authentication: Header Auth with Upload-Post API key.  
    - Connect "Get File Video" node output here.

12. **Add HTTP Request Node ("Upload on TikTok")**  
    - Same as YouTube upload but set `platform[]` to `"tiktok"`.  
    - Note: Requires paid Upload-Post plan for TikTok uploads.  
    - Connect "Get File Video" node output here.

13. **Add Sticky Notes** to the canvas for:  
    - API key setup instructions for Fal.ai (Authorization header).  
    - Upload-Post API key and profile creation instructions.  
    - Example input data for testing.  
    - Workflow purpose summary.  
    - Reminder to set your social media username in upload nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Create an account at [https://fal.ai/](https://fal.ai/) to obtain your API key for video generation. | Fal.ai API key signup |
| Upload-Post API key management and profile creation is available at [https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) | Upload-Post API documentation and signup |
| The workflow uses GPT-5-mini model for AI-generated titles; ensure your OpenAI API credentials have access to this or a comparable model. | OpenAI API |
| Free Upload-Post plan allows YouTube uploads but TikTok uploads require plan upgrade. | Upload-Post subscription details |
| Video files are uploaded to a specific Google Drive folder; ensure the Google Drive OAuth2 credentials have access and quota. | Google Drive integration |
| Example test input prompt: "A woman with colorful hair talking on a podcast." with sample video and audio URLs is included in sticky notes. | Example test data |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. All processing complies with content policies and contains no illegal, offensive, or protected materials. All handled data is legal and public.