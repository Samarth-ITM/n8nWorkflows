Transform Old Photos into Animated Videos with FLUX & Kling AI for Social Media

https://n8nworkflows.xyz/workflows/transform-old-photos-into-animated-videos-with-flux---kling-ai-for-social-media-5755


# Transform Old Photos into Animated Videos with FLUX & Kling AI for Social Media

### 1. Workflow Overview

This workflow automates the transformation of old photos into colorized and animated videos, then automatically publishes the results on multiple social media platforms. It targets users who want to breathe new life into vintage or black-and-white photographs by enhancing and animating them with AI, followed by easy distribution.

The workflow consists of these main logical blocks:

- **1.1 Input Reception:** Users upload an old photo and optionally provide a custom animation description through a web form.
- **1.2 Image Upload & Colorization:** The uploaded photo is sent to an image hosting service (imgbb), then colorized and enhanced using the FLUX Kontext AI service.
- **1.3 Colorization Status Polling:** The workflow periodically checks if the colorization process is completed.
- **1.4 Animation Request & Polling:** Once colorization is done, the enhanced image is animated with Kling Video AI, followed by status checks until animation is finished.
- **1.5 Video Download & Storage:** The resulting video is downloaded and saved to Google Drive.
- **1.6 Finalization & Social Upload:** The workflow compiles result URLs and metadata, then uploads the video to multiple social media platforms (Facebook, Instagram Reels, YouTube, and X).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception Block

- **Overview:**  
  Captures user input via a web form, including the old photo file and an optional custom animation description.

- **Nodes Involved:**  
  - Photo Upload Form

- **Node Details:**  
  - **Photo Upload Form**  
    - Type: Form Trigger  
    - Role: Receives photo file and optional text input from user via HTTP webhook.  
    - Configuration: Path `animate-photo-form` with fields: file upload (required) and textarea for animation description (optional).  
    - Inputs: HTTP request (user form submission)  
    - Outputs: JSON containing uploaded file binary data and description string  
    - Edge Cases: Missing file upload (form enforces required), empty or malformed animation description  
    - Version: 2.1  

#### 2.2 Image Upload & Colorization Block

- **Overview:**  
  Uploads the photo binary to imgbb, then sends the hosted image URL to FLUX Kontext API for colorization and enhancement.

- **Nodes Involved:**  
  - Upload Image to imgbb  
  - Colorize Image

- **Node Details:**  
  - **Upload Image to imgbb**  
    - Type: HTTP Request  
    - Role: Uploads binary photo data to imgbb image hosting service using multipart form-data.  
    - Configuration: POST to `https://api.imgbb.com/1/upload` with form-data including image binary and API key (must be set by user).  
    - Input: Binary data from Photo Upload Form  
    - Output: JSON with hosted image URL (`data.url`)  
    - Edge Cases: API key invalid or expired, upload failure, network timeouts  
    - Version: 4.2  
  - **Colorize Image**  
    - Type: HTTP Request  
    - Role: Sends hosted image URL to FLUX Kontext API to colorize and enhance the photo.  
    - Configuration: POST JSON body with a detailed prompt requesting natural colorization and enhancement; uses HTTP Header authentication via credentials.  
    - Inputs: Hosted image URL from imgbb response  
    - Outputs: JSON including a `request_id` used to track colorization progress  
    - Edge Cases: Authentication failure, API downtime, malformed prompt, invalid image URL  
    - Version: 4.2  
    - Credentials: fal.ai HTTP Header Auth (user must configure)  

#### 2.3 Colorization Status Polling Block

- **Overview:**  
  Waits and polls the FLUX Kontext API until the colorization job completes.

- **Nodes Involved:**  
  - Wait for Colorization  
  - Check Colorization Status  
  - Colorization Completed?  
  - Get Colorized Image

- **Node Details:**  
  - **Wait for Colorization**  
    - Type: Wait  
    - Role: Pauses workflow until webhook triggered or timer expires (no explicit timer set, waits for external webhook call).  
    - Inputs: Triggered after Colorize Image node  
    - Outputs: Triggers status check  
    - Edge Cases: Webhook not triggered, indefinite wait  
    - Version: 1.1  
  - **Check Colorization Status**  
    - Type: HTTP Request  
    - Role: Queries FLUX Kontext API for current status of colorization using `request_id`.  
    - Configuration: GET request to URL containing `request_id` with HTTP Header Auth.  
    - Inputs: Triggered by Wait node  
    - Outputs: JSON status response  
    - Edge Cases: API errors, status not changing, authentication failure  
    - Version: 4.2  
  - **Colorization Completed?**  
    - Type: If  
    - Role: Checks if colorization status equals "COMPLETED".  
    - Inputs: Status JSON from Check Colorization Status  
    - Outputs: Branches workflow to either proceed or wait again  
    - Edge Cases: Status other than "COMPLETED", unexpected status format  
    - Version: 2.2  
  - **Get Colorized Image**  
    - Type: HTTP Request  
    - Role: Fetches final colorized image details once completed.  
    - Configuration: GET request with `request_id`, expects JSON with image URLs.  
    - Inputs: If node positive branch  
    - Outputs: Colorized image URL for animation  
    - Edge Cases: API failure, missing image URL  
    - Version: 4.2  

#### 2.4 Animation Request & Polling Block

- **Overview:**  
  Sends the colorized image to Kling Video AI for animation, then polls until the animation completes.

- **Nodes Involved:**  
  - Animate Image  
  - Wait for Animation  
  - Check Animation Status  
  - Animation Completed?  
  - Get Animated Video

- **Node Details:**  
  - **Animate Image**  
    - Type: HTTP Request  
    - Role: Requests video animation for the colorized image with optional user prompt or default animation instructions.  
    - Configuration: POST JSON body with prompt (from form or default), image URL, duration = 5 seconds, quality = standard; HTTP Header Auth.  
    - Inputs: Colorized image URL from Get Colorized Image node  
    - Outputs: JSON with `request_id` for animation job  
    - Edge Cases: Invalid prompt, API errors, authentication issues  
    - Version: 4.2  
  - **Wait for Animation**  
    - Type: Wait  
    - Role: Pauses workflow for 60 seconds before polling animation status.  
    - Inputs: After Animate Image  
    - Outputs: Triggers Check Animation Status  
    - Edge Cases: Insufficient wait time causing premature polling  
    - Version: 1.1  
  - **Check Animation Status**  
    - Type: HTTP Request  
    - Role: Polls Kling Video API for animation job status using `request_id`.  
    - Inputs: After Wait for Animation  
    - Outputs: Status JSON  
    - Edge Cases: API errors, authentication failure, status stuck or delayed  
    - Version: 4.2  
  - **Animation Completed?**  
    - Type: If  
    - Role: Checks if animation status equals "COMPLETED".  
    - Inputs: Status JSON  
    - Outputs: Proceeds if completed or loops back to wait  
    - Edge Cases: Unexpected status states  
    - Version: 2.2  
  - **Get Animated Video**  
    - Type: HTTP Request  
    - Role: Retrieves final animated video metadata including download URL.  
    - Inputs: If positive branch of Animation Completed?  
    - Outputs: JSON with video URL  
    - Edge Cases: API failure, missing video URL  
    - Version: 4.2  

#### 2.5 Video Download & Storage Block

- **Overview:**  
  Downloads the animated video file and uploads it to Google Drive for persistent storage and access.

- **Nodes Involved:**  
  - Download Video  
  - Upload to Google Drive

- **Node Details:**  
  - **Download Video**  
    - Type: HTTP Request  
    - Role: Downloads video file binary from URL retrieved in Get Animated Video.  
    - Inputs: Video URL JSON  
    - Outputs: Binary video data  
    - Edge Cases: Download failures, broken URL, network timeouts  
    - Version: 4.2  
  - **Upload to Google Drive**  
    - Type: Google Drive  
    - Role: Uploads video binary to user’s Google Drive root folder with timestamped filename.  
    - Configuration: Uses OAuth2 credentials, names file `animated-photo-YYYYMMDD-HHmmss.mp4`  
    - Inputs: Binary video data from Download Video  
    - Outputs: JSON including Google Drive webViewLink URL  
    - Edge Cases: Auth token expiration, quota limits, file size restrictions  
    - Version: 3  

#### 2.6 Finalization & Social Upload Block

- **Overview:**  
  Sets final success message and relevant URLs, then uploads the video to multiple social media platforms.

- **Nodes Involved:**  
  - Final Results  
  - Upload Post

- **Node Details:**  
  - **Final Results**  
    - Type: Set  
    - Role: Packages success flag, message, colorized image URL, animated video URL, Google Drive link, and timestamp into output JSON.  
    - Inputs: Output from Upload to Google Drive  
    - Outputs: JSON result summary  
    - Edge Cases: Missing or incomplete data from previous nodes  
    - Version: 3.4  
  - **Upload Post**  
    - Type: Upload Post (custom node)  
    - Role: Publishes the animated video on Instagram (as Reels), Facebook, YouTube, and X (Twitter) using the Upload Post API.  
    - Configuration: User must provide upload post API user credentials; video URL taken from Get Animated Video node JSON.  
    - Inputs: JSON with video URL  
    - Outputs: Upload API responses  
    - Edge Cases: Platform API rate limits, invalid credentials, video format restrictions  
    - Version: 1  
    - Credentials: uploadPostApi OAuth or token-based auth (user configured)  

---

### 3. Summary Table

| Node Name             | Node Type                     | Functional Role                          | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                             |
|-----------------------|-------------------------------|----------------------------------------|----------------------------|---------------------------|-------------------------------------------------------------------------------------------------------|
| Photo Upload Form      | Form Trigger                  | Receive photo and animation description| HTTP (user)                | Upload Image to imgbb     |                                                                                                       |
| Upload Image to imgbb  | HTTP Request                  | Upload photo binary to imgbb            | Photo Upload Form          | Colorize Image            | "Setup Required: api.imgbb.com free token needed"                                                     |
| Colorize Image         | HTTP Request                  | Send image URL to FLUX Kontext for colorization | Upload Image to imgbb   | Wait for Colorization     | "Setup Required: FAL.AI API Key from fal.ai"                                                          |
| Wait for Colorization  | Wait                         | Pause until triggered or webhook call  | Colorize Image             | Check Colorization Status |                                                                                                       |
| Check Colorization Status | HTTP Request               | Poll FLUX Kontext for colorization status | Wait for Colorization      | Colorization Completed?   |                                                                                                       |
| Colorization Completed?| If                           | Branch if colorization done             | Check Colorization Status  | Get Colorized Image / Wait for Colorization |                                                                                                       |
| Get Colorized Image    | HTTP Request                  | Retrieve colorized image URL            | Colorization Completed?    | Animate Image             |                                                                                                       |
| Animate Image          | HTTP Request                  | Request animation video creation        | Get Colorized Image        | Wait for Animation        | "Setup Required: FAL.AI API Key from fal.ai"                                                          |
| Wait for Animation     | Wait                         | Pause 60 seconds before polling animation status | Animate Image          | Check Animation Status    |                                                                                                       |
| Check Animation Status | HTTP Request                  | Poll Kling Video AI for animation status | Wait for Animation         | Animation Completed?      |                                                                                                       |
| Animation Completed?   | If                           | Branch if animation done                | Check Animation Status     | Get Animated Video / Wait for Animation |                                                                                                       |
| Get Animated Video     | HTTP Request                  | Retrieve animated video URL             | Animation Completed?       | Download Video            |                                                                                                       |
| Download Video         | HTTP Request                  | Download animated video binary          | Get Animated Video         | Upload to Google Drive    |                                                                                                       |
| Upload to Google Drive | Google Drive                  | Upload video to Google Drive             | Download Video             | Final Results             | "Setup Required: Connect Google Drive account"                                                        |
| Final Results          | Set                          | Compile success message and URLs         | Upload to Google Drive     | Upload Post               |                                                                                                       |
| Upload Post            | Upload Post (custom node)     | Publish video on Facebook, Instagram, YouTube, X | Final Results           |                           | "Setup Required: Configure Upload Post API credentials"                                              |
| Workflow Description   | Sticky Note                  | Overall workflow purpose and summary     | -                          | -                         | "# Colorize and Animate Old Photos with AI ..."                                                       |
| Usage Instructions     | Sticky Note                  | User instructions on how to use workflow | -                          | -                         | "How to Use: Upload photo, get colorized & animated video, saved to Google Drive"                     |
| Setup Instructions     | Sticky Note                  | Setup prerequisites and API details      | -                          | -                         | "Setup Required: imgbb token, FAL.AI key, Google Drive account, API endpoints used"                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Photo Upload Form"**  
   - Type: Form Trigger  
   - Webhook Path: `animate-photo-form`  
   - Form Title: "Colorize and Animate Old Photos"  
   - Form Fields:  
     - File upload field labeled "Upload Old Photo" (required)  
     - Textarea labeled "Custom Animation Description (Optional)" with placeholder text  
   - Save and activate webhook.

2. **Add HTTP Request Node: "Upload Image to imgbb"**  
   - Connect from "Photo Upload Form" main output  
   - Method: POST  
   - URL: `https://api.imgbb.com/1/upload`  
   - Content Type: multipart-form-data  
   - Body Parameters:  
     - `image`: binary data from uploaded photo file field  
     - `key`: your personal imgbb API token (must be generated and inserted here)  
   - Save.

3. **Add HTTP Request Node: "Colorize Image"**  
   - Connect from "Upload Image to imgbb" output  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/flux-pro/kontext`  
   - Authentication: HTTP Header Auth (configure with your fal.ai API key)  
   - Headers: `Content-Type: application/json`  
   - JSON Body:  
     ```json
     {
       "prompt": "Colorize this old black and white or sepia photograph. Make it look natural and vibrant with realistic colors. Remove any noise, scratches, or artifacts. Enhance the image quality and make it crystal clear. Keep all the original details and people exactly as they are, just add beautiful, realistic colors and improve the overall quality",
       "image_url": "{{ $json.data.url }}",
       "num_images": 1
     }
     ```
   - Save.

4. **Add Wait Node: "Wait for Colorization"**  
   - Connect from "Colorize Image" output  
   - Configure as webhook wait node (no timer) — waits for external webhook trigger indicating colorization progress  
   - Save.

5. **Add HTTP Request Node: "Check Colorization Status"**  
   - Connect from "Wait for Colorization" output  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/flux-pro/requests/{{ $('Colorize Image').item.json.request_id }}/status`  
   - Authentication: same as "Colorize Image" node  
   - Save.

6. **Add If Node: "Colorization Completed?"**  
   - Connect from "Check Colorization Status" output  
   - Condition: Check if `status` field equals "COMPLETED" (case-sensitive, strict)  
   - Save.

7. **Add HTTP Request Node: "Get Colorized Image"**  
   - Connect from If node TRUE branch  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/flux-pro/requests/{{ $('Colorize Image').item.json.request_id }}`  
   - Authentication: same as above  
   - Save.

8. **Connect If node FALSE branch back to "Wait for Colorization"**  
   - To retry polling until completion.

9. **Add HTTP Request Node: "Animate Image"**  
   - Connect from "Get Colorized Image" output  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/kling-video/v2.1/standard/image-to-video`  
   - Authentication: HTTP Header Auth with fal.ai API key  
   - Headers: `Content-Type: application/json`  
   - JSON Body:  
     ```json
     {
       "prompt": "{{ $('Photo Upload Form').item.json['Custom Animation Description (Optional)'] ? $('Photo Upload Form').item.json['Custom Animation Description (Optional)'] : 'Animate this colorized vintage photograph. If there are multiple people, make them interact naturally - perhaps talking, smiling, or making subtle gestures. If there\\'s only one person, animate them with natural movements like breathing, blinking, slight head movements, or gentle smiling. Keep the animation subtle and realistic, maintaining the vintage charm. The movement should feel natural and bring the photo to life. Do not move the camera' }}",
       "image_url": "{{ $('Get Colorized Image').item.json.images[0].url }}",
       "duration": 5,
       "quality": "standard"
     }
     ```
   - Save.

10. **Add Wait Node: "Wait for Animation"**  
    - Connect from "Animate Image" output  
    - Wait for 60 seconds before next action  
    - Save.

11. **Add HTTP Request Node: "Check Animation Status"**  
    - Connect from "Wait for Animation" output  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/kling-video/requests/{{ $('Animate Image').item.json.request_id }}/status`  
    - Authentication: same fal.ai credentials  
    - Save.

12. **Add If Node: "Animation Completed?"**  
    - Connect from "Check Animation Status" output  
    - Condition: `status` equals "COMPLETED" (strict, case-sensitive)  
    - Save.

13. **Add HTTP Request Node: "Get Animated Video"**  
    - Connect from If node TRUE branch  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/kling-video/requests/{{ $('Animate Image').item.json.request_id }}`  
    - Authentication: fal.ai  
    - Save.

14. **Connect If node FALSE branch back to "Wait for Animation"**  
    - To retry polling.

15. **Add HTTP Request Node: "Download Video"**  
    - Connect from "Get Animated Video" output  
    - Method: GET  
    - URL: `={{ $('Get Animated Video').item.json.video.url }}` (dynamic)  
    - Save.

16. **Add Google Drive Node: "Upload to Google Drive"**  
    - Connect from "Download Video" output  
    - Operation: Upload File  
    - File Name: `animated-photo-{{ $now.format('yyyyMMdd-HHmmss') }}.mp4`  
    - Folder: root or user preferred folder  
    - Credentials: Connect Google Drive OAuth2 API  
    - Save.

17. **Add Set Node: "Final Results"**  
    - Connect from "Upload to Google Drive" output  
    - Assign fields:  
      - `success`: true (boolean)  
      - `message`: "Your old photo has been successfully colorized and animated!" (string)  
      - `colorized_image_url`: `={{ $('Get Colorized Image').item.json.images[0].url }}`  
      - `animated_video_url`: `={{ $('Get Animated Video').item.json.video.url }}`  
      - `google_drive_url`: `={{ $('Upload to Google Drive').item.json.webViewLink }}`  
      - `processing_time`: current timestamp formatted as `yyyy-MM-dd HH:mm:ss`  
    - Save.

18. **Add Upload Post Node: "Upload Post"**  
    - Connect from "Final Results" output  
    - Configure:  
      - User account for upload-post API (must be set up by user)  
      - Title: "This video was uploaded with the upload-post api"  
      - Video URL: `={{ $('Get Animated Video').item.json.video.url }}`  
      - Platforms: Instagram (Reels), Facebook, YouTube, X  
    - Credentials: Configure Upload Post API credentials  
    - Save.

19. **Add Sticky Notes for Documentation**  
    - Workflow Description, Usage Instructions, Setup Instructions with the content as provided in the original workflow for user clarity.

20. **Activate the workflow and test end-to-end by uploading an old photo and optionally providing an animation description.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow enables automatic colorization and animation of old photos with AI, then publishes videos to multiple social media platforms. | Workflow Overview sticky note                                                                            |
| Setup requires free imgbb API key, fal.ai API key, Google Drive OAuth2 credentials, and Upload Post API credentials.                    | Setup Instructions sticky note                                                                           |
| Usage instructions emphasize uploading a JPG or PNG photo and optionally providing a custom animation prompt.                          | Usage Instructions sticky note                                                                            |
| APIs used: imgbb (image upload), FLUX Kontext (colorization), Kling Video AI (animation), Google Drive (storage), Upload Post API (social upload) | Setup Instructions sticky note and technical description                                               |
| For best results, input images should be clear old photographs; custom animation prompts allow subtle, natural movement descriptions.    | Implied best practices from prompt examples in Animate Image node                                       |
| Video duration is fixed at 5 seconds with standard quality for animation.                                                               | Animate Image node configuration                                                                           |
| The workflow uses webhook-based waits and timed waits to handle asynchronous AI processing jobs robustly.                               | Wait nodes and polling loops                                                                             |
| Social media upload supports Instagram Reels, Facebook, YouTube, and X (Twitter) via Upload Post API.                                    | Upload Post node configuration                                                                            |

---

This documentation fully describes the workflow logic, node configurations, and integration details to facilitate understanding, reproduction, and modification by both human users and AI agents.