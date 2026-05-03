Transform Images into AI Videos with MiniMax Hailuo 02 & Upload to YouTube/TikTok

https://n8nworkflows.xyz/workflows/transform-images-into-ai-videos-with-minimax-hailuo-02---upload-to-youtube-tiktok-5633


# Transform Images into AI Videos with MiniMax Hailuo 02 & Upload to YouTube/TikTok

### 1. Workflow Overview

This workflow automates the process of transforming static images into short AI-generated videos using the MiniMax Hailuo 02 model, then uploads these videos to YouTube and TikTok, and finally updates a Google Sheet with the relevant video links and metadata. It is designed for content creators or marketers who want to efficiently produce and publish AI-enhanced videos from images and textual prompts.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initialization:** Receives new records from a Google Sheet containing images and prompts.
- **1.2 Video Generation Request:** Sends the image and prompt to the MiniMax Hailuo 02 API to create a video.
- **1.3 Video Generation Status Polling:** Waits and polls the API until video generation is complete.
- **1.4 Video Retrieval & Title Generation:** Retrieves the generated video URL and uses OpenAI GPT to generate an SEO-optimized YouTube video title.
- **1.5 Video Upload & Metadata Update:** Downloads the video, uploads it to Google Drive, YouTube, and TikTok, then updates the Google Sheet with the video and YouTube URLs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

**Overview:**  
This block initiates the workflow by retrieving new entries from a Google Sheet where images and prompts are defined but videos are not yet processed. It extracts necessary data to trigger the video creation process.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking ‘Test workflow’ (manual trigger)  
- Get new video  
- Set data  

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Periodically triggers the workflow (configured to trigger every minute) to check for new inputs.  
  - Inputs: None (trigger)  
  - Outputs: Connects to Get new video node  
  - Edge cases: If the sheet is inaccessible or empty, no new data is processed.

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation for testing purposes.  
  - Inputs: None  
  - Outputs: Connects to Get new video node  
  - Edge cases: Manual trigger bypasses schedule, useful for debugging.

- **Get new video**  
  - Type: Google Sheets (Read)  
  - Role: Reads rows from the Google Sheet where the "VIDEO" column is empty (i.e., new requests).  
  - Configuration: Filters to only fetch rows missing a video link. Reads from a specific Google Sheet document and sheet ID.  
  - Inputs: Trigger nodes  
  - Outputs: Connects to Set data node  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: API rate limits, sheet access errors, or empty results.

- **Set data**  
  - Type: Set  
  - Role: Extracts 'IMAGE' and 'PROMPT' columns from the Google Sheet row and prepares them as workflow variables.  
  - Key variables assigned: `image` (image URL), `prompt` (text prompt)  
  - Inputs: Get new video  
  - Outputs: Create video node  
  - Edge cases: Missing or malformed image URL or prompt may cause downstream failures.

---

#### 2.2 Video Generation Request

**Overview:**  
This block sends a POST request to the MiniMax Hailuo 02 API providing the image URL and prompt to generate an AI video of 6 seconds duration.

**Nodes Involved:**  
- Create video  

**Node Details:**

- **Create video**  
  - Type: HTTP Request (POST)  
  - Role: Sends a JSON payload to the MiniMax Hailuo 02 API endpoint to start video generation.  
  - Configuration:  
    - Endpoint: `https://queue.fal.run/fal-ai/minimax/hailuo-02/standard/image-to-video`  
    - Body: JSON including prompt, image_url, duration (6 seconds), prompt_optimizer enabled  
    - Headers: Content-Type: application/json, Authorization header with API Key (Fal.run API)  
  - Inputs: Set data node  
  - Outputs: Wait 60 sec. node  
  - Credentials: HTTP Header Auth with Fal.run API key  
  - Edge cases: API authentication failure, invalid image or prompt, API rate limits, network errors.

---

#### 2.3 Video Generation Status Polling

**Overview:**  
Since video generation is asynchronous, this block waits for 60 seconds and polls the API repeatedly until the video status is marked as "COMPLETED".

**Nodes Involved:**  
- Wait 60 sec.  
- Get status  
- Completed? (If)  

**Node Details:**

- **Wait 60 sec.**  
  - Type: Wait  
  - Role: Pauses workflow execution for 60 seconds to allow video processing.  
  - Inputs: Create video node or Get status node (on incomplete)  
  - Outputs: Get status node  
  - Edge cases: Excessive delays if video generation takes very long.

- **Get status**  
  - Type: HTTP Request (GET)  
  - Role: Queries the video generation status using the request_id returned by the Create video node.  
  - Configuration: URL dynamically constructed with request_id, authorization header included.  
  - Inputs: Wait 60 sec. node  
  - Outputs: Completed? node  
  - Credentials: Fal.run API key  
  - Edge cases: API errors, invalid request_id, network issues.

- **Completed?**  
  - Type: If  
  - Role: Checks if the status returned is "COMPLETED".  
  - Inputs: Get status node  
  - Outputs:  
    - True: Get Url Video node (next block)  
    - False: Wait 60 sec. (poll again)  
  - Edge cases: Status values other than expected, infinite polling loops if stuck.

---

#### 2.4 Video Retrieval & Title Generation

**Overview:**  
Once video generation is complete, this block retrieves the video URL, generates a suitable YouTube title using OpenAI GPT, and downloads the video file.

**Nodes Involved:**  
- Get Url Video  
- Generate title  
- Get File Video  

**Node Details:**

- **Get Url Video**  
  - Type: HTTP Request (GET)  
  - Role: Retrieves the video metadata and URL using the request_id.  
  - Inputs: Completed? node (True branch)  
  - Outputs: Generate title node  
  - Credentials: Fal.run API key  
  - Edge cases: Missing or expired video URL, API errors.

- **Generate title**  
  - Type: OpenAI (Langchain)  
  - Role: Uses GPT-4o-MINI to generate an SEO-friendly YouTube video title based on the prompt.  
  - Configuration: Prompt includes instructions to produce a catchy, relevant title under 60 characters in the same language as the prompt.  
  - Inputs: Get Url Video node  
  - Outputs: Get File Video node  
  - Credentials: OpenAI API key  
  - Edge cases: API rate limits, model errors, irrelevant titles.

- **Get File Video**  
  - Type: HTTP Request (GET)  
  - Role: Downloads the video file from the URL retrieved in Get Url Video node for uploading.  
  - Inputs: Generate title node  
  - Outputs: Upload Video, Upload on Youtube, Upload on TikTok nodes (parallel)  
  - Edge cases: Download failures, large file size timeouts.

---

#### 2.5 Video Upload & Metadata Update

**Overview:**  
This final block uploads the generated video to Google Drive, YouTube, and TikTok using respective APIs, and then updates the Google Sheet with the video URL and YouTube link.

**Nodes Involved:**  
- Upload Video (Google Drive)  
- Update result (Google Sheets)  
- Upload on Youtube (Upload-post.com API)  
- Upload on TikTok (Upload-post.com API)  
- Update Youtube URL (Google Sheets)  

**Node Details:**

- **Upload Video**  
  - Type: Google Drive (Upload)  
  - Role: Uploads the video file to a specified Google Drive folder, naming the file with timestamp and original filename.  
  - Inputs: Get File Video  
  - Outputs: Update result node  
  - Credentials: Google Drive OAuth2  
  - Edge cases: Storage limits, permission errors.

- **Update result**  
  - Type: Google Sheets (Update)  
  - Role: Updates the Google Sheet row with the video URL from Google Drive.  
  - Inputs: Upload Video node  
  - Outputs: None (end)  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Row mismatch, sheet access errors.

- **Upload on Youtube**  
  - Type: HTTP Request (POST)  
  - Role: Uploads the video file to YouTube via Upload-post.com API with the generated title and user profile.  
  - Inputs: Get File Video  
  - Outputs: Update Youtube URL node  
  - Credentials: Upload-post.com API key  
  - Edge cases: Authentication failure, API limits, video processing delays.

- **Upload on TikTok**  
  - Type: HTTP Request (POST)  
  - Role: Uploads the video file to TikTok via Upload-post.com API with the generated title and user profile.  
  - Inputs: Get File Video  
  - Outputs: None  
  - Credentials: Upload-post.com API key  
  - Edge cases: TikTok upload restrictions on free plans, API errors.

- **Update Youtube URL**  
  - Type: Google Sheets (Update)  
  - Role: Updates the Google Sheet row with the YouTube video URL after upload.  
  - Inputs: Upload on Youtube node  
  - Outputs: None (end)  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Timing issues if YouTube video ID is not immediately available.

---

### 3. Summary Table

| Node Name            | Node Type                | Functional Role                         | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                   |
|----------------------|--------------------------|---------------------------------------|--------------------------|--------------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger      | scheduleTrigger          | Periodic workflow start                | None                     | Get new video                  | ## STEP 4 - MAIN FLOW: Start workflow manually or with schedule every 5 min recommended        |
| When clicking ‘Test workflow’ | manualTrigger            | Manual workflow start                  | None                     | Get new video                  |                                                                                               |
| Get new video         | googleSheets             | Get new rows with missing video        | Schedule Trigger, Manual Trigger | Set data                      |                                                                                               |
| Set data             | set                      | Extract image and prompt from sheet    | Get new video             | Create video                  |                                                                                               |
| Create video          | httpRequest              | Request video creation from API        | Set data                  | Wait 60 sec.                  | ## STEP 2 - GET API KEY: Use fal.ai API key authorization header for this node                 |
| Wait 60 sec.          | wait                     | Delay for video processing              | Create video, Completed? (false) | Get status                   |                                                                                               |
| Get status            | httpRequest              | Poll API for video generation status   | Wait 60 sec.              | Completed?                   |                                                                                               |
| Completed?            | if                       | Check if video generation completed    | Get status                | Get Url Video (true), Wait 60 sec. (false) |                                                                                               |
| Get Url Video         | httpRequest              | Retrieve video URL and metadata        | Completed? (true)         | Generate title                |                                                                                               |
| Generate title        | openAi                   | Generate SEO-optimized YouTube title   | Get Url Video             | Get File Video                |                                                                                               |
| Get File Video        | httpRequest              | Download video file                     | Generate title            | Upload Video, Upload on Youtube, Upload on TikTok |                                                                                               |
| Upload Video          | googleDrive              | Upload video to Google Drive            | Get File Video            | Update result                 |                                                                                               |
| Update result         | googleSheets             | Update Google Sheet with video URL      | Upload Video              | None                         |                                                                                               |
| Upload on Youtube     | httpRequest              | Upload video to YouTube via Upload-post API | Get File Video            | Update Youtube URL            | ## STEP 3 - Upload video on Youtube and TikTok: Use Upload-post API key and set YOUR_USERNAME  |
| Upload on TikTok      | httpRequest              | Upload video to TikTok via Upload-post API | Get File Video            | None                         | ## STEP 3 - Upload video on Youtube and TikTok: Use Upload-post API key and set YOUR_USERNAME  |
| Update Youtube URL    | googleSheets             | Update Google Sheet with YouTube URL    | Upload on Youtube         | None                         |                                                                                               |
| Sticky Note1          | stickyNote               | Display example input image             | None                     | None                         | ## Image ![image](https://n3wstorage.b-cdn.net/n3witalia/girl-beach.jpeg)                      |
| Sticky Note2          | stickyNote               | Display sample prompt and result        | None                     | None                         | ## Prompt: The girl is windsurfing with her dog; ## Result: https://n3wstorage.b-cdn.net/n3witalia/girl-beach.mp4 |
| Sticky Note3          | stickyNote               | Workflow overview and purpose           | None                     | None                         | # Image to Video with MiniMax Hailuo 02: transform image + prompt into video, upload to YouTube/TikTok, update sheet |
| Sticky Note4          | stickyNote               | Instructions for Google Sheet setup     | None                     | None                         | ## STEP 1 - GOOGLE SHEET: Create sheet and populate IMAGE and PROMPT columns; VIDEO and YOUTUBE columns left blank |
| Sticky Note5          | stickyNote               | Workflow start instructions             | None                     | None                         | ## STEP 4 - MAIN FLOW: Start workflow manually or with schedule trigger                        |
| Sticky Note6          | stickyNote               | API key instructions for MiniMax Hailuo 02 | None                     | None                         | ## STEP 2 - GET API KEY: Register at fal.ai, set header authorization in Create video node    |
| Sticky Note7          | stickyNote               | Duration configuration                  | None                     | None                         | Set duration (6 or 10 sec.)                                                                   |
| Sticky Note8          | stickyNote               | Upload-post.com API key and profile setup | None                     | None                         | ## STEP 3 - Upload video on Youtube and TikTok: Create API key on Upload-post.com, set auth header, create profiles; free plan limits TikTok uploads |
| Sticky Note           | stickyNote               | Reminder for username setting           | None                     | None                         | Set YOUR_USERNAME in Step 3                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Schedule Trigger** node: Set to trigger every 1 minute (or desired interval).  
   - Add a **Manual Trigger** node for testing purposes.

2. **Fetch New Videos from Google Sheet**  
   - Add **Google Sheets** node ("Get new video"):  
     - Operation: Read rows  
     - Document ID: Your Google Sheet ID (must match your sheet with columns: IMAGE, PROMPT, VIDEO, YOUTUBE_URL)  
     - Sheet Name: Select appropriate sheet or use gid=0  
     - Filter rows where VIDEO column is empty (to find new tasks)  
     - Connect Schedule Trigger & Manual Trigger nodes to this node.

3. **Set Variables for Image and Prompt**  
   - Add a **Set** node ("Set data"):  
     - Assign `image` = `{{$json.IMAGE}}`  
     - Assign `prompt` = `{{$json.PROMPT}}`  
     - Connect "Get new video" node output to this node.

4. **Create Video Request to MiniMax Hailuo 02 API**  
   - Add **HTTP Request** node ("Create video"):  
     - Method: POST  
     - URL: `https://queue.fal.run/fal-ai/minimax/hailuo-02/standard/image-to-video`  
     - Body Content Type: JSON  
     - Body:  
       ```json
       {
         "prompt": "={{$json.prompt}}",
         "image_url": "={{$json.image}}",
         "duration": "6",
         "prompt_optimizer": true
       }
       ```  
     - Headers:  
       - Content-Type: application/json  
       - Authorization: Key YOUR_FAL_API_KEY (set via HTTP Header Auth credential)  
     - Connect Set data node to this node.

5. **Wait for Video Processing**  
   - Add **Wait** node ("Wait 60 sec."):  
     - Duration: 60 seconds  
     - Connect "Create video" node output to this node.

6. **Poll Video Generation Status**  
   - Add **HTTP Request** node ("Get status"):  
     - Method: GET  
     - URL: `https://queue.fal.run/fal-ai/minimax/requests/{{ $('Create video').item.json.request_id }}/status`  
     - Authorization: Use the same Fal.run API key credential.  
     - Connect "Wait 60 sec." node to this node.

7. **Check Completion Status**  
   - Add **If** node ("Completed?"):  
     - Condition: Check if `{{$json.status}}` equals "COMPLETED"  
     - Connect "Get status" node output to this node.

8. **Loop or Proceed Based on Status**  
   - If **False** (not completed): Connect back to "Wait 60 sec." (loop to poll again).  
   - If **True** (completed): Connect to "Get Url Video" node.

9. **Retrieve Video URL and Metadata**  
   - Add **HTTP Request** node ("Get Url Video"):  
     - Method: GET  
     - URL: `https://queue.fal.run/fal-ai/minimax/requests/{{ $json.request_id }}`  
     - Authorization: Fal.run API key  
     - Connect "Completed?" node (True) to this node.

10. **Generate YouTube Title**  
    - Add **OpenAI** node ("Generate title"):  
      - Model: GPT-4o-MINI or equivalent  
      - Messages:  
        - User: Input prompt text = `{{$json.PROMPT}}`  
        - System: Instructions to generate catchy, SEO-friendly YouTube title within 60 characters, matching input language  
      - Connect "Get Url Video" node to this node.  
      - Credentials: OpenAI API key.

11. **Download Video File**  
    - Add **HTTP Request** node ("Get File Video"):  
      - Method: GET  
      - URL: `={{ $('Get Url Video').item.json.video.url }}`  
      - Connect "Generate title" node output to this node.

12. **Upload Video to Google Drive**  
    - Add **Google Drive** node ("Upload Video"):  
      - Operation: Upload file  
      - Folder ID: Your target Google Drive folder  
      - File name: Use expression combining current timestamp and original video filename  
      - Credentials: Google Drive OAuth2  
      - Connect "Get File Video" node output to this node.

13. **Update Google Sheet with Video URL**  
    - Add **Google Sheets** node ("Update result"):  
      - Operation: Update row  
      - Document & Sheet: Same as initial sheet  
      - Match by row_number from "Get new video"  
      - Update "RESULT" column with Google Drive video URL  
      - Credentials: Google Sheets OAuth2  
      - Connect "Upload Video" node output to this node.

14. **Upload Video to YouTube via Upload-post.com API**  
    - Add **HTTP Request** node ("Upload on Youtube"):  
      - Method: POST  
      - URL: `https://api.upload-post.com/api/upload`  
      - Authentication: HTTP Header Auth with Upload-post.com API key  
      - Content-Type: multipart/form-data  
      - Body Parameters:  
        - title: `={{ $('Generate title').item.json.message.content }}`  
        - user: YOUR_USERNAME (your Upload-post profile)  
        - platform[]: youtube  
        - video: binary file from "Get File Video"  
      - Connect "Get File Video" node to this node.

15. **Update Google Sheet with YouTube URL**  
    - Add **Google Sheets** node ("Update Youtube URL"):  
      - Operation: Update row  
      - Document & Sheet: Same as initial sheet  
      - Match by row_number  
      - Update "YOUTUBE_URL" column with `https://youtu.be/{{ $json.results.youtube.video_id }}`  
      - Credentials: Google Sheets OAuth2  
      - Connect "Upload on Youtube" node output to this node.

16. **Upload Video to TikTok via Upload-post.com API**  
    - Add **HTTP Request** node ("Upload on TikTok"):  
      - Same configuration as YouTube upload, but platform[] = tiktok  
      - Connect "Get File Video" node output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                              | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Create a [Google Sheet like this](https://docs.google.com/spreadsheets/d/193tDO6xw8aSMO7lVC5kFyvlnmAdpHyGMmUbLBJUhhNs/edit?usp=sharing). Populate IMAGE and PROMPT columns; leave VIDEO and YOUTUBE columns empty for automatic update.                            | Step 1 in Sticky Note4                                                                                              |
| Register at [fal.ai](https://fal.ai/) to obtain API key. Set "Authorization: Key YOURAPIKEY" in the Create video node header authorization.                                                                                                               | Step 2 in Sticky Note6                                                                                              |
| Use [Upload-post.com](https://app.upload-post.com/) for YouTube and TikTok uploads. Free plan allows 10 uploads/month; TikTok requires paid plan. Create API key and profiles. Set YOUR_USERNAME in Upload nodes.                                         | Step 3 in Sticky Note8                                                                                              |
| Recommended video duration: 6 or 10 seconds; configured in Create video node.                                                                                                                                                                               | Sticky Note7                                                                                                        |
| Start workflow manually or via schedule trigger every 5 minutes for best results.                                                                                                                                                                          | Sticky Note5                                                                                                        |
| Example input image and expected output video sample provided via Sticky Note1 and Sticky Note2 with URLs.                                                                                                                                                   | Sticky Note1 and Sticky Note2                                                                                       |
| Workflow purpose summary: Transform static images to videos using MiniMax Hailuo 02, upload to YouTube/TikTok, and update Google Sheet with links and metadata.                                                                                             | Sticky Note3                                                                                                        |
| Set YOUR_USERNAME in the Upload on Youtube and TikTok nodes to match your Upload-post.com profile name.                                                                                                                                                    | Sticky Note                                                                                                         |

---

**Disclaimer:** The provided content originates exclusively from an n8n workflow automation. All data handled is legal and public. No illegal or offensive content is included.