Generate AI Videos with Sora2 Pro, Save to Google Drive and Upload to YouTube

https://n8nworkflows.xyz/workflows/generate-ai-videos-with-sora2-pro--save-to-google-drive-and-upload-to-youtube-9576


# Generate AI Videos with Sora2 Pro, Save to Google Drive and Upload to YouTube

### 1. Workflow Overview

This n8n workflow automates the generation of AI videos using the OpenAI-powered Sora2 Pro service, saves the resulting videos to Google Drive, and uploads them to YouTube automatically. It is designed for content creators or marketers who want to streamline video production, storage, and publication processes using a Google Sheet as a central input/output interface.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Polls Google Sheets for new video requests and prepares the prompt for video creation.
- **1.2 AI Video Generation Request:** Sends the video creation request to Sora2 Pro AI service.
- **1.3 Video Processing Status Polling:** Repeatedly checks the status of the video generation until completed.
- **1.4 Video Retrieval and Storage:** Retrieves the generated video URL, downloads the video, and uploads it to Google Drive.
- **1.5 YouTube Title Generation:** Uses GPT-5 to generate an SEO-optimized YouTube video title based on the original prompt.
- **1.6 Video Upload to YouTube:** Uploads the video to YouTube through the Upload-Post API and updates the Google Sheet with the YouTube URL.
- **1.7 Google Sheets Update:** Updates the Google Sheet with the video URL and YouTube link to keep track of progress and results.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
  This block fetches new rows from a Google Sheet where users input video descriptions and durations. It formats the prompt for the AI video generation.

- **Nodes Involved:**  
  - Get new video  
  - Set data

- **Node Details:**

  - **Get new video**  
    - Type: Google Sheets (Read operation)  
    - Role: Retrieves new video requests where the "VIDEO" column is empty, filtering to only fetch unsatisfied rows.  
    - Configuration: Reads from a specific Google Sheet document and sheet (gid=0). Uses OAuth2 credentials.  
    - Inputs: Triggered manually or via schedule trigger indirectly.  
    - Outputs: Provides rows with PROMPT and DURATION fields.  
    - Edge Cases: Possible authentication errors, empty or malformed rows, API limits on Google Sheets.  

  - **Set data**  
    - Type: Set Node  
    - Role: Constructs a single prompt string combining the video description and duration from the Google Sheet row.  
    - Configuration: Assigns a new variable `prompt` with the template: `"{{PROMPT}}\n\nDuration of the video: {{DURATION}}"` by expression.  
    - Inputs: From "Get new video" node.  
    - Outputs: JSON with a single field `prompt` for the video creation API.  
    - Edge Cases: Missing PROMPT or DURATION fields could result in incomplete prompts.

---

#### 2.2 AI Video Generation Request

- **Overview:**  
  Sends the formatted prompt to the Sora2 Pro API to start video generation.

- **Nodes Involved:**  
  - Create Video

- **Node Details:**

  - **Create Video**  
    - Type: HTTP Request  
    - Role: POST request to Sora2 Pro API endpoint `/fal-ai/sora-2/text-to-video/pro` with the prompt as JSON body.  
    - Configuration: Uses HTTP header authentication with API key in header "Authorization: Key YOURAPIKEY".  
    - Inputs: From "Set data" node.  
    - Outputs: Returns a `request_id` used to check video generation status.  
    - Edge Cases: Network errors, invalid API key, API quota limits, unexpected response format.

---

#### 2.3 Video Processing Status Polling

- **Overview:**  
  Polls the video generation status every 60 seconds until the status is "COMPLETED".

- **Nodes Involved:**  
  - Wait 60 sec.  
  - Get status  
  - Completed? (IF node)

- **Node Details:**

  - **Wait 60 sec.**  
    - Type: Wait  
    - Role: Waits 60 seconds before next status check to avoid API flooding.  
    - Inputs: From "Create Video" and "Get status" nodes (loop back).  
    - Outputs: Triggers "Get status" node.

  - **Get status**  
    - Type: HTTP Request  
    - Role: GET request to check status of video generation using `request_id`.  
    - Configuration: URL constructed dynamically using the `request_id` from "Create Video" node. Uses same API key authentication.  
    - Inputs: From "Wait 60 sec." or initial from "Create Video".  
    - Outputs: Returns status JSON field.  
    - Edge Cases: Network errors, API rate limits, invalid request_id.

  - **Completed?**  
    - Type: IF Node  
    - Role: Checks if the `status` field in response equals "COMPLETED".  
    - Inputs: From "Get status".  
    - Outputs:  
      - If yes: proceeds to "Get Url Video" node.  
      - If no: loops back to "Wait 60 sec." node for another poll.  
    - Edge Cases: Status field missing or unexpected values.

---

#### 2.4 Video Retrieval and Storage

- **Overview:**  
  Retrieves the video URL after completion, downloads the video file, then uploads it to Google Drive.

- **Nodes Involved:**  
  - Get Url Video  
  - Generate title  
  - Get File Video  
  - Upload Video  
  - Update result

- **Node Details:**

  - **Get Url Video**  
    - Type: HTTP Request  
    - Role: GET request to retrieve detailed video info using `request_id`.  
    - Inputs: From "Completed?" node on true branch.  
    - Outputs: Provides video URL and file name.  
    - Edge Cases: API errors, incomplete data.

  - **Generate title**  
    - Type: OpenAI (Langchain node)  
    - Role: Generates an SEO-optimized YouTube title using GPT-5, based on original video prompt.  
    - Configuration:  
      - Model: GPT-5  
      - System prompt instructs to create short (max 60 characters), catchy, keyword-rich titles without clickbait, in the input language.  
      - User input: Original prompt from Google Sheet.  
    - Inputs: From "Get Url Video".  
    - Outputs: Video title string.  
    - Edge Cases: API rate limits, unexpected model output.

  - **Get File Video**  
    - Type: HTTP Request  
    - Role: Downloads the video file from the URL obtained in "Get Url Video".  
    - Inputs: From "Generate title".  
    - Outputs: Binary video data for upload.  
    - Edge Cases: Network errors, large file timeouts.

  - **Upload Video**  
    - Type: Google Drive  
    - Role: Uploads the downloaded video file to a specific Google Drive folder.  
    - Configuration:  
      - Filename uses current timestamp + original file name.  
      - Folder ID is set to a pre-configured Google Drive folder.  
      - Uses Google Drive OAuth2 credentials.  
    - Inputs: From "Get File Video".  
    - Outputs: Google Drive file metadata.  
    - Edge Cases: Authentication failures, quota limits, file size limits.

  - **Update result**  
    - Type: Google Sheets (Update operation)  
    - Role: Updates the Google Sheet row with the Google Drive video URL and row number for tracking.  
    - Configuration: Matches row by row_number and updates VIDEO column with Drive URL.  
    - Inputs: From "Upload Video".  
    - Outputs: Confirmation of update.  
    - Edge Cases: Row mismatch, API quota limits.

---

#### 2.5 YouTube Title Generation and Video Upload

- **Overview:**  
  Generates the video title for YouTube and uploads the video using Upload-Post API, then updates the Google Sheet with the YouTube URL.

- **Nodes Involved:**  
  - HTTP Request (Upload to YouTube via Upload-Post)  
  - Update Youtube URL

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Uploads the video to YouTube via the Upload-Post service API.  
    - Configuration:  
      - POST multipart/form-data with parameters: title (from "Generate title"), user (Upload-Post profile username), platform[] = "youtube", and video binary data.  
      - Auth header uses API key for Upload-Post.  
    - Inputs: From "Get File Video" (binary video) and "Generate title" (title).  
    - Outputs: Upload result containing video ID for YouTube.  
    - Edge Cases: Authentication errors, upload failures, quota limits.

  - **Update Youtube URL**  
    - Type: Google Sheets (Update operation)  
    - Role: Updates the same Google Sheet row to set the YOUTUBE_URL column with the new video link formatted as `https://youtu.be/{video_id}`.  
    - Inputs: From "HTTP Request" upload node.  
    - Outputs: Confirmation of update.  
    - Edge Cases: Row mismatch, API limits.

---

#### 2.6 Workflow Triggers and Auxiliary Notes

- **Nodes Involved:**  
  - Schedule Trigger  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Sticky Notes

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Periodically triggers the workflow every minute (configurable interval).  
    - Inputs: None  
    - Outputs: Starts the workflow chain by triggering "Get new video".  
    - Edge Cases: Timing overlaps, concurrency issues.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual testing of the workflow on demand.  
    - Inputs: None  
    - Outputs: Triggers "Get new video".  
    - Edge Cases: Human error or manual triggering during ongoing runs.

  - **Sticky Notes**  
    - Provide documentation and instructions embedded in the workflow canvas for setup steps including:  
      - Google Sheet setup  
      - API key acquisition for Fal.ai and Upload-Post  
      - Authentication header configuration  
      - Scheduling recommendations  
      - Username configuration for Upload-Post profile

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                               | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                         |
|-------------------------|-----------------------|-----------------------------------------------|-----------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger        | Manual workflow start                         | -                           | Get new video               |                                                                                                                                    |
| Schedule Trigger        | Schedule Trigger       | Periodic workflow start                        | -                           | Get new video               | Recommended to set 5-minute intervals                                                                                              |
| Get new video           | Google Sheets          | Reads new video requests from Google Sheet    | When clicking ‘Test workflow’, Schedule Trigger | Set data                   |                                                                                                                                    |
| Set data                | Set                   | Formats prompt combining description and duration | Get new video               | Create Video                |                                                                                                                                    |
| Create Video            | HTTP Request           | Sends video generation request to Sora2 Pro  | Set data                    | Wait 60 sec.                | Set API key in header "Authorization: Key YOURAPIKEY"                                                                              |
| Wait 60 sec.            | Wait                   | Waits 60 seconds before polling status        | Create Video, Get status     | Get status                  |                                                                                                                                    |
| Get status              | HTTP Request           | Polls video generation status                  | Wait 60 sec.                | Completed?                  |                                                                                                                                    |
| Completed?              | IF                     | Checks if video generation is completed       | Get status                  | Get Url Video (if yes), Wait 60 sec. (if no) |                                                                                                                                    |
| Get Url Video           | HTTP Request           | Gets detailed video info including video URL  | Completed?                  | Generate title              |                                                                                                                                    |
| Generate title          | OpenAI (Langchain)     | Generates YouTube SEO-optimized video title   | Get Url Video               | Get File Video              |                                                                                                                                    |
| Get File Video          | HTTP Request           | Downloads video file from URL                   | Generate title              | Upload Video, HTTP Request (Upload to YouTube) |                                                                                                                                    |
| Upload Video            | Google Drive           | Uploads video file to Google Drive folder      | Get File Video              | Update result               |                                                                                                                                    |
| Update result           | Google Sheets          | Updates Google Sheet with Google Drive video URL | Upload Video               | -                          |                                                                                                                                    |
| HTTP Request (Upload)   | HTTP Request           | Uploads video to YouTube via Upload-Post API  | Get File Video, Generate title | Update Youtube URL          | Set YOUR_USERNAME in Upload-Post API                                                                                              |
| Update Youtube URL      | Google Sheets          | Updates Google Sheet with YouTube video URL   | HTTP Request (Upload)       | -                          |                                                                                                                                    |
| Sticky Note3            | Sticky Note            | Workflow title and overview                     | -                           | -                          | # Generate AI Videos with Sora2 Pro, Save to Google Drive and Upload to YouTube (full description)                                 |
| Sticky Note4            | Sticky Note            | Google Sheet input instructions                 | -                           | -                          | Link to example Google Sheet for input                                                                                            |
| Sticky Note5            | Sticky Note            | Main flow starting recommendations              | -                           | -                          | Recommended to trigger manually or via schedule every 5 minutes                                                                    |
| Sticky Note6            | Sticky Note            | API key instructions for Fal.ai                  | -                           | -                          | Link to Fal.ai account creation and API key setup                                                                                  |
| Sticky Note7            | Sticky Note            | Reminder to set API key in "Create Video" node | -                           | -                          |                                                                                                                                    |
| Sticky Note8            | Sticky Note            | Upload-Post API key and profile setup instructions | -                           | -                          | Link to Upload-Post manage API keys and usage                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’". No parameters needed.  
   - Add a **Schedule Trigger** node named "Schedule Trigger" set to run every 1 minute or configurable interval (recommended 5 minutes).

2. **Google Sheets - Fetch New Video Requests:**  
   - Add a **Google Sheets** node named "Get new video".  
   - Operation: Read rows from your Google Sheet (document ID and sheet gid).  
   - Filter: Only rows where "VIDEO" column is empty (to fetch unprocessed requests).  
   - Credentials: Configure Google Sheets OAuth2 credential with access to the sheet.

3. **Prepare Video Prompt:**  
   - Add a **Set** node named "Set data".  
   - Create a new variable `prompt` with the expression:  
     `{{$json.PROMPT}}\n\nDuration of the video: {{$json.DURATION}}`  
   - Connect "Get new video" → "Set data".

4. **Send Video Creation Request:**  
   - Add an **HTTP Request** node named "Create Video".  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/sora-2/text-to-video/pro`  
   - Body (JSON): `{ "prompt": "{{$json.prompt}}" }`  
   - Headers: Content-Type: application/json, Authorization: Key YOURAPIKEY  
   - Authentication: HTTP Header Auth with Fal.ai API key credentials.  
   - Connect "Set data" → "Create Video".

5. **Wait and Poll Status:**  
   - Add a **Wait** node named "Wait 60 sec." set to 60 seconds.  
   - Connect "Create Video" → "Wait 60 sec.".

   - Add an **HTTP Request** node named "Get status".  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/sora-2/requests/{{$json.request_id}}/status`  
   - Authentication: Same as "Create Video".  
   - Connect "Wait 60 sec." → "Get status".

   - Add an **IF** node named "Completed?".  
   - Condition: Check if `{{$json.status}}` equals "COMPLETED".  
   - Connect "Get status" → "Completed?".

   - Connect "Completed?" FALSE output → back to "Wait 60 sec." (loop).  
   - Connect "Completed?" TRUE output → "Get Url Video" node (next step).

6. **Retrieve Video URL:**  
   - Add an **HTTP Request** node named "Get Url Video".  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/sora-2/requests/{{$json.request_id}}`  
   - Authentication: Same as above.  
   - Connect "Completed?" TRUE → "Get Url Video".

7. **Generate YouTube Title with GPT-5:**  
   - Add an **OpenAI (Langchain)** node named "Generate title".  
   - Model: GPT-5  
   - Messages:  
     - User: `Input: {{$json.PROMPT}}`  
     - System: Instructions for SEO optimized YouTube title (max 60 chars, keywords, no clickbait, same language).  
   - Credentials: OpenAI API key.  
   - Connect "Get Url Video" → "Generate title".

8. **Download Video File:**  
   - Add an **HTTP Request** node named "Get File Video".  
   - Method: GET  
   - URL: `{{$json.video.url}}` (from "Get Url Video").  
   - Connect "Generate title" → "Get File Video".

9. **Upload Video to Google Drive:**  
   - Add **Google Drive** node named "Upload Video".  
   - Operation: Upload file  
   - File Name: `{{$now.format('yyyyLLddHHmmss')}}-{{$json.video.file_name}}`  
   - Folder ID: your target Google Drive folder ID  
   - Credentials: Google Drive OAuth2.  
   - Connect "Get File Video" → "Upload Video".

10. **Update Google Sheet with Drive Video URL:**  
    - Add Google Sheets node named "Update result".  
    - Operation: Update row matching `row_number` from the original video request.  
    - Update "VIDEO" column with Google Drive URL from "Upload Video".  
    - Credentials: Google Sheets OAuth2.  
    - Connect "Upload Video" → "Update result".

11. **Upload Video to YouTube via Upload-Post:**  
    - Add HTTP Request node named "HTTP Request".  
    - Method: POST multipart/form-data to `https://api.upload-post.com/api/upload`.  
    - Parameters:  
      - title: from "Generate title" output  
      - user: YOUR_USERNAME (Upload-Post profile)  
      - platform[]: "youtube"  
      - video: binary data from "Get File Video"  
    - Authentication: HTTP Header Auth with Upload-Post API key.  
    - Connect "Get File Video" and "Generate title" → "HTTP Request".

12. **Update Google Sheet with YouTube URL:**  
    - Add Google Sheets node named "Update Youtube URL".  
    - Operation: Update row by `row_number`.  
    - Update "YOUTUBE_URL" column with `https://youtu.be/{{$json.results.youtube.video_id}}`.  
    - Credentials: Google Sheets OAuth2.  
    - Connect "HTTP Request" → "Update Youtube URL".

13. **Connect Triggers to Start:**  
    - Connect both "When clicking ‘Test workflow’" and "Schedule Trigger" nodes → "Get new video".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Create a [Google Sheet like this](https://docs.google.com/spreadsheets/d/1pcoY9N_vQp44NtSRR5eskkL5Qd0N0BGq7Jh_4m-7VEQ/edit?usp=sharing) with columns PROMPT, DURATION, VIDEO. | Google Sheet input template.                                                                             |
| Create an account on [Fal.ai](https://fal.ai/) to obtain your API key for Sora2 Pro video generation. Set it in the "Create Video" node header authorization.             | Fal.ai API key setup.                                                                                     |
| Create and manage API keys at [Upload-Post Manage Api Keys](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) for YouTube upload.    | Upload-Post API key and profile management.                                                              |
| Recommended to trigger workflow manually or on a schedule every 5 minutes to process new video requests.                                                                 | Scheduling recommendation.                                                                                |
| Use consistent naming and credentials for Google Sheets, Google Drive, OpenAI, and HTTP Header Auth nodes to ensure seamless integration.                                | Credential management best practices.                                                                     |

---

_Disclaimer: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._