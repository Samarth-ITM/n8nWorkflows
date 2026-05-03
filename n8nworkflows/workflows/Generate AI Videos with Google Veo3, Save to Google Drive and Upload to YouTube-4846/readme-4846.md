Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube

https://n8nworkflows.xyz/workflows/generate-ai-videos-with-google-veo3--save-to-google-drive-and-upload-to-youtube-4846


# Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube

### 1. Workflow Overview

This workflow automates the generation of AI videos using Google Veo3, saves the videos to Google Drive, generates optimized YouTube titles via GPT-4o, and uploads the videos to YouTube. It is designed for content creators and marketers who want a streamlined, automated pipeline from video idea entry to video publishing and tracking.

**Logical blocks:**

- **1.1 Input Reception and Initialization:**  
  Trigger the workflow manually or on schedule; read video requests from Google Sheets.

- **1.2 Video Creation Request:**  
  Send prompt and duration data to Google Veo3 API to create the video.

- **1.3 Video Processing Status Polling:**  
  Poll the Google Veo3 queue API to check if video creation is completed.

- **1.4 Video Retrieval and Storage:**  
  Fetch the generated video URL, download the video file, and upload it to Google Drive.

- **1.5 YouTube Title Generation and Upload:**  
  Generate an SEO-optimized YouTube title with GPT-4o based on the video description, upload the video to YouTube via Upload-Post.com API, and update the Google Sheet with YouTube URL.

- **1.6 Google Sheet Updates:**  
  Update the Google Sheet with the video URL and YouTube video link for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block triggers the workflow either manually or on a schedule and reads new video requests from a Google Sheet where users enter video prompts and durations.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Schedule Trigger  
  - Get new video (Google Sheets)  
  - Set data (Set node)  

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Start workflow manually for testing  
    - Config: Default manual trigger, no parameters  
    - Inputs: None  
    - Outputs: Connected to "Get new video" node  
    - Edge Cases: None  
    - Notes: Used to manually initiate the workflow.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automated periodic trigger (recommended every 5 minutes)  
    - Config: Interval set to minutes, no other filters  
    - Inputs: None  
    - Outputs: Not connected to the main flow in the JSON, but recommended for automation  
    - Edge Cases: None, but consider rate limits and overlapping runs.

  - **Get new video**  
    - Type: Google Sheets  
    - Role: Fetch rows where the "VIDEO" column is empty to find new video requests  
    - Config: Filters rows with empty VIDEO column, reads sheet "gid=0" from a specific doc  
    - Inputs: From manual trigger  
    - Outputs: Connects to "Set data"  
    - Credential: Google Sheets OAuth2  
    - Edge Cases: API quota errors, empty or malformed rows  

  - **Set data**  
    - Type: Set  
    - Role: Compose a prompt string combining "PROMPT" and "DURATION" for video creation  
    - Config: Assigns a single new field "prompt" with template:  
      ```  
      {{ $json.PROMPT }}  
      
      Duration of the video: {{ $json.DURATION }}  
      ```  
    - Inputs: From "Get new video"  
    - Outputs: To "Create Video" node  
    - Edge Cases: Missing PROMPT or DURATION fields could lead to incomplete prompt construction.

---

#### 2.2 Video Creation Request

- **Overview:**  
  Sends the constructed prompt to the Google Veo3 API to request video creation.

- **Nodes Involved:**  
  - Create Video (HTTP Request)  
  - Wait 60 sec. (Wait node)  

- **Node Details:**  

  - **Create Video**  
    - Type: HTTP Request  
    - Role: POST video generation request to Google Veo3 API  
    - Config:  
      - URL: https://queue.fal.run/fal-ai/veo3  
      - Method: POST  
      - Body: JSON with `{ "prompt": "<text>" }` from Set data node  
      - Headers: Content-Type application/json; Authorization header using Fal.run API key  
    - Inputs: From "Set data"  
    - Outputs: To "Wait 60 sec."  
    - Credential: HTTP Header Auth with API key  
    - Edge Cases: API errors, invalid prompt, auth failure, network timeout.

  - **Wait 60 sec.**  
    - Type: Wait  
    - Role: Pause workflow for 60 seconds before polling status  
    - Config: Wait for 60 seconds  
    - Inputs: From "Create Video"  
    - Outputs: To "Get status"  
    - Edge Cases: None; ensures API has time to process video.

---

#### 2.3 Video Processing Status Polling

- **Overview:**  
  Polls the Veo3 API repeatedly to check if the video creation is complete.

- **Nodes Involved:**  
  - Get status (HTTP Request)  
  - Completed? (If node)  
  - Wait 60 sec. (Wait node, same as above, looping)  

- **Node Details:**  

  - **Get status**  
    - Type: HTTP Request  
    - Role: GET status of video creation request using request_id from "Create Video"  
    - Config:  
      - URL templated with request_id: `https://queue.fal.run/fal-ai/veo3/requests/{{ request_id }}/status`  
      - Auth: Same API key as before  
    - Inputs: From "Wait 60 sec."  
    - Outputs: To "Completed?"  
    - Edge Cases: API errors, request_id missing, network issues.

  - **Completed?**  
    - Type: If  
    - Role: Check if status field equals "COMPLETED"  
    - Config: Compare `$json.status` to string "COMPLETED"  
    - Inputs: From "Get status"  
    - Outputs:  
      - True: To "Get Url Video" (retrieve video info)  
      - False: Loop back to "Wait 60 sec." to poll again  
    - Edge Cases: Status field missing or unexpected status values.

---

#### 2.4 Video Retrieval and Storage

- **Overview:**  
  Once video creation completes, retrieve the video URL, download the video file, and upload to Google Drive.

- **Nodes Involved:**  
  - Get Url Video (HTTP Request)  
  - Generate title (OpenAI via Langchain)  
  - Get File Video (HTTP Request)  
  - Upload Video (Google Drive)  

- **Node Details:**  

  - **Get Url Video**  
    - Type: HTTP Request  
    - Role: Fetch video metadata including the downloadable URL  
    - Config: GET request to `https://queue.fal.run/fal-ai/veo3/requests/{{ request_id }}`  
    - Inputs: From "Completed?" true branch  
    - Outputs: To "Generate title"  
    - Credential: Fal.run API Key  
    - Edge Cases: API errors, missing data.

  - **Generate title**  
    - Type: Langchain OpenAI node  
    - Role: Generate a SEO optimized YouTube title using GPT-4o-mini model  
    - Config:  
      - Prompt includes the original prompt text from Google Sheet  
      - System message instructs to generate catchy, SEO-friendly titles max 60 chars  
    - Inputs: From "Get Url Video"  
    - Outputs: To "Get File Video"  
    - Credential: OpenAI API key  
    - Edge Cases: API quota, model errors, unexpected output format.

  - **Get File Video**  
    - Type: HTTP Request  
    - Role: Download the video file from the Veo3 video URL  
    - Config: URL taken from `$('Get Url Video').item.json.video.url`  
    - Inputs: From "Generate title"  
    - Outputs: Two branches:  
      - To "Upload Video" (Google Drive upload)  
      - To "HTTP Request" (upload to YouTube via Upload-post.com)  
    - Edge Cases: Download failures, large file size, timeouts.

  - **Upload Video**  
    - Type: Google Drive  
    - Role: Upload the downloaded video file to a specified Google Drive folder  
    - Config:  
      - File name: timestamp + original file name  
      - Folder ID: specific Google Drive folder for Fal.run videos  
      - Drive ID: "My Drive"  
    - Inputs: From "Get File Video"  
    - Outputs: To "Update result" (update Google Sheet video URL)  
    - Credential: Google Drive OAuth2  
    - Edge Cases: Upload failures, permission errors, quota limits.

---

#### 2.5 YouTube Title Generation and Upload

- **Overview:**  
  Upload the video to YouTube using Upload-post.com API with the generated title, then update the Google Sheet with the YouTube URL.

- **Nodes Involved:**  
  - HTTP Request (Upload-post.com API)  
  - Update Youtube URL (Google Sheets)  

- **Node Details:**  

  - **HTTP Request (Upload-post.com)**  
    - Type: HTTP Request  
    - Role: Upload video file to YouTube via Upload-post.com API  
    - Config:  
      - URL: https://api.upload-post.com/api/upload  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameters:  
        - title: from "Generate title" content  
        - user: YOUR_USERNAME (must be replaced by user)  
        - platform[]: "youtube"  
        - video: binary data from downloaded file  
      - Auth: Header Auth with Upload-post.com API key  
    - Inputs: From "Get File Video"  
    - Outputs: To "Update Youtube URL"  
    - Edge Cases: Auth failure, file upload errors, API limits.

  - **Update Youtube URL**  
    - Type: Google Sheets  
    - Role: Update the Google Sheet row with the YouTube video URL  
    - Config:  
      - Matching column: row_number  
      - Updates "YOUTUBE_URL" with `https://youtu.be/{{ video_id }}` from upload response  
      - Clears VIDEO column value (set to empty string)  
    - Inputs: From "HTTP Request" upload response  
    - Credential: Google Sheets OAuth2  
    - Edge Cases: Sheet update conflicts, API errors, malformed video_id.

---

#### 2.6 Google Sheet Updates

- **Overview:**  
  Updates the Google Sheet to track video URL and YouTube link as the workflow progresses.

- **Nodes Involved:**  
  - Update result (Google Sheets)  
  - Update Youtube URL (Google Sheets)  

- **Node Details:**  

  - **Update result**  
    - Type: Google Sheets  
    - Role: Update the "VIDEO" column with the Google Drive video URL  
    - Config:  
      - Matching on row_number  
      - Sets "VIDEO" to the URL from Google Drive upload  
    - Inputs: From "Upload Video"  
    - Credential: Google Sheets OAuth2  
    - Edge Cases: API errors, concurrency issues.

  - **Update Youtube URL**  
    - Refer to 2.5 section (also updates YOUTUBE_URL column)

---

### 3. Summary Table

| Node Name              | Node Type                     | Functional Role                                          | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                     |
|------------------------|-------------------------------|----------------------------------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                | Manual start trigger                                      | None                         | Get new video               |                                                                                                |
| Schedule Trigger       | Schedule Trigger              | Scheduled start trigger                                   | None                         | None (recommended)           | "Start the workflow manually or periodically by hooking the \"Schedule Trigger\" node."         |
| Get new video          | Google Sheets                 | Fetch new video requests (rows with empty VIDEO)         | When clicking ‘Test workflow’ | Set data                    |                                                                                                |
| Set data               | Set                          | Compose prompt with duration                              | Get new video                | Create Video                |                                                                                                |
| Create Video           | HTTP Request                 | Send video creation request to Google Veo3 API           | Set data                    | Wait 60 sec.                | "Set API Key created in Step 2"                                                                 |
| Wait 60 sec.           | Wait                         | Wait before polling status                                | Create Video, Get status     | Get status, Completed?      |                                                                                                |
| Get status             | HTTP Request                 | Poll video creation status                                | Wait 60 sec.                | Completed?                  |                                                                                                |
| Completed?             | If                           | Check if video creation is completed                      | Get status                  | Get Url Video (true), Wait 60 sec. (false) |                                                                                                |
| Get Url Video          | HTTP Request                 | Get video metadata and URL                                | Completed? (true)            | Generate title              |                                                                                                |
| Generate title         | OpenAI (Langchain)           | Generate optimized YouTube title                          | Get Url Video               | Get File Video              |                                                                                                |
| Get File Video         | HTTP Request                 | Download video file                                       | Generate title              | Upload Video, HTTP Request  |                                                                                                |
| Upload Video           | Google Drive                 | Upload video to Google Drive                              | Get File Video              | Update result               |                                                                                                |
| Update result          | Google Sheets                | Update Google Sheet VIDEO column with Drive URL          | Upload Video                | None                       |                                                                                                |
| HTTP Request           | HTTP Request                 | Upload video to YouTube via Upload-post.com API          | Get File Video              | Update Youtube URL          | "Set YOUR_USERNAME in Step 3"                                                                    |
| Update Youtube URL     | Google Sheets                | Update Google Sheet YOUTUBE_URL column                    | HTTP Request (upload)       | None                       |                                                                                                |
| Sticky Note3           | Sticky Note                  | Overview: Workflow description                            | None                       | None                       | "# Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube\n\nThis workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube**. The entire process is triggered from a Google Sheet that acts as the central interface for input and output.\n\nIT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube." |
| Sticky Note4           | Sticky Note                  | Google Sheet setup instructions                           | None                       | None                       | "Create a [Google Sheet like this](https://docs.google.com/spreadsheets/d/1pcoY9N_vQp44NtSRR5eskkL5Qd0N0BGq7Jh_4m-7VEQ/edit?usp=sharing).\n\nPlease insert:\n- in the \"PROMPT\" column the accurate description of the video you want to create\n- in the \"DURATION\" column the lenght of the video you want to create\n\nLeave the \"VIDEO\" column unfilled. It will be inserted by the system once the video has been created" |
| Sticky Note5           | Sticky Note                  | Workflow start recommendation                            | None                       | None                       | "Start the workflow manually or periodically by hooking the \"Schedule Trigger\" node. It is recommended to set it at 5 minute intervals." |
| Sticky Note6           | Sticky Note                  | API key instruction for Fal.run                          | None                       | None                       | "Create an account [here](https://fal.ai/) and obtain API KEY.\nIn the node \"Create Image\" set \"Header Auth\" and set:\n- Name: \"Authorization\"\n- Value: \"Key YOURAPIKEY\"" |
| Sticky Note7           | Sticky Note                  | Reminder for API key setup                               | None                       | None                       | "Set API Key created in Step 2"                                                                 |
| Sticky Note8           | Sticky Note                  | Upload-Post.com YouTube upload setup                     | None                       | None                       | "Find your API key in your [Upload-Post Manage Api Keys](https://app.upload-post.com/) 10 FREE uploads per month\n- Set the the \"Auth Header\":\n-- Name: Authorization\n-- Value: Apikey YOUR_API_KEY_HERE\n- Create profiles to manage your social media accounts. The \"Profile\" you choose will be used in the field YOUR_USRNAME (eg. test1 or test2)." |
| Sticky Note            | Sticky Note                  | Reminder to set YOUR_USERNAME in Upload-post.com API    | None                       | None                       | "Set YOUR_USERNAME in Step 3"                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheet** as the input/output interface:  
   - Columns: PROMPT (video description), DURATION (video length), VIDEO (output video URL), YOUTUBE_URL (uploaded video link), row_number (read-only)  
   - Use the template linked in Sticky Note4.

2. **Create credentials:**  
   - Fal.run API key as HTTP Header Auth with name "Authorization" and value "Key YOURAPIKEY"  
   - Google Sheets OAuth2 with access to the created spreadsheet  
   - Google Drive OAuth2 with upload permission to target folder  
   - OpenAI API key for GPT-4o-mini model  
   - Upload-post.com API key as HTTP Header Auth with "Authorization" and value "Apikey YOUR_API_KEY_HERE"

3. **Create nodes in order:**

   - **Manual Trigger** ("When clicking ‘Test workflow’"): no config  
   - **Google Sheets** ("Get new video"):  
     - Operation: Read rows with VIDEO column empty from your Sheet, sheetName gid=0  
     - Credentials: Google Sheets OAuth2

   - **Set node** ("Set data"):  
     - Assign one field named `prompt` with expression:  
       ```
       {{$json.PROMPT}}\n\nDuration of the video: {{$json.DURATION}}
       ```

   - **HTTP Request** ("Create Video"):  
     - Method: POST  
     - URL: https://queue.fal.run/fal-ai/veo3  
     - Body type: JSON  
     - Body: `{ "prompt": "={{$json.prompt}}" }`  
     - Headers: Content-Type application/json  
     - Auth: HTTP Header Auth (Fal.run API key)

   - **Wait** ("Wait 60 sec."): 60 seconds delay

   - **HTTP Request** ("Get status"):  
     - Method: GET  
     - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $('Create Video').item.json.request_id }}/status`  
     - Auth: HTTP Header Auth (Fal.run API key)

   - **If** ("Completed?"):  
     - Condition: Check if `$json.status` equals "COMPLETED" (string strict)  
     - True path: to "Get Url Video"  
     - False path: back to "Wait 60 sec." (loop)

   - **HTTP Request** ("Get Url Video"):  
     - Method: GET  
     - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}`  
     - Auth: HTTP Header Auth (Fal.run API key)

   - **OpenAI (Langchain)** ("Generate title"):  
     - Model: gpt-4o-mini  
     - Messages:  
       - User content: `Input: {{ $('Get new video').item.json.PROMPT }}`  
       - System content: Instructions to generate SEO-optimized YouTube title max 60 chars, same language  
     - Credentials: OpenAI API key

   - **HTTP Request** ("Get File Video"):  
     - Method: GET  
     - URL: `{{ $('Get Url Video').item.json.video.url }}`  
     - Download video content as binary

   - **Google Drive** ("Upload Video"):  
     - Name: `{{ $now.format('yyyyLLddHHmmss') }}-{{ $('Get Url Video').item.json.video.file_name }}`  
     - Folder ID: target Google Drive folder ID  
     - Drive ID: "My Drive"  
     - Credentials: Google Drive OAuth2

   - **Google Sheets** ("Update result"):  
     - Operation: Update  
     - Update VIDEO column with Google Drive file URL  
     - Match on row_number from "Get new video"  
     - Credentials: Google Sheets OAuth2

   - **HTTP Request** ("HTTP Request" for Upload-post):  
     - Method: POST  
     - URL: https://api.upload-post.com/api/upload  
     - Content-Type: multipart/form-data  
     - Body parameters:  
       - title: from "Generate title" response content  
       - user: YOUR_USERNAME (replace with your Upload-post profile name)  
       - platform[]: "youtube"  
       - video: binary data from "Get File Video"  
     - Auth: HTTP Header Auth (Upload-post.com API key)

   - **Google Sheets** ("Update Youtube URL"):  
     - Operation: Update  
     - Update YOUTUBE_URL column with YouTube link: `https://youtu.be/{{ $json.results.youtube.video_id }}`  
     - Clear VIDEO column (set empty string)  
     - Match on row_number  
     - Credentials: Google Sheets OAuth2

4. **Connect nodes as per the flow:**  
   Manual Trigger → Get new video → Set data → Create Video → Wait 60 sec. → Get status → Completed?  
   Completed? true → Get Url Video → Generate title → Get File Video → Upload Video → Update result  
   Get File Video → HTTP Request (Upload-post) → Update Youtube URL  
   Completed? false → Wait 60 sec.

5. **Test thoroughly:**  
   - Check API keys and permissions  
   - Validate Google Sheet formatting  
   - Monitor API rate limits and errors  
   - Confirm video uploads and URLs update correctly

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                           | Context or Link                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Create a [Google Sheet like this](https://docs.google.com/spreadsheets/d/1pcoY9N_vQp44NtSRR5eskkL5Qd0N0BGq7Jh_4m-7VEQ/edit?usp=sharing) with PROMPT, DURATION, VIDEO, YOUTUBE_URL columns.                                                                                     | Google Sheet input/output template                                                                                            |
| Create an account [here](https://fal.ai/) and obtain API key for Google Veo3 video generation.                                                                                                                                                                         | Fal.run API registration and key setup                                                                                        |
| Upload-post.com allows 10 free uploads per month for YouTube video upload automation. Get API key and manage social profiles at [Upload-Post Manage Api Keys](https://app.upload-post.com/).                                                                             | Upload-post.com API usage and quota limits                                                                                    |
| Use OpenAI GPT-4o-mini model with system prompt to generate optimized YouTube video titles in the same language as the video prompt, max 60 characters for SEO.                                                                                                       | OpenAI GPT model and prompt design                                                                                            |
| Recommended to trigger workflow every 5 minutes via Schedule Trigger node for continuous processing of Google Sheets entries.                                                                                                                                           | Schedule Trigger best practice                                                                                               |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.