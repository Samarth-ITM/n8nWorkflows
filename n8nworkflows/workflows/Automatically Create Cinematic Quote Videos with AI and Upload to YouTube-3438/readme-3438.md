Automatically Create Cinematic Quote Videos with AI and Upload to YouTube

https://n8nworkflows.xyz/workflows/automatically-create-cinematic-quote-videos-with-ai-and-upload-to-youtube-3438


# Automatically Create Cinematic Quote Videos with AI and Upload to YouTube

### 1. Workflow Overview

This workflow automates the creation of cinematic vertical quote videos in Thai language, integrating AI-generated imagery, video, ambient sound, and text overlays, then uploads the final video to YouTube while updating a Google Sheet with all relevant media URLs. It is designed for digital content creators and marketers who want to streamline and scale the production of inspirational or motivational quote videos optimized for social media platforms like YouTube Shorts, Instagram Reels, and TikTok.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Quote Retrieval:** Trigger and fetch quote data from Google Sheets.
- **1.2 AI Image Generation:** Generate a vertical background image using Flux AI.
- **1.3 AI Video Generation:** Convert the generated image into a cinematic vertical video using Kling API.
- **1.4 AI Audio Generation:** Create ambient background sound with ElevenLabs API.
- **1.5 Media Storage and URL Updates:** Save generated media locally or to Google Drive and update Google Sheets with URLs.
- **1.6 Text Overlay Preparation:** Prepare dynamic text overlay commands for FFmpeg based on the quote and author.
- **1.7 Final Video Composition:** Combine video, audio, and text overlay into a final video clip using FFmpeg.
- **1.8 YouTube Upload and Status Update:** Upload the final video to YouTube and update the Google Sheet with the YouTube URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Quote Retrieval

- **Overview:**  
  This block initiates the workflow manually and retrieves a single quote entry from Google Sheets that has not yet been processed (filtered by empty "Video Status").

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get data from Google Sheet

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or execution  
    - Configuration: No parameters  
    - Inputs: None  
    - Outputs: Triggers next node  
    - Edge Cases: None  
  - **Get data from Google Sheet**  
    - Type: Google Sheets  
    - Role: Fetches the first quote entry where "Video Status" is empty (unprocessed)  
    - Configuration:  
      - Document ID: Provided Google Sheet ID  
      - Sheet: gid=0 (Sheet1)  
      - Filter: Return first match where "Video Status" is empty  
    - Inputs: Trigger from manual node  
    - Outputs: JSON with quote data including Index, Quote (Thai), Pen Name (Thai), Background (EN), Prompt (EN)  
    - Edge Cases: No matching rows found (workflow should handle empty data gracefully)  
    - Credentials: Google Sheets OAuth2

---

#### 1.2 AI Image Generation

- **Overview:**  
  Generates a vertical background image based on the quote’s background description and prompt using Flux AI’s txt2img API.

- **Nodes Involved:**  
  - Generate Image  
  - Wait image 2 min  
  - Get image  
  - Update image background URL  
  - Sticky Note (Create Image Background)

- **Node Details:**  
  - **Generate Image**  
    - Type: HTTP Request  
    - Role: Sends POST request to Flux AI txt2img endpoint with prompt and negative prompt  
    - Configuration:  
      - URL: https://api.piapi.ai/api/v1/task  
      - Body: JSON with model "Qubico/flux1-dev", task_type "txt2img", prompt constructed dynamically from Google Sheet fields  
      - Headers: X-API-Key with Flux API key  
      - Image size: 540x960 (vertical)  
    - Inputs: Quote data from Google Sheets  
    - Outputs: Task ID for image generation  
    - Edge Cases: API key invalid, rate limits, network errors  
  - **Wait image 2 min**  
    - Type: Wait  
    - Role: Pauses workflow 2 minutes to allow image generation to complete  
    - Inputs: From Generate Image  
    - Outputs: Triggers Get image  
  - **Get image**  
    - Type: HTTP Request  
    - Role: Polls Flux API to retrieve generated image by task ID  
    - Configuration: GET request to task endpoint with task_id  
    - Inputs: Task ID from Generate Image  
    - Outputs: JSON with image URL  
    - Edge Cases: Image not ready, API errors  
  - **Update image background URL**  
    - Type: Google Sheets  
    - Role: Updates the "Background Image" column in Google Sheet with generated image URL  
    - Configuration: Matches row by "Index" and updates "Background Image"  
    - Inputs: Image URL from Get image and quote data  
    - Outputs: Triggers next block (Image-to-Video)  
    - Credentials: Google Sheets OAuth2  
  - **Sticky Note (Create Image Background)**  
    - Type: Sticky Note  
    - Role: Documentation for this block  
    - Content: "Generate an image using prompt from Google Sheet via PiAPI Flux (Txt2img)."

---

#### 1.3 AI Video Generation

- **Overview:**  
  Converts the generated image into a cinematic vertical video with subtle animation using Kling API.

- **Nodes Involved:**  
  - Image-to-Video  
  - Wait video 5 min  
  - Get Video  
  - Get Binary Video Background  
  - Save Video Background Locally1  
  - Update video background URL  
  - Sticky Note1 (Create Video Background)

- **Node Details:**  
  - **Image-to-Video**  
    - Type: HTTP Request  
    - Role: Sends POST request to Kling API for video generation from image URL  
    - Configuration:  
      - URL: https://api.piapi.ai/api/v1/task  
      - Body: JSON with model "kling", task_type "video_generation", prompt constructed dynamically from Google Sheet fields, image_url from Get image  
      - Duration: 5 seconds  
      - Camera control: zoom 5, no pan/tilt  
      - Headers: X-API-Key with Flux API key  
    - Inputs: Image URL from Update image background URL  
    - Outputs: Task ID for video generation  
    - Edge Cases: API errors, invalid image URL  
  - **Wait video 5 min**  
    - Type: Wait  
    - Role: Pauses workflow 5 minutes to allow video generation to complete  
    - Inputs: From Image-to-Video  
    - Outputs: Triggers Get Video  
  - **Get Video**  
    - Type: HTTP Request  
    - Role: Polls Kling API to retrieve generated video by task ID  
    - Configuration: GET request to task endpoint with task_id  
    - Inputs: Task ID from Image-to-Video  
    - Outputs: JSON with video URL  
    - Edge Cases: Video not ready, API errors  
  - **Get Binary Video Background**  
    - Type: HTTP Request  
    - Role: Downloads the video file as binary data for local saving  
    - Configuration: URL from Get Video output video_url, response format file  
    - Inputs: Video URL from Get Video  
    - Outputs: Binary video data  
  - **Save Video Background Locally1**  
    - Type: Read/Write File  
    - Role: Saves downloaded video binary as "VideoBackground.mp4" locally on the self-hosted n8n instance  
    - Inputs: Binary video data from Get Binary Video Background  
    - Outputs: Triggers Update video background URL  
  - **Update video background URL**  
    - Type: Google Sheets  
    - Role: Updates "Background Video" column in Google Sheet with video URL from Get Video  
    - Inputs: Video URL and quote data  
    - Outputs: Triggers Generate Audio  
    - Credentials: Google Sheets OAuth2  
  - **Sticky Note1 (Create Video Background)**  
    - Type: Sticky Note  
    - Role: Documentation for this block  
    - Content: "Create a cinematic vertical video from the generated image using PiAPI Kling."

---

#### 1.4 AI Audio Generation

- **Overview:**  
  Generates a 5-second ambient background soundscape using ElevenLabs sound generation API based on the scene description.

- **Nodes Involved:**  
  - Generate Audio  
  - Upload Sound to Google Drive  
  - Save Music Background Locally1  
  - Update Sound background URL  
  - Sticky Note2 (Create Sound Background)

- **Node Details:**  
  - **Generate Audio**  
    - Type: HTTP Request  
    - Role: Sends POST request to ElevenLabs sound generation API with a descriptive text prompt for ambient sound  
    - Configuration:  
      - URL: https://api.elevenlabs.io/v1/sound-generation  
      - Body: JSON with text prompt dynamically constructed from Google Sheet fields, duration 5 seconds, model "sound-effects-v1", output mp3  
      - Headers: xi-api-key with ElevenLabs API key  
    - Inputs: Quote data from Google Sheet  
    - Outputs: Binary audio data and metadata including webContentLink  
    - Edge Cases: API key invalid, rate limits, network errors  
  - **Upload Sound to Google Drive**  
    - Type: Google Drive  
    - Role: Uploads generated mp3 audio file to specified Google Drive folder  
    - Configuration:  
      - File name based on Background (EN) field  
      - Drive ID and folder ID specified  
    - Inputs: Audio binary data from Generate Audio  
    - Outputs: Google Drive file metadata including webContentLink  
    - Credentials: Google Drive OAuth2  
  - **Save Music Background Locally1**  
    - Type: Read/Write File  
    - Role: Saves audio binary as "SoundBackground.mp3" locally for FFmpeg use  
    - Inputs: Audio binary data from Generate Audio  
    - Outputs: Triggers Prepare Overlay Text  
  - **Update Sound background URL**  
    - Type: Google Sheets  
    - Role: Updates "Music Background" column in Google Sheet with Google Drive URL of uploaded audio  
    - Inputs: Google Drive file metadata and quote data  
    - Credentials: Google Sheets OAuth2  
  - **Sticky Note2 (Create Sound Background)**  
    - Type: Sticky Note  
    - Role: Documentation for this block  
    - Content: "Generate ambient sound using ElevenLabs based on the scene prompt."

---

#### 1.5 Text Overlay Preparation

- **Overview:**  
  Prepares FFmpeg drawtext filter commands to overlay the Thai quote and author text dynamically on the video, ensuring proper line breaks and typography using the Kanit font.

- **Nodes Involved:**  
  - Prepare Overlay Text (Quote & Author)1

- **Node Details:**  
  - **Prepare Overlay Text (Quote & Author)1**  
    - Type: Code (JavaScript)  
    - Role:  
      - Reads quote and author from Google Sheet data  
      - Calculates line breaks based on video width and font size  
      - Constructs FFmpeg drawtext filter commands for each line of quote and author text with positioning and font styling  
    - Configuration:  
      - Quote font: Kanit-Italic.ttf, size 70  
      - Author font: Kanit-Italic.ttf, size 50  
      - Font color: white  
      - Video width: 1080px  
      - Margin: 40px  
    - Inputs: Quote and author text from Google Sheet  
    - Outputs: JSON with drawText filter string for FFmpeg  
    - Edge Cases: Missing quote or author throws error  
    - Version: Requires n8n supporting JavaScript code node v2

---

#### 1.6 Final Video Composition

- **Overview:**  
  Combines the cinematic video background, ambient audio, and text overlay into a final vertical video clip using FFmpeg executed locally.

- **Nodes Involved:**  
  - Generate Final Video Clip1  
  - Sticky Note5 (Combine All)

- **Node Details:**  
  - **Generate Final Video Clip1**  
    - Type: Execute Command  
    - Role: Runs FFmpeg command to:  
      - Scale and crop video to 1080x1920 vertical format  
      - Overlay a semi-transparent black background for text readability  
      - Apply the prepared drawtext filter for quote and author  
      - Mix audio with volume adjustment  
      - Output final video as output.mp4  
    - Configuration:  
      - Command dynamically references local files "VideoBackground.mp4" and "SoundBackground.mp3"  
      - Uses drawText filter string from previous node  
      - Video codec: libx264, audio codec: aac  
      - Aspect ratio: 9:16  
      - Overwrites output file if exists  
    - Inputs: Local video and audio files, drawText filter string  
    - Outputs: Triggers YouTube upload  
    - Requirements: FFmpeg installed on self-hosted n8n instance, Kanit font installed and accessible to FFmpeg  
    - Edge Cases: FFmpeg execution errors, missing files, font issues  
  - **Sticky Note5 (Combine All)**  
    - Type: Sticky Note  
    - Role: Documentation for this block  
    - Content: "Merge video, sound, and quote text into final clip using FFmpeg."

---

#### 1.7 YouTube Upload and Status Update

- **Overview:**  
  Uploads the final video to YouTube using the resumable upload API and updates the Google Sheet with the YouTube video URL.

- **Nodes Involved:**  
  - Initiate YouTube Resumable Upload  
  - Read output file  
  - Upload Video to YouTube  
  - Update Quote Upload Status  
  - Sticky Note3 (Video Upload & Post-Processing)

- **Node Details:**  
  - **Initiate YouTube Resumable Upload**  
    - Type: HTTP Request  
    - Role: Initiates a resumable upload session with YouTube API, sending video metadata (title, description, privacy)  
    - Configuration:  
      - URL: YouTube upload endpoint with part=snippet,status and uploadType=resumable  
      - Body: JSON with snippet (title and description from quote and author), status (public, license, embeddable)  
      - Headers: Content-Type application/json, X-Upload-Content-Type video/webm  
      - Authentication: YouTube OAuth2  
    - Inputs: Final video metadata from Google Sheet  
    - Outputs: Upload session URL in response headers  
    - Edge Cases: Auth errors, quota limits  
  - **Read output file**  
    - Type: Read/Write File  
    - Role: Reads the locally saved final video file "output.mp4" as binary data for upload  
    - Inputs: Trigger from previous node  
    - Outputs: Binary video data  
  - **Upload Video to YouTube**  
    - Type: HTTP Request  
    - Role: Uploads video binary data to YouTube using the resumable upload URL from previous node  
    - Configuration:  
      - Method: PUT  
      - URL: From Initiate YouTube Resumable Upload response header location  
      - Headers: Content-Type video/webm  
      - Authentication: YouTube OAuth2  
      - Input data field: binary data from Read output file  
    - Inputs: Binary video data and upload URL  
    - Outputs: YouTube video ID in response JSON  
    - Edge Cases: Upload interruptions, auth errors  
  - **Update Quote Upload Status**  
    - Type: Google Sheets  
    - Role: Updates "Video Status" column in Google Sheet with YouTube video URL constructed from returned video ID  
    - Inputs: YouTube video ID and quote data  
    - Credentials: Google Sheets OAuth2  
  - **Sticky Note3 (Video Upload & Post-Processing)**  
    - Type: Sticky Note  
    - Role: Documentation for this block  
    - Content: "Upload the final video to YouTube using the YouTube API and update your Google Sheets with upload statuses and YouTube links."

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                      |
|--------------------------------|---------------------|-----------------------------------------------|----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger      | Workflow start trigger                         | None                             | Get data from Google Sheet        |                                                                                                |
| Get data from Google Sheet      | Google Sheets       | Retrieve quote data                            | When clicking ‘Test workflow’    | Generate Image                   | ## Get Quote: Retrieve quote data from Google Sheets including text, author, and background prompts. |
| Generate Image                 | HTTP Request        | Generate vertical background image via Flux AI| Get data from Google Sheet       | Wait image 2 min                 | ## Create Image Background: Generate an image using prompt from Google Sheet via PiAPI Flux (Txt2img). |
| Wait image 2 min               | Wait                | Wait for image generation completion          | Generate Image                  | Get image                       |                                                                                                |
| Get image                     | HTTP Request        | Retrieve generated image URL                   | Wait image 2 min                | Update image background URL      |                                                                                                |
| Update image background URL    | Google Sheets       | Update Google Sheet with image URL             | Get image                      | Image-to-Video                  |                                                                                                |
| Image-to-Video                | HTTP Request        | Generate cinematic video from image via Kling | Update image background URL     | Wait video 5 min                | ## Create Video Background: Create a cinematic vertical video from the generated image using PiAPI Kling. |
| Wait video 5 min              | Wait                | Wait for video generation completion           | Image-to-Video                 | Get Video                      |                                                                                                |
| Get Video                    | HTTP Request        | Retrieve generated video URL                    | Wait video 5 min               | Get Binary Video Background      |                                                                                                |
| Get Binary Video Background    | HTTP Request        | Download video binary data                       | Get Video                     | Save Video Background Locally1   |                                                                                                |
| Save Video Background Locally1 | Read/Write File     | Save video locally for FFmpeg processing        | Get Binary Video Background    | Update video background URL      |                                                                                                |
| Update video background URL    | Google Sheets       | Update Google Sheet with video URL              | Save Video Background Locally1 | Generate Audio                 |                                                                                                |
| Generate Audio               | HTTP Request        | Generate ambient sound via ElevenLabs           | Update video background URL    | Upload Sound to Google Drive, Save Music Background Locally1 | ## Create Sound Background: Generate ambient sound using ElevenLabs based on the scene prompt. |
| Upload Sound to Google Drive   | Google Drive        | Upload generated audio to Google Drive          | Generate Audio                | Update Sound background URL      |                                                                                                |
| Save Music Background Locally1 | Read/Write File     | Save audio locally for FFmpeg                    | Generate Audio                | Prepare Overlay Text (Quote & Author)1 |                                                                                                |
| Update Sound background URL    | Google Sheets       | Update Google Sheet with audio URL               | Upload Sound to Google Drive   | None                          |                                                                                                |
| Prepare Overlay Text (Quote & Author)1 | Code (JavaScript) | Prepare FFmpeg drawtext filter commands          | Save Music Background Locally1 | Generate Final Video Clip1       |                                                                                                |
| Generate Final Video Clip1     | Execute Command     | Combine video, audio, and text overlay via FFmpeg| Prepare Overlay Text (Quote & Author)1 | Initiate YouTube Resumable Upload | ## Combine All: Merge video, sound, and quote text into final clip using FFmpeg.                |
| Initiate YouTube Resumable Upload | HTTP Request    | Start YouTube video upload session               | Generate Final Video Clip1     | Read output file                | ## Video Upload & Post-Processing: Upload the final video to YouTube using the YouTube API and update your Google Sheets with upload statuses and YouTube links. |
| Read output file              | Read/Write File     | Read final video file for upload                  | Initiate YouTube Resumable Upload | Upload Video to YouTube         |                                                                                                |
| Upload Video to YouTube        | HTTP Request        | Upload video binary to YouTube                    | Read output file              | Update Quote Upload Status       |                                                                                                |
| Update Quote Upload Status     | Google Sheets       | Update Google Sheet with YouTube video URL       | Upload Video to YouTube       | None                          |                                                                                                |
| Sticky Note                   | Sticky Note         | Documentation                                    | None                         | None                          | ## Create Image Background: Generate an image using prompt from Google Sheet via PiAPI Flux (Txt2img). |
| Sticky Note1                  | Sticky Note         | Documentation                                    | None                         | None                          | ## Create Video Background: Create a cinematic vertical video from the generated image using PiAPI Kling. |
| Sticky Note2                  | Sticky Note         | Documentation                                    | None                         | None                          | ## Create Sound Background: Generate ambient sound using ElevenLabs based on the scene prompt. |
| Sticky Note3                  | Sticky Note         | Documentation                                    | None                         | None                          | ## Video Upload & Post-Processing: Upload the final video to YouTube using the YouTube API and update your Google Sheets with upload statuses and YouTube links. |
| Sticky Note4                  | Sticky Note         | Documentation                                    | None                         | None                          | ## Get Quote: Retrieve quote data from Google Sheets including text, author, and background prompts. |
| Sticky Note5                  | Sticky Note         | Documentation                                    | None                         | None                          | ## Combine All: Merge video, sound, and quote text into final clip using FFmpeg.                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually for testing or execution.

2. **Add Google Sheets Node to Fetch Quote:**  
   - Type: Google Sheets  
   - Operation: Read (return first match)  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: gid=0 (Sheet1)  
   - Filter: "Video Status" is empty (unprocessed)  
   - Credentials: Google Sheets OAuth2

3. **Add HTTP Request Node to Generate Image:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: https://api.piapi.ai/api/v1/task  
   - Headers: X-API-Key with your Flux API key  
   - Body (raw JSON):  
     ```json
     {
       "model": "Qubico/flux1-dev",
       "task_type": "txt2img",
       "input": {
         "prompt": "Ultra-realistic vertical nature landscape, {{ Background (EN) }}, featuring {{ Prompt (EN) }}, high detail, soft atmospheric lighting, cinematic golden hour glow, vertical composition, photorealistic texture, natural depth of field, calm and serene mood, no people, no buildings, HDR, 8k resolution, masterpiece",
         "negative_prompt": "taking a photo of a room, recording a video of a room, photos app, video recorder, illegible text, blurry text, low quality text, DSLR, unnatural",
         "width": 540,
         "height": 960
       }
     }
     ```
   - Use expressions to insert Google Sheet fields dynamically.

4. **Add Wait Node:**  
   - Type: Wait  
   - Duration: 2 minutes  
   - Purpose: Allow image generation to complete.

5. **Add HTTP Request Node to Get Image:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://api.piapi.ai/api/v1/task/{{task_id}} (use task_id from previous node)  
   - Headers: X-API-Key with Flux API key

6. **Add Google Sheets Node to Update Image URL:**  
   - Type: Google Sheets  
   - Operation: Update row matching "Index"  
   - Update "Background Image" column with image URL from previous node  
   - Credentials: Google Sheets OAuth2

7. **Add HTTP Request Node to Generate Video:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: https://api.piapi.ai/api/v1/task  
   - Headers: X-API-Key with Flux API key  
   - Body (raw JSON):  
     ```json
     {
       "model": "kling",
       "task_type": "video_generation",
       "input": {
         "prompt": "Cinematic vertical video from image of {{ Background (EN) }}, with {{ Prompt (EN) }}, animated subtly from image, with soft light, mist, swaying trees, slow zoom effect, no people or buildings",
         "negative_prompt": "blurry motion, distorted faces, unnatural lighting, over produced, bad quality",
         "cfg_scale": 0.5,
         "duration": 5,
         "mode": "std",
         "image_url": "{{ Background Image }}",
         "version": "1.0",
         "camera_control": {
           "type": "simple",
           "config": {
             "horizontal": 0,
             "vertical": 0,
             "pan": 0,
             "tilt": 0,
             "roll": 0,
             "zoom": 5
           }
         }
       }
     }
     ```
   - Use expressions to insert dynamic fields.

8. **Add Wait Node:**  
   - Type: Wait  
   - Duration: 5 minutes  
   - Purpose: Allow video generation to complete.

9. **Add HTTP Request Node to Get Video:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://api.piapi.ai/api/v1/task/{{task_id}} (from video generation)  
   - Headers: X-API-Key with Flux API key

10. **Add HTTP Request Node to Download Video Binary:**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: Video URL from previous node  
    - Response Format: File (binary)

11. **Add Read/Write File Node to Save Video Locally:**  
    - Type: Read/Write File  
    - Operation: Write  
    - File Name: VideoBackground.mp4  
    - Input: Binary video data

12. **Add Google Sheets Node to Update Video URL:**  
    - Type: Google Sheets  
    - Operation: Update row matching "Index"  
    - Update "Background Video" column with video URL  
    - Credentials: Google Sheets OAuth2

13. **Add HTTP Request Node to Generate Audio:**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: https://api.elevenlabs.io/v1/sound-generation  
    - Headers: xi-api-key with ElevenLabs API key  
    - Body Parameters:  
      - text: Descriptive prompt including Background (EN) and Prompt (EN) fields  
      - duration_seconds: 5  
      - model_id: sound-effects-v1  
      - output_format: mp3

14. **Add Google Drive Node to Upload Audio:**  
    - Type: Google Drive  
    - Operation: Upload file  
    - File Name: Based on Background (EN) field with .mp3 extension  
    - Drive ID and Folder ID: Your Google Drive setup  
    - Credentials: Google Drive OAuth2

15. **Add Read/Write File Node to Save Audio Locally:**  
    - Type: Read/Write File  
    - Operation: Write  
    - File Name: SoundBackground.mp3  
    - Input: Audio binary data

16. **Add Google Sheets Node to Update Audio URL:**  
    - Type: Google Sheets  
    - Operation: Update row matching "Index"  
    - Update "Music Background" column with Google Drive file URL  
    - Credentials: Google Sheets OAuth2

17. **Add Code Node to Prepare Overlay Text:**  
    - Type: Code (JavaScript)  
    - Purpose: Generate FFmpeg drawtext filter string for quote and author text  
    - Inputs: Quote (Thai) and Pen Name (Thai) from Google Sheet  
    - Outputs: JSON with drawText filter string  
    - Use Kanit-Italic.ttf font, font sizes 70 (quote) and 50 (author), white color, centered alignment

18. **Add Execute Command Node to Run FFmpeg:**  
    - Type: Execute Command  
    - Command:  
      ```
      ffmpeg -i VideoBackground.mp4 -i SoundBackground.mp3 -filter_complex "[0:v]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920[vid]; color=black@0.3:size=1080x1920:d=10[bg]; [vid][bg]overlay=shortest=1[bgvid]; [bgvid]{{drawText}}[outv]; [1:a]volume=0.8[aout]" -map "[outv]" -map "[aout]" -aspect 9:16 -c:v libx264 -c:a aac -shortest output.mp4 -y
      ```
    - Replace `{{drawText}}` with output from previous code node  
    - Requirements: FFmpeg installed with Kanit font available

19. **Add HTTP Request Node to Initiate YouTube Upload:**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: https://www.googleapis.com/upload/youtube/v3/videos?part=snippet,status&uploadType=resumable  
    - Headers: Content-Type application/json, X-Upload-Content-Type video/webm  
    - Body: JSON with snippet (title and description from quote and author), status (public, embeddable)  
    - Authentication: YouTube OAuth2

20. **Add Read/Write File Node to Read Final Video:**  
    - Type: Read/Write File  
    - Operation: Read  
    - File Name: output.mp4

21. **Add HTTP Request Node to Upload Video to YouTube:**  
    - Type: HTTP Request  
    - Method: PUT  
    - URL: Upload URL from previous node’s response header  
    - Headers: Content-Type video/webm  
    - Authentication: YouTube OAuth2  
    - Input Data Field: Binary data from Read output file

22. **Add Google Sheets Node to Update YouTube URL:**  
    - Type: Google Sheets  
    - Operation: Append or Update row matching "Index"  
    - Update "Video Status" column with YouTube video URL constructed from returned video ID  
    - Credentials: Google Sheets OAuth2

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires a self-hosted n8n instance with FFmpeg installed for video processing.   | FFmpeg is not supported on n8n Cloud; install on your self-hosted environment.                      |
| Use Kanit font (Kanit-Italic.ttf) for Thai language text overlay in FFmpeg.                      | Ensure the font is installed and accessible by FFmpeg on your server.                              |
| Google Sheets template for quotes management is provided: [Google Sheets Template](https://docs.google.com/spreadsheets/d/1p1iPoiu2uI3qGbHi0diS7QwsMcLuzDIqwo3AeSUVrGQ/edit?usp=sharing) | Use this template to prepare your quotes, authors, and prompts.                                   |
| API keys required: Flux AI, Kling API, ElevenLabs, Google Sheets, Google Drive, YouTube OAuth2. | Authenticate all APIs in n8n credentials before running the workflow.                              |
| Workflow is tailored for Thai language but can be adapted by changing fonts and prompts.        | Adjust FFmpeg font settings and ElevenLabs prompt accordingly.                                    |
| The workflow demonstrates advanced integration of AI-generated media and automated publishing.  | Ideal for content creators targeting social media platforms with vertical video formats.          |

---

This structured documentation provides a complete understanding of the workflow’s architecture, node configurations, and stepwise reproduction instructions, enabling advanced users and automation agents to maintain, customize, or extend the workflow confidently.