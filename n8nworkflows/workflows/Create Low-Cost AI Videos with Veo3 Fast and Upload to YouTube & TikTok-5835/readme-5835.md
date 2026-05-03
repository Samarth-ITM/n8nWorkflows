Create Low-Cost AI Videos with Veo3 Fast and Upload to YouTube & TikTok

https://n8nworkflows.xyz/workflows/create-low-cost-ai-videos-with-veo3-fast-and-upload-to-youtube---tiktok-5835


# Create Low-Cost AI Videos with Veo3 Fast and Upload to YouTube & TikTok

### 1. Workflow Overview

This workflow automates the creation of low-cost AI-generated videos using the Google Veo3 Fast model, then uploads the resulting videos to YouTube and TikTok, while tracking progress and metadata through Google Sheets and Google Drive. It integrates multiple services to streamline video production, metadata generation, hosting, and social media publishing.

The workflow logically divides into the following blocks:

- **1.1 Input Reception and Scheduling**  
  Receives user input from a Google Sheet and triggers the workflow manually or periodically.

- **1.2 Video Creation Request**  
  Sends video generation requests to the Veo3 Fast API using the prompt and duration from the Google Sheet.

- **1.3 Video Creation Status Polling**  
  Periodically checks the status of the video generation request until completion.

- **1.4 Video Retrieval and Storage**  
  Downloads the completed video and uploads it to a designated Google Drive folder.

- **1.5 YouTube Metadata Generation and Upload**  
  Generates an SEO-optimized YouTube title using GPT-4o based on the prompt, uploads the video to YouTube via Upload-Post API, and updates the Google Sheet with the YouTube URL.

- **1.6 TikTok Upload**  
  Uploads the same video to TikTok through Upload-Post API.

- **1.7 Result Update**  
  Updates the original Google Sheet row with the video URL once the process is complete.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Scheduling

**Overview:**  
This block initializes workflow execution either manually or on a schedule, and reads new video requests from the Google Sheet.

**Nodes involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger  
- Get new video (Google Sheets)

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow for testing or on-demand execution.  
  - Inputs: None  
  - Outputs: Triggers "Get new video" node  
  - Edge cases: None, but manual trigger requires user intervention.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Periodically starts the workflow, recommended every 5 minutes.  
  - Configuration: Interval set to every minute (can be adjusted)  
  - Inputs: None  
  - Outputs: None (directly initiates workflow)  
  - Edge cases: Scheduling conflicts or overlap if execution time exceeds interval.

- **Get new video**  
  - Type: Google Sheets (Read operation)  
  - Role: Reads rows from Google Sheet where the "VIDEO" column is empty, i.e., new video requests.  
  - Configuration: Filters on "VIDEO" column to select empty fields, reads from sheet "gid=0" and specific Google Sheet document.  
  - Inputs: Trigger from manual or schedule trigger  
  - Outputs: Passes new video data (prompt, duration, row number) to "Set data" node  
  - Edge cases: Google Sheets API quota limits, empty or malformed rows, access permission issues.

---

#### 1.2 Video Creation Request

**Overview:**  
Builds the prompt with duration metadata and sends the video creation request to the Veo3 Fast AI API.

**Nodes involved:**  
- Set data  
- Create Video

**Node Details:**  

- **Set data**  
  - Type: Set  
  - Role: Formats the prompt by embedding the original prompt and video duration into a single string.  
  - Configuration: Assigns `"prompt": "{{ $json.PROMPT }}\n\nDuration of the video: {{ $json.DURATION }}"`  
  - Inputs: Output from "Get new video"  
  - Outputs: JSON with combined prompt passed to "Create Video"  
  - Edge cases: Missing or malformed input prompt or duration fields.

- **Create Video**  
  - Type: HTTP Request  
  - Role: Sends POST request to Veo3 Fast API endpoint to request AI video generation, passing the formatted prompt.  
  - Configuration:  
    - URL: `https://queue.fal.run/fal-ai/veo3/fast`  
    - Method: POST  
    - Body: JSON with `prompt` parameter from "Set data"  
    - Headers: Content-Type application/json, Authorization header set from stored credential  
  - Inputs: Combined prompt JSON  
  - Outputs: Receives a response with a `request_id` for status tracking. Passed to "Wait 60 sec." node  
  - Edge cases: API key invalid or missing, network timeout, improper prompt formatting.

---

#### 1.3 Video Creation Status Polling

**Overview:**  
Periodically checks the status of the video creation request until it is marked "COMPLETED".

**Nodes involved:**  
- Wait 60 sec.  
- Get status  
- Completed? (IF node)

**Node Details:**  

- **Wait 60 sec.**  
  - Type: Wait  
  - Role: Pauses workflow execution for 60 seconds before polling status again.  
  - Configuration: Wait time set to 60 seconds  
  - Inputs: From "Create Video" or "Get status"  
  - Outputs: Triggers "Get status" after wait  
  - Edge cases: Delays in execution, workflow timeout if process takes too long.

- **Get status**  
  - Type: HTTP Request  
  - Role: Requests status of video generation by querying Veo3 API with `request_id`.  
  - Configuration:  
    - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $('Create Video').item.json.request_id }}/status`  
    - Method: GET  
    - Authentication: HTTP Header Auth with API key  
  - Inputs: Triggers after wait  
  - Outputs: Passes status JSON to "Completed?" node  
  - Edge cases: Authorization errors, request_id invalid, network issues.

- **Completed? (IF node)**  
  - Type: If  
  - Role: Checks if the status returned equals "COMPLETED".  
  - Configuration: Condition equals to "COMPLETED"  
  - Inputs: Status JSON from "Get status"  
  - Outputs:  
    - If TRUE: Proceeds to "Get Url Video" node  
    - If FALSE: Loops back to "Wait 60 sec." node to poll again  
  - Edge cases: Status field missing or unexpected.

---

#### 1.4 Video Retrieval and Storage

**Overview:**  
Once video generation is completed, retrieves the video URL, downloads the video file, and uploads it to Google Drive.

**Nodes involved:**  
- Get Url Video  
- Generate title (OpenAI)  
- Get File Video  
- Upload Video

**Node Details:**  

- **Get Url Video**  
  - Type: HTTP Request  
  - Role: Fetches detailed video info including the direct video URL using the `request_id`.  
  - Configuration:  
    - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}`  
    - Method: GET  
    - Auth: HTTP Header Auth  
  - Inputs: From "Completed?" node (TRUE branch)  
  - Outputs: Passes video metadata to "Generate title" node  
  - Edge cases: Video info missing or corrupted.

- **Generate title**  
  - Type: OpenAI (LangChain)  
  - Role: Uses GPT-4o-mini model to generate an SEO-optimized YouTube video title from the prompt.  
  - Configuration:  
    - Model: gpt-4o-mini  
    - Messages:  
      - System prompt defines SEO rules and output format  
      - User prompt passes original video description  
  - Inputs: From "Get Url Video"  
  - Outputs: Passes generated title to "Get File Video"  
  - Edge cases: OpenAI API limits, prompt fails, title too long or empty.

- **Get File Video**  
  - Type: HTTP Request  
  - Role: Downloads the video file from the URL retrieved.  
  - Configuration:  
    - URL: `{{ $('Get Url Video').item.json.video.url }}`  
    - Method: GET  
  - Inputs: From "Generate title"  
  - Outputs: Binary video data to "Upload Video", "HTTP Request" (YouTube upload), and "Upload on TikTok" nodes  
  - Edge cases: Network errors, video URL expired, large file download issues.

- **Upload Video**  
  - Type: Google Drive  
  - Role: Uploads the downloaded video file to a specific Google Drive folder.  
  - Configuration:  
    - File name generated dynamically as timestamp + original file name  
    - Target folder ID specified  
  - Inputs: Binary video data from "Get File Video"  
  - Outputs: Passes file metadata to "Update result"  
  - Edge cases: Google Drive quota limits, permission errors, upload failures.

---

#### 1.5 YouTube Metadata Generation and Upload

**Overview:**  
Uploads the video to YouTube via Upload-Post API using the generated title and updates the Google Sheet with the YouTube URL.

**Nodes involved:**  
- HTTP Request (Upload to YouTube)  
- Update Youtube URL (Google Sheets)  
- Update result (Google Sheets)

**Node Details:**  

- **HTTP Request (Upload to YouTube)**  
  - Type: HTTP Request  
  - Role: Uploads video binary to YouTube platform via Upload-Post API.  
  - Configuration:  
    - URL: `https://api.upload-post.com/api/upload`  
    - Method: POST  
    - Body: multipart/form-data with fields: title (generated title), user (YOUR_USERNAME), platform set to "youtube", video binary data  
    - Auth: HTTP Header Auth with API key  
  - Inputs: Binary video from "Get File Video" and title from "Generate title"  
  - Outputs: Passes upload response to "Update Youtube URL"  
  - Edge cases: API quota limits, invalid user or auth errors, network errors.

- **Update Youtube URL**  
  - Type: Google Sheets (Update operation)  
  - Role: Updates the corresponding Google Sheet row with the YouTube video URL after upload.  
  - Configuration:  
    - Matches on `row_number` from initial sheet row  
    - Updates `YOUTUBE_URL` column with `https://youtu.be/{{ $json.results.youtube.video_id }}`  
  - Inputs: Upload response JSON  
  - Outputs: None (final update step for YouTube link)  
  - Edge cases: Google Sheets API quota, row mismatch, permission issues.

- **Update result**  
  - Type: Google Sheets (Update operation)  
  - Role: Updates the original Google Sheet row with the `VIDEO` column set to the uploaded Google Drive video URL.  
  - Configuration:  
    - Matches on `row_number`  
    - Updates `VIDEO` column with Google Drive file URL  
  - Inputs: From "Upload Video" node  
  - Outputs: End of workflow for data update  
  - Edge cases: Same as above, syncing issues.

---

#### 1.6 TikTok Upload

**Overview:**  
Uploads the same video to TikTok platform via Upload-Post API.

**Nodes involved:**  
- Upload on TikTok

**Node Details:**  

- **Upload on TikTok**  
  - Type: HTTP Request  
  - Role: Uploads video binary to TikTok platform via Upload-Post API.  
  - Configuration:  
    - URL: `https://api.upload-post.com/api/upload`  
    - Method: POST  
    - Body: multipart/form-data with fields: title (generated title), user (YOUR_USERNAME), platform set to "tiktok", video binary data  
    - Auth: HTTP Header Auth with a separate credential  
  - Inputs: Video binary and title from "Get File Video" and "Generate title"  
  - Outputs: None (no further workflow steps)  
  - Edge cases: Paid plan required for TikTok uploads, auth errors, quota limits.

---

### 3. Summary Table

| Node Name                 | Node Type                   | Functional Role                                               | Input Node(s)              | Output Node(s)                | Sticky Note                                                                                                      |
|---------------------------|-----------------------------|---------------------------------------------------------------|----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger              | Manual workflow start                                         | -                          | Get new video                |                                                                                                                 |
| Schedule Trigger          | Schedule Trigger             | Periodic workflow start                                       | -                          | Get new video (implied)       | STEP 4 - MAIN FLOW: Recommended 5-minute intervals                                                             |
| Get new video             | Google Sheets                | Reads new video requests from Google Sheet                    | When clicking ‘Test workflow’, Schedule Trigger | Set data                   |                                                                                                                 |
| Set data                  | Set                         | Formats prompt including duration                             | Get new video               | Create Video                 |                                                                                                                 |
| Create Video              | HTTP Request                | Sends video generation request to Veo3 Fast API               | Set data                   | Wait 60 sec.                 | STEP 2 - GET API KEY: Set Authorization header with API key                                                     |
| Wait 60 sec.              | Wait                        | Waits 60 seconds between status checks                        | Create Video, Get status    | Get status                   |                                                                                                                 |
| Get status                | HTTP Request                | Polls video generation status                                 | Wait 60 sec.               | Completed?                   |                                                                                                                 |
| Completed?                | If                          | Checks if video generation is complete                        | Get status                 | Get Url Video (if complete), Wait 60 sec. (if not) |                                                                                                                 |
| Get Url Video             | HTTP Request                | Gets video metadata and URL                                   | Completed?                 | Generate title               |                                                                                                                 |
| Generate title            | OpenAI (LangChain)           | Creates SEO-optimized YouTube title                           | Get Url Video              | Get File Video               |                                                                                                                 |
| Get File Video            | HTTP Request                | Downloads video file from URL                                 | Generate title             | Upload Video, HTTP Request (YouTube), Upload on TikTok |                                                                                                                 |
| Upload Video              | Google Drive                | Uploads video file to Google Drive                            | Get File Video             | Update result                |                                                                                                                 |
| HTTP Request (YouTube)    | HTTP Request                | Uploads video to YouTube via Upload-Post API                 | Get File Video, Generate title | Update Youtube URL           | STEP 3 - Upload video on YouTube: Set API key and username in header and body                                   |
| Update Youtube URL        | Google Sheets               | Updates Google Sheet row with YouTube URL                     | HTTP Request (YouTube)     | -                           |                                                                                                                 |
| Update result             | Google Sheets               | Updates Google Sheet row with Google Drive video URL          | Upload Video               | -                           |                                                                                                                 |
| Upload on TikTok          | HTTP Request                | Uploads video to TikTok via Upload-Post API                   | Get File Video, Generate title | -                           | STEP 3 - Upload video on YouTube: Note about TikTok upload requiring paid plan                                    |
| Sticky Note3              | Sticky Note                 | Workflow description and overview                             | -                          | -                           | Workflow purpose and integrations overview                                                                      |
| Sticky Note4              | Sticky Note                 | Instructions for Google Sheet setup                           | -                          | -                           | Link to example Google Sheet for input                                                                          |
| Sticky Note5              | Sticky Note                 | Scheduling recommendation                                    | -                          | -                           | Recommended schedule trigger interval                                                                            |
| Sticky Note6              | Sticky Note                 | API key setup instructions                                   | -                          | -                           | How to set API key for Veo3 Fast API                                                                             |
| Sticky Note7              | Sticky Note                 | Reminder to set API key                                       | -                          | -                           |                                                                                                                 |
| Sticky Note8              | Sticky Note                 | YouTube/TikTok upload instructions                           | -                          | -                           | Upload-Post API key setup details and limits                                                                     |
| Sticky Note               | Sticky Note                 | Reminder to set username for Upload-Post                     | -                          | -                           |                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual and Schedule Triggers:**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’". No parameters needed.  
   - Add a **Schedule Trigger** node named "Schedule Trigger". Set interval to every 5 minutes (or desired frequency).

2. **Read New Video Requests from Google Sheet:**  
   - Add a **Google Sheets** node named "Get new video".  
   - Configure to read rows where the "VIDEO" column is empty (filter on "VIDEO").  
   - Set documentId to your Google Sheet containing video requests.  
   - Sheet name: Use the sheet ID or name (e.g., "gid=0").  
   - Connect both triggers ("Manual Trigger" and "Schedule Trigger") to this node.

3. **Format Prompt with Duration:**  
   - Add a **Set** node named "Set data".  
   - Add a field "prompt" with the value:  
     `{{$json.PROMPT}}\n\nDuration of the video: {{$json.DURATION}}`  
   - Connect "Get new video" to "Set data".

4. **Send Video Creation Request:**  
   - Add an **HTTP Request** node named "Create Video".  
   - Set method to POST; URL to `https://queue.fal.run/fal-ai/veo3/fast`.  
   - Body type: JSON with field "prompt" set to `{{$json.prompt}}`.  
   - Add header `Content-Type: application/json`.  
   - Configure HTTP Header Authentication using your Veo3 API key (header name: "Authorization", value: "Key YOURAPIKEY").  
   - Connect "Set data" to "Create Video".

5. **Wait Before Polling Status:**  
   - Add a **Wait** node named "Wait 60 sec." with a 60-second wait time.  
   - Connect "Create Video" to "Wait 60 sec.".

6. **Poll Video Generation Status:**  
   - Add an **HTTP Request** node named "Get status".  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $('Create Video').item.json.request_id }}/status`  
   - Use same HTTP Header Auth credentials.  
   - Connect "Wait 60 sec." to "Get status".

7. **Check if Video is Completed:**  
   - Add an **If** node named "Completed?".  
   - Condition: `$json.status` equals "COMPLETED" (case-sensitive).  
   - Connect "Get status" to "Completed?".

8. **Loop or Proceed Based on Completion:**  
   - From "Completed?" node:  
     - If FALSE, connect back to "Wait 60 sec." node to continue polling.  
     - If TRUE, connect to "Get Url Video".

9. **Get Video Metadata and URL:**  
   - Add an **HTTP Request** node named "Get Url Video".  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}`  
   - Use same HTTP Header Auth credentials.  
   - Connect "Completed?" TRUE branch to "Get Url Video".

10. **Generate YouTube Title with GPT-4o:**  
    - Add an **OpenAI (LangChain)** node named "Generate title".  
    - Model: gpt-4o-mini  
    - Messages:  
      - System prompt: SEO expert instructions as per block 2.  
      - User prompt: `Input: {{ $('Get new video').item.json.PROMPT }}`  
    - Connect "Get Url Video" to "Generate title".

11. **Download Video File:**  
    - Add an **HTTP Request** node named "Get File Video".  
    - Method: GET  
    - URL: `{{ $('Get Url Video').item.json.video.url }}`  
    - Connect "Generate title" to "Get File Video".

12. **Upload Video to Google Drive:**  
    - Add a **Google Drive** node named "Upload Video".  
    - Upload binary data from "Get File Video".  
    - File name: `{{$now.format('yyyyLLddHHmmss')}}-{{ $('Get Url Video').item.json.video.file_name }}`  
    - Folder: Set target folder ID on Google Drive.  
    - Connect "Get File Video" to "Upload Video".

13. **Upload Video to YouTube via Upload-Post API:**  
    - Add an **HTTP Request** node named "HTTP Request".  
    - Method: POST  
    - URL: `https://api.upload-post.com/api/upload`  
    - Body type: multipart-form-data with parameters:  
      - title: `{{ $('Generate title').item.json.message.content }}`  
      - user: YOUR_USERNAME (replace with your Upload-Post username)  
      - platform[]: "youtube"  
      - video: binary data from "Get File Video"  
    - Authentication: HTTP Header Auth with Upload-Post API key.  
    - Connect "Get File Video" and "Generate title" to this node.

14. **Update Google Sheet with YouTube URL:**  
    - Add a **Google Sheets** node named "Update Youtube URL".  
    - Operation: Update  
    - Match on `row_number` from "Get new video".  
    - Update `YOUTUBE_URL` column with `https://youtu.be/{{ $json.results.youtube.video_id }}`.  
    - Connect "HTTP Request" to "Update Youtube URL".

15. **Update Google Sheet with Video URL (Google Drive):**  
    - Add a **Google Sheets** node named "Update result".  
    - Operation: Update  
    - Match on `row_number` from "Get new video".  
    - Update `VIDEO` column with `{{ $('Get Url Video').item.json.video.url }}` or Google Drive URL from "Upload Video".  
    - Connect "Upload Video" to "Update result".

16. **Upload Video to TikTok via Upload-Post API:**  
    - Add an **HTTP Request** node named "Upload on TikTok".  
    - Same configuration as YouTube upload but set platform[] to "tiktok".  
    - Use separate HTTP Header Auth credentials if applicable.  
    - Connect "Get File Video" and "Generate title" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow allows creating cheaper AI videos with Google Veo3 Fast, saving to Google Drive, and auto-uploading. | Workflow overview sticky note inside the workflow ("Sticky Note3")                                  |
| Google Sheet template example for input setup. Insert PROMPT and DURATION, leave VIDEO column blank.          | https://docs.google.com/spreadsheets/d/1pcoY9N_vQp44NtSRR5eskkL5Qd0N0BGq7Jh_4m-7VEQ/edit?usp=sharing   |
| Instructions for obtaining Veo3 API key at https://fal.ai/ and setting HTTP Header Auth in "Create Video" node. | Sticky Note6 inside the workflow                                                                     |
| Instructions for Upload-Post API key setup and usage for YouTube and TikTok uploads.                          | Sticky Note8 inside the workflow                                                                     |
| TikTok upload requires paid Upload-Post plan; free plan supports only YouTube uploads.                        | Sticky Note8 inside the workflow                                                                     |
| Recommended to schedule the workflow trigger every 5 minutes for best performance.                            | Sticky Note5 inside the workflow                                                                     |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. All processing complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.