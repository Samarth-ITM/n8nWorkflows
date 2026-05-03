Extend and Merge UGC Viral Videos Using Kling 2.1, Then Publish on Social Media

https://n8nworkflows.xyz/workflows/extend-and-merge-ugc-viral-videos-using-kling-2-1--then-publish-on-social-media-11710


# Extend and Merge UGC Viral Videos Using Kling 2.1, Then Publish on Social Media

### 1. Workflow Overview

This n8n workflow, titled **"Extend and Merge UGC Viral Videos Using Kling 2.1, Then Publish on Social Media"**, automates the end-to-end process of extending short User-Generated Content (UGC) viral videos using AI, merging these extended clips with originals, and publishing the final outputs onto cloud storage and multiple social media platforms.

**Primary Use Cases:**
- Content creators and social media marketers looking to automate video extension and distribution.
- Agencies managing UGC video campaigns requiring AI-driven content generation and bulk publishing.
- Automation enthusiasts integrating multiple APIs for seamless video workflows.

**Logical Blocks:**

- **1.1 Input Reception and Parameter Setup:** Fetch input video data and parameters from a Google Sheet to initiate processing.
- **1.2 Extract Last Frame:** Retrieve the last frame of the original video using Fal AI’s FFmpeg API.
- **1.3 Video Extension via AI (Kling 2.1 Model):** Generate extended video clips based on the extracted frame and prompt using RunPod’s Kling 2.1 AI model.
- **1.4 Video Merge Preparation and Execution:** Collect video URLs (original + extended), merge them into a seamless video with Fal AI’s merge API.
- **1.5 Upload and Publish:** Upload the final merged video to Google Drive and publish it on social media platforms via Upload-Post (YouTube) and Postiz (TikTok, Instagram, Facebook, X).
- **1.6 Status Monitoring and Sheet Updates:** Continuously monitor asynchronous API job statuses, update Google Sheets with progress and final URLs.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Parameter Setup

**Overview:**  
This block initiates the workflow by manually triggering it, fetching video data and prompts from Google Sheets, and preparing parameters for further processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get video  
- Set params  
- Sticky Note5  
- Sticky Note1  
- Sticky Note (Setup instructions for Fal AI API key)  
- Sticky Note5 (Workflow description)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts workflow manually.  
  - *Config:* No parameters; user-triggered.  
  - *Connections:* Outputs to `Get video`.  
  - *Failures:* None anticipated.

- **Get video**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves video URLs and metadata from a Google Sheet.  
  - *Config:* Reads sheet with columns including `START`, `PROMPT`, `DURATION`, `VIDEO URL`, `MERGE`, `row_number`. Filters on `VIDEO URL`.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Connections:* Outputs to `Set params`.  
  - *Failures:* Possible auth errors, sheet access issues, or empty rows.

- **Set params**  
  - *Type:* Set  
  - *Role:* Assigns workflow variables from input JSON: `prompt`, `duration`, and `video`.  
  - *Config:* Extracts from current item JSON fields `PROMPT`, `DURATION`, and `START`.  
  - *Connections:* Outputs to `Extract last frame`.  
  - *Failures:* Missing keys in input JSON may cause empty parameters.

- **Sticky Notes**  
  - Provide detailed setup instructions for users including API key configuration for Fal AI and RunPod, and workflow explanation.  
  - No functional impact but crucial for correct setup.

---

#### 2.2 Extract Last Frame

**Overview:**  
Extracts the last frame from the original video URL using Fal AI’s FFmpeg API to prepare the image for AI video extension.

**Nodes Involved:**  
- Extract last frame  
- Wait 10 sec.  
- Get status Extract Frame  
- Completed?1  
- Get Last frame  
- Sticky Note (Fal AI API key instructions)

**Node Details:**

- **Extract last frame**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST to Fal AI API to extract last frame image.  
  - *Config:* JSON body includes `video_url` from `Set params` node and `frame_type` = "last".  
  - *Credentials:* HTTP Header Auth with Fal.run API key.  
  - *Connections:* Output to `Wait 10 sec.`.  
  - *Failures:* HTTP errors, auth failures, invalid video URL, API rate limits.

- **Wait 10 sec.**  
  - *Type:* Wait  
  - *Role:* Pauses workflow 10 seconds to allow async extraction processing.  
  - *Config:* Amount = 10 seconds.  
  - *Connections:* Output to `Get status Extract Frame`.  
  - *Failures:* Minimal; only timing delays.

- **Get status Extract Frame**  
  - *Type:* HTTP Request  
  - *Role:* Polls Fal AI API for extraction job status using request ID from previous step.  
  - *Config:* Uses dynamic URL with `request_id`.  
  - *Credentials:* Fal.run API key.  
  - *Connections:* Output to `Completed?1`.  
  - *Failures:* Timeout, API errors, invalid request ID.

- **Completed?1**  
  - *Type:* If  
  - *Role:* Branches workflow based on status JSON field equals "COMPLETED".  
  - *Config:* Checks `$json.status == "COMPLETED"`.  
  - *Connections:* True → `Get Last frame`; False → loops back or wait.  
  - *Failures:* Expression errors if status field missing.

- **Get Last frame**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the extracted last frame image data from Fal AI API.  
  - *Config:* Uses request ID from extraction response.  
  - *Credentials:* Fal.run API key.  
  - *Connections:* Outputs to `Generate clip`.  
  - *Failures:* HTTP errors, auth failure.

---

#### 2.3 Video Extension via AI (Kling 2.1 Model)

**Overview:**  
Uses the extracted last frame and prompt to invoke RunPod’s Kling 2.1 model to generate an extended video clip matching the style and content prompt.

**Nodes Involved:**  
- Generate clip  
- Wait 60 sec.  
- Get status clip  
- Completed?  
- Sticky Note1

**Node Details:**

- **Generate clip**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to RunPod API to start Kling 2.1 video generation.  
  - *Config:* JSON body includes prompt (from Google Sheet), extracted image URL, duration, and other AI parameters like `guidance_scale`.  
  - *Credentials:* HTTP Bearer Auth with RunPod API token.  
  - *Connections:* Outputs to `Wait 60 sec.`.  
  - *Failures:* Auth failures, malformed prompt, API errors.

- **Wait 60 sec.**  
  - *Type:* Wait  
  - *Role:* Pauses workflow 60 seconds for video generation processing.  
  - *Connections:* Outputs to `Get status clip`.  
  - *Failures:* Minimal.

- **Get status clip**  
  - *Type:* HTTP Request  
  - *Role:* Polls RunPod API for the status of the video generation job.  
  - *Config:* Dynamic URL with job ID.  
  - *Credentials:* RunPod Bearer token.  
  - *Connections:* Outputs to `Completed?`.  
  - *Failures:* API errors, timeouts.

- **Completed?**  
  - *Type:* If  
  - *Role:* Checks if status equals "COMPLETED" to proceed.  
  - *Connections:* True → next steps; False → wait/retry.  
  - *Failures:* Expression errors if status field missing.

---

#### 2.4 Video Merge Preparation and Execution

**Overview:**  
Collects video URLs for merge (original and AI-generated clips) and submits a merge request to Fal AI’s API. Polls until merging is complete.

**Nodes Involved:**  
- Update video url  
- Get videos to merge  
- Set VideoUrls Json  
- Merge Videos  
- Wait 30 sec.  
- Get status  
- Completed?5  
- Get final video url  
- Update video url2  
- Get final video file  
- Sticky Note2  
- Sticky Note3  

**Node Details:**

- **Update video url**  
  - *Type:* Google Sheets  
  - *Role:* Updates Google Sheet row with the new video URL of the generated clip.  
  - *Config:* Matches rows by `row_number`. Updates `VIDEO URL` column.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Connections:* Outputs to `Get videos to merge`.  
  - *Failures:* Sheet access issues, write conflicts.

- **Get videos to merge**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves videos flagged for merging from the sheet (filter on `MERGE` column).  
  - *Credentials:* Google Sheets OAuth2.  
  - *Connections:* Outputs to `Set VideoUrls Json`.  
  - *Failures:* Same as above.

- **Set VideoUrls Json**  
  - *Type:* Code  
  - *Role:* Constructs JSON array of video URLs (original + generated) for merge API.  
  - *Config:* Creates JSON with `urls` array from `START` and `VIDEO URL` fields.  
  - *Connections:* Outputs to `Merge Videos`.  
  - *Failures:* Possible coding errors or missing URLs.

- **Merge Videos**  
  - *Type:* HTTP Request  
  - *Role:* POST request to Fal AI merge endpoint with video URL list and target framerate.  
  - *Config:* JSON body with `video_urls` array and `target_fps`=24.  
  - *Credentials:* Fal.run API key.  
  - *Connections:* Outputs to `Wait 30 sec.`.  
  - *Failures:* Auth, API limit, invalid URLs.

- **Wait 30 sec.**  
  - *Type:* Wait  
  - *Role:* Waits 30 seconds for merge job processing.  
  - *Connections:* Outputs to `Get status`.  
  - *Failures:* Minimal.

- **Get status**  
  - *Type:* HTTP Request  
  - *Role:* Polls merge job status from Fal AI API.  
  - *Config:* Uses merge job request ID.  
  - *Credentials:* Fal.run API key.  
  - *Connections:* Outputs to `Completed?5`.  
  - *Failures:* API errors.

- **Completed?5**  
  - *Type:* If  
  - *Role:* Checks if merge job status equals "COMPLETED".  
  - *Connections:* True → `Get final video url`; False → wait/retry.  
  - *Failures:* Expression errors.

- **Get final video url**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves final merged video URL from Fal AI API.  
  - *Config:* Uses merge job request ID.  
  - *Credentials:* Fal.run API key.  
  - *Connections:* Outputs to `Update video url2`.  
  - *Failures:* API errors.

- **Update video url2**  
  - *Type:* Google Sheets  
  - *Role:* Updates Google Sheet row with final merged video URL and marks `MERGE` column with "x".  
  - *Credentials:* Google Sheets OAuth2.  
  - *Connections:* Outputs to `Get final video file`.  
  - *Failures:* Sheet write errors.

- **Get final video file**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the final merged video file for upload steps.  
  - *Config:* URL from previous node's JSON path.  
  - *Connections:* Outputs to upload nodes.  
  - *Failures:* File not found, download errors.

- **Sticky Notes**  
  - Provide setup instructions for merge API key usage and video preparation.

---

#### 2.5 Upload and Publish

**Overview:**  
Uploads the final merged video to Google Drive, and publishes it to social media platforms via Upload-Post (YouTube) and Postiz (TikTok, Instagram, Facebook, X).

**Nodes Involved:**  
- Upload Video2  
- Upload to Youtube  
- Upload to Postiz  
- Upload to Social  
- Sticky Note6  
- Sticky Note7  
- Sticky Note8  

**Node Details:**

- **Upload Video2**  
  - *Type:* Google Drive  
  - *Role:* Uploads final video file to a specified Google Drive folder.  
  - *Config:* Dynamic filename with timestamp and original file name. Folder set by ID.  
  - *Credentials:* Google Drive OAuth2.  
  - *Connections:* None (end of upload chain).  
  - *Failures:* Auth errors, upload failure, file size limits.

- **Upload to Youtube**  
  - *Type:* HTTP Request  
  - *Role:* Uploads video to Upload-Post API for YouTube publishing.  
  - *Config:* Multipart-form data with parameters `title`, `user` (username), `platform[]` set to "youtube", and video as binary data.  
  - *Credentials:* HTTP Header Auth with Upload-Post API key.  
  - *Connections:* None.  
  - *Failures:* API errors, invalid credentials.

- **Upload to Postiz**  
  - *Type:* HTTP Request  
  - *Role:* Uploads video to Postiz API for multi-platform social publishing.  
  - *Config:* Multipart-form data with `file` binary data.  
  - *Credentials:* HTTP Header Auth with Postiz API key.  
  - *Connections:* Outputs to `Upload to Social`.  
  - *Failures:* API errors.

- **Upload to Social**  
  - *Type:* Postiz (n8n node)  
  - *Role:* Posts to configured social media channels via Postiz API with content and images.  
  - *Config:* Defines post date, content, integration ID, and short link option.  
  - *Credentials:* Postiz API OAuth2.  
  - *Connections:* None.  
  - *Failures:* Post errors, auth failures.

- **Sticky Notes:**  
  - Instructions for creating accounts on Upload-Post and Postiz, setting credentials, and uploading final videos.

---

#### 2.6 Status Monitoring and Sheet Updates

**Overview:**  
This cross-cutting block ensures the workflow waits for asynchronous API jobs (frame extraction, AI generation, merging) to complete and updates the Google Sheet accordingly.

**Nodes Involved:**  
- Completed?  
- Completed?1  
- Completed?5  
- Wait nodes (10 sec, 30 sec, 60 sec)  
- Get status nodes  
- Update nodes (Google Sheets)

**Node Details:**

- **Completed? / Completed?1 / Completed?5**  
  - *Type:* If nodes checking job completion status fields.  
  - *Config:* Check if `$json.status == "COMPLETED"`.  
  - *Role:* Control flow gating asynchronous operations.  
  - *Failures:* Expression errors if status missing or misformatted.

- **Wait nodes**  
  - Pauses for appropriate duration to allow external API processing.

- **Get status nodes**  
  - Poll external APIs for job statuses.

- **Update nodes**  
  - Log current progress and results to Google Sheets.

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                                    | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                       |
|---------------------------|----------------------------|---------------------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Starts workflow manually                           |                              | Get video                   |                                                                                                 |
| Get video                 | Google Sheets              | Fetch input video data and prompts from sheet     | When clicking ‘Execute workflow’ | Set params                  |                                                                                                 |
| Set params                | Set                        | Assigns key parameters for processing              | Get video                    | Extract last frame           |                                                                                                 |
| Extract last frame        | HTTP Request               | Requests last frame extraction from Fal AI API    | Set params                   | Wait 10 sec.                | ## STEP 2 - EXTRACT LAST FRAME (Fal AI API key instructions)                                    |
| Wait 10 sec.              | Wait                       | Waits 10 seconds for extraction to process         | Extract last frame           | Get status Extract Frame     |                                                                                                 |
| Get status Extract Frame  | HTTP Request               | Polls extraction job status                         | Wait 10 sec.                 | Completed?1                 |                                                                                                 |
| Completed?1               | If                         | Checks if extraction is completed                   | Get status Extract Frame     | Get Last frame / Wait 10 sec. |                                                                                                 |
| Get Last frame            | HTTP Request               | Retrieves extracted last frame image                | Completed?1 (true)           | Generate clip               |                                                                                                 |
| Generate clip             | HTTP Request               | Sends request to RunPod Kling 2.1 to generate clip | Get Last frame               | Wait 60 sec.                | ## STEP 3 - GENERATE EXTENDED VIDEO WITH KLING 2.1 (RunPod API key instructions)                |
| Wait 60 sec.              | Wait                       | Waits 60 seconds for AI generation                  | Generate clip                | Get status clip             |                                                                                                 |
| Get status clip           | HTTP Request               | Polls RunPod job status                             | Wait 60 sec.                 | Completed?                  |                                                                                                 |
| Completed?                | If                         | Checks if AI video generation is completed          | Get status clip              | Update video url / Wait 60 sec. |                                                                                                 |
| Update video url          | Google Sheets              | Updates sheet with AI-generated video URL           | Completed? (true)            | Get videos to merge          |                                                                                                 |
| Get videos to merge       | Google Sheets              | Retrieves videos to merge flagged in sheet          | Update video url             | Set VideoUrls Json           |                                                                                                 |
| Set VideoUrls Json        | Code                       | Constructs JSON list of videos to merge             | Get videos to merge          | Merge Videos                |                                                                                                 |
| Merge Videos             | HTTP Request               | Sends merge request to Fal AI API                    | Set VideoUrls Json           | Wait 30 sec.                | ## STEP 4 - MERGE VIDEOS (Fal AI API key instructions)                                         |
| Wait 30 sec.              | Wait                       | Waits 30 seconds for merge processing                | Merge Videos                | Get status                  |                                                                                                 |
| Get status                | HTTP Request               | Polls merge job status                               | Wait 30 sec.                | Completed?5                 |                                                                                                 |
| Completed?5               | If                         | Checks if merge is completed                          | Get status                  | Get final video url / Wait 30 sec. |                                                                                                 |
| Get final video url       | HTTP Request               | Retrieves final merged video URL                      | Completed?5 (true)           | Update video url2           |                                                                                                 |
| Update video url2         | Google Sheets              | Updates sheet with merged video URL                   | Get final video url          | Get final video file         |                                                                                                 |
| Get final video file      | HTTP Request               | Downloads final merged video file                      | Update video url2            | Upload Video2 / Upload to Youtube / Upload to Postiz |                                                                                                 |
| Upload Video2             | Google Drive               | Uploads final video file to Google Drive              | Get final video file         |                             | Upload final video to Google Drive                                                             |
| Upload to Youtube         | HTTP Request               | Uploads video to Upload-Post for YouTube publishing  | Get final video file         |                             | Create a FREE account on [Upload-Post](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) and Set YOUR_USERNAME and TITLE for Upload-Post |
| Upload to Postiz          | HTTP Request               | Uploads video to Postiz API for socials               | Get final video file         | Upload to Social            | Create a FREE account on [Postiz](https://postiz.com/?ref=n3witalia) and Set Channel_ID and TITLE for Postiz (TikTok, Instagram, Facebook, X, Youtube) |
| Upload to Social          | Postiz Node                | Publishes video on social media via Postiz           | Upload to Postiz             |                             |                                                                                                 |
| Sticky Note5              | Sticky Note                | Workflow description and overview                      |                              |                             | Workflow automates extending, merging, and social publishing of UGC videos using AI models.    |
| Sticky Note               | Sticky Note                | Fal AI Extraction API key instructions                 |                              |                             | ## STEP 2 - EXTRACT LAST FRAME - Setup API key                                                |
| Sticky Note1              | Sticky Note                | RunPod Kling 2.1 API key instructions                  |                              |                             | ## STEP 3 - GENERATE EXTENDED VIDEO WITH KLING 2.1 - Setup API key                            |
| Sticky Note2              | Sticky Note                | Fal AI Merge API key instructions                       |                              |                             | ## STEP 4 - MERGE VIDEOS - Setup API key                                                     |
| Sticky Note3              | Sticky Note                | Video merge preparation note                            |                              |                             | ## PREPARE VIDEOS TO MERGE                                                                   |
| Sticky Note6              | Sticky Note                | Upload-Post account and settings instructions          |                              |                             | Create a FREE account on Upload-Post and set credentials and titles                          |
| Sticky Note7              | Sticky Note                | Postiz account and social channel setup instructions   |                              |                             | Create a FREE account on Postiz and set channel ID and titles                               |
| Sticky Note8              | Sticky Note                | Google Drive upload reminder                            |                              |                             | Upload final video to Google Drive                                                          |
| Sticky Note4              | Sticky Note                | Upload and social media publishing block note          |                              |                             | ## STEP 5 - UPLOAD TO GOOGLE DRIVE AND SOCIAL MEDIA                                         |
| Sticky Note5              | Sticky Note                | Setup Google Sheet instructions                         |                              |                             | ## STEP 1 - SET PARAMS - Clone and fill Google Sheet with START, PROMPT, DURATION           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Execute workflow’` with default settings.

2. **Add a Google Sheets node** named `Get video`:
   - Set operation to "Read Rows".
   - Configure with your Google Sheets OAuth2 credentials.
   - Select the document ID for your spreadsheet.
   - Choose the sheet (gid=0).
   - Set a filter on the `VIDEO URL` column to fetch appropriate rows.
   - Connect `When clicking ‘Execute workflow’` → `Get video`.

3. **Add a Set node** named `Set params`:
   - Assign three variables from input JSON:
     - `prompt` = `{{$json.PROMPT}}`
     - `duration` = `{{$json.DURATION}}`
     - `video` = `{{$json.START}}`
   - Connect `Get video` → `Set params`.

4. **Add an HTTP Request node** named `Extract last frame`:
   - Method: POST
   - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/extract-frame`
   - Body (JSON):
     ```json
     {
       "video_url": "{{ $json.video }}",
       "frame_type": "last"
     }
     ```
   - Set "Send Body" to true, "Specify Body" as JSON.
   - Authentication: HTTP Header Auth; header `Content-Type: application/json`.
   - Use your Fal.run API key for header `Authorization: Key YOURAPIKEY`.
   - Connect `Set params` → `Extract last frame`.

5. **Add a Wait node** named `Wait 10 sec.`:
   - Amount: 10 seconds.
   - Connect `Extract last frame` → `Wait 10 sec.`.

6. **Add an HTTP Request node** named `Get status Extract Frame`:
   - Method: GET
   - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}/status`
   - Authentication: Fal.run API key as before.
   - Connect `Wait 10 sec.` → `Get status Extract Frame`.

7. **Add an If node** named `Completed?1`:
   - Condition: Check if `{{$json.status}} == "COMPLETED"`.
   - Connect `Get status Extract Frame` → `Completed?1`.

8. **Add an HTTP Request node** named `Get Last frame`:
   - Method: GET
   - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
   - Authentication: Fal.run API key.
   - Connect `Completed?1` (true branch) → `Get Last frame`.

9. **Add an HTTP Request node** named `Generate clip`:
   - Method: POST
   - URL: `https://api.runpod.ai/v2/kling-v2-1-i2v-pro/run`
   - Body (JSON):
     ```json
     {
       "input": {
         "prompt": "{{ $('Get prompts').item.json.PROMPT }}",
         "image": "{{ $json.images[0].url }}",
         "negative_prompt": "",
         "guidance_scale": 0.5,
         "duration": {{ $('Get prompts').item.json.DURATION }},
         "enable_safety_checker": true
       }
     }
     ```
   - Authentication: Bearer token with RunPod API key.
   - Connect `Get Last frame` → `Generate clip`.

10. **Add a Wait node** named `Wait 60 sec.`:
    - Amount: 60 seconds.
    - Connect `Generate clip` → `Wait 60 sec.`.

11. **Add an HTTP Request node** named `Get status clip`:
    - Method: GET
    - URL: `https://api.runpod.ai/v2/kling-v2-1-i2v-pro/status/{{ $json.id }}`
    - Authentication: RunPod Bearer token.
    - Connect `Wait 60 sec.` → `Get status clip`.

12. **Add an If node** named `Completed?`:
    - Condition: Check if `{{$json.status}} == "COMPLETED"`.
    - Connect `Get status clip` → `Completed?`.

13. **Add a Google Sheets node** named `Update video url`:
    - Operation: Update Row.
    - Map `VIDEO URL` to `{{$json.output.result}}`.
    - Match by `row_number` from `Get prompts`.
    - Use Google Sheets OAuth2 credentials.
    - Connect `Completed?` (true branch) → `Update video url`.

14. **Add a Google Sheets node** named `Get videos to merge`:
    - Operation: Read Rows.
    - Filter by `MERGE` column.
    - Use same sheet and document.
    - Connect `Update video url` → `Get videos to merge`.

15. **Add a Code node** named `Set VideoUrls Json`:
    - Code:
      ```js
      const item = items[0].json;
      return [{
        json: { urls: [item.START, item["VIDEO URL"]] }
      }];
      ```
    - Connect `Get videos to merge` → `Set VideoUrls Json`.

16. **Add an HTTP Request node** named `Merge Videos`:
    - Method: POST
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`
    - Body (JSON):
      ```json
      {
        "video_urls": {{ JSON.stringify($json.urls) }},
        "target_fps": 24
      }
      ```
    - Authentication: Fal.run API key.
    - Connect `Set VideoUrls Json` → `Merge Videos`.

17. **Add a Wait node** named `Wait 30 sec.`:
    - Amount: 30 seconds.
    - Connect `Merge Videos` → `Wait 30 sec.`.

18. **Add an HTTP Request node** named `Get status`:
    - Method: GET
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $('Merge Videos').item.json.request_id }}/status`
    - Authentication: Fal.run API key.
    - Connect `Wait 30 sec.` → `Get status`.

19. **Add an If node** named `Completed?5`:
    - Condition: Check if `{{$json.status}} == "COMPLETED"`.
    - Connect `Get status` → `Completed?5`.

20. **Add an HTTP Request node** named `Get final video url`:
    - Method: GET
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
    - Authentication: Fal.run API key.
    - Connect `Completed?5` (true) → `Get final video url`.

21. **Add a Google Sheets node** named `Update video url2`:
    - Operation: Update Row.
    - Set `MERGE` = "x", clear `VIDEO URL`, set `FINAL VIDEO` = `{{$json.video.url}}`.
    - Match by `row_number`.
    - Connect `Get final video url` → `Update video url2`.

22. **Add an HTTP Request node** named `Get final video file`:
    - Method: GET
    - URL: `{{$json.video.url}}` (from previous node).
    - Connect `Update video url2` → `Get final video file`.

23. **Add a Google Drive node** named `Upload Video2`:
    - Upload the final video file to a specific Google Drive folder.
    - Use Google Drive OAuth2 credentials.
    - Connect `Get final video file` → `Upload Video2`.

24. **Add an HTTP Request node** named `Upload to Youtube`:
    - POST to `https://api.upload-post.com/api/upload`.
    - Multipart form data with parameters for `title`, `user`, `platform[]` = "youtube", and video binary.
    - Use HTTP Header Auth for Upload-Post API key.
    - Connect `Get final video file` → `Upload to Youtube`.

25. **Add an HTTP Request node** named `Upload to Postiz`:
    - POST to `https://api.postiz.com/public/v1/upload`.
    - Multipart form data with binary file.
    - Use HTTP Header Auth for Postiz API key.
    - Connect `Get final video file` → `Upload to Postiz`.

26. **Add a Postiz node** named `Upload to Social`:
    - Posts video content with date, content, integration ID, and short link.
    - Use Postiz OAuth2 credentials.
    - Connect `Upload to Postiz` → `Upload to Social`.

27. **Add Sticky Notes** throughout the workflow to provide setup instructions for API keys and usage guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow automates extending, merging, and publishing UGC videos using AI. Integrates Fal AI, RunPod/Kling 2.1, Upload-Post, Postiz, Google Sheets, and Google Drive.                                                                                                                                      | Workflow description (Sticky Note5)                                                                              |
| Create an account on [Fal AI](https://fal.ai/) and obtain API key for frame extraction and video merging.                                                                                                                                                                                                | Fal AI API key setup (Sticky Note)                                                                               |
| Create an account on [RunPod](https://runpod.io?ref=moqjqsgw), obtain API key, and set Bearer token for Kling 2.1 video generation.                                                                                                                                                                      | RunPod API key setup (Sticky Note1)                                                                              |
| Create a free account on [Upload-Post](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app), set username and post title for YouTube uploads.                                                                                                                       | Upload-Post account setup (Sticky Note6)                                                                         |
| Create a free account on [Postiz](https://postiz.com/?ref=n3witalia), set Channel_ID and post title for TikTok, Instagram, Facebook, X, and YouTube publishing.                                                                                                                                           | Postiz account setup (Sticky Note7)                                                                               |
| Clone the provided Google Sheet template to prepare input data columns `START`, `PROMPT`, and `DURATION` (seconds, typically 5 or 10 for Kling 2.1).                                                                                                                                                       | Google Sheets setup (Sticky Note5)                                                                                |
| Upload final merged videos to Google Drive for archival or manual download.                                                                                                                                                                                                                              | Upload to Google Drive reminder (Sticky Note8)                                                                    |

---

This structured reference provides a comprehensive understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.