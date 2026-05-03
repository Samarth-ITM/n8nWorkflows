Generate cheap Viral AI Videos to TikTok with Google Veo3 Fast and Postiz

https://n8nworkflows.xyz/workflows/generate-cheap-viral-ai-videos-to-tiktok-with-google-veo3-fast-and-postiz-7420


# Generate cheap Viral AI Videos to TikTok with Google Veo3 Fast and Postiz

### 1. Workflow Overview

This workflow automates the generation of low-cost viral AI videos using the Google Veo3 Fast AI model, stores the created videos on Google Drive, generates optimized YouTube titles using GPT-4o, and uploads the videos to TikTok via Postiz. It is designed for content creators or marketers seeking to automate video production and social media posting efficiently.

The workflow is structured in the following logical blocks:

- **1.1 Input Reception and Triggering**: Receives triggers manually or periodically to start processing new video requests from a Google Sheet.
- **1.2 Video Request Initialization**: Fetches new video requests from the Google Sheet and prepares the prompt for video creation.
- **1.3 Video Generation and Status Polling**: Sends requests to the Google Veo3 Fast API, polls for completion status, and handles timing/retries.
- **1.4 Video Retrieval and Storage**: Downloads the completed video file and uploads it to Google Drive.
- **1.5 Title Generation**: Uses GPT-4o via LangChain to generate engaging YouTube-optimized video titles based on the prompt.
- **1.6 TikTok Upload via Postiz**: Uploads the video to TikTok through Postiz API with the generated title as content.
- **1.7 Google Sheet Update**: Updates the Google Sheet row with the final video URL once processing is complete.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

**Overview:**  
This block enables starting the workflow manually or on a scheduled interval, ensuring new video requests in the Google Sheet are picked up regularly.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Manually initiates the workflow for immediate testing or execution.  
  - Inputs: None  
  - Outputs: Connected to “Get new video” for fetching new video requests.  
  - Edge cases: Trigger failure is unlikely but workflow execution depends on downstream node availability.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Periodically triggers the workflow every minute as configured.  
  - Configuration: Interval set to every 1 minute.  
  - Inputs: None  
  - Outputs: No direct connection in JSON but logically serves as an alternative trigger option.  
  - Edge cases: Scheduling errors may occur if n8n server time or settings are misconfigured.

---

#### 2.2 Video Request Initialization

**Overview:**  
Fetches new video requests from the Google Sheet where the “VIDEO” column is empty, then formats the prompt for video creation.

**Nodes Involved:**  
- Get new video (Google Sheets)  
- Set data  

**Node Details:**  

- **Get new video**  
  - Type: Google Sheets (read)  
  - Role: Queries Google Sheet for rows where the VIDEO column is empty, indicating new requests.  
  - Configuration: Document and sheet specified; OAuth2 credentials for Google Sheets used.  
  - Inputs: Trigger from manual or schedule trigger node.  
  - Outputs: Passes filtered rows to “Set data”.  
  - Edge cases: API quota limits, authorization errors, or no matching rows found.

- **Set data**  
  - Type: Set  
  - Role: Constructs the JSON payload with a prompt combining the original PROMPT and DURATION from the sheet.  
  - Key expression:  
    ```
    prompt = PROMPT + "\n\nDuration of the video: " + DURATION
    ```  
  - Inputs: Receives rows from “Get new video”.  
  - Outputs: Connects to “Create Video” node.  
  - Edge cases: Missing or malformed PROMPT or DURATION fields.

---

#### 2.3 Video Generation and Status Polling

**Overview:**  
Submits the video creation request to the Google Veo3 Fast API and polls periodically until the video generation is completed.

**Nodes Involved:**  
- Create Video (HTTP Request)  
- Wait 60 sec. (Wait)  
- Get status (HTTP Request)  
- Completed? (If)  

**Node Details:**  

- **Create Video**  
  - Type: HTTP Request  
  - Role: Sends POST request to Veo3 Fast API with prompt to start video generation.  
  - Configuration: URL `https://queue.fal.run/fal-ai/veo3/fast`, header auth with API key, JSON body containing the prompt.  
  - Inputs: Receives prompt from “Set data”.  
  - Outputs: Passes request_id to “Wait 60 sec.”.  
  - Edge cases: API authentication failure, invalid prompt format, network errors.

- **Wait 60 sec.**  
  - Type: Wait  
  - Role: Delays workflow for 60 seconds to allow video processing.  
  - Inputs: Receives request_id from “Create Video”.  
  - Outputs: Proceeds to “Get status”.  
  - Edge cases: Delays may cause workflow timeout if downstream nodes take too long.

- **Get status**  
  - Type: HTTP Request  
  - Role: Polls Veo3 Fast API for the status of video request using request_id.  
  - Configuration: GET request to `https://queue.fal.run/fal-ai/veo3/requests/{{request_id}}/status` with header authentication.  
  - Inputs: Receives request_id from “Wait 60 sec.”.  
  - Outputs: Passes status JSON to “Completed?”.  
  - Edge cases: API downtime, invalid request_id, response parsing errors.

- **Completed?**  
  - Type: If  
  - Role: Checks if status returned is “COMPLETED”.  
  - Configuration: Condition equals `status == "COMPLETED"`.  
  - Inputs: Receives status from “Get status”.  
  - Outputs:  
    - If true: proceeds to “Get Url Video” to fetch video details.  
    - If false: loops back to “Wait 60 sec.” to poll again.  
  - Edge cases: Status may never become “COMPLETED” (stuck requests), causing infinite loop or timeout.

---

#### 2.4 Video Retrieval and Storage

**Overview:**  
Retrieves the video URL once generation is complete, downloads the video file, and uploads it to Google Drive for storage.

**Nodes Involved:**  
- Get Url Video (HTTP Request)  
- Generate title (OpenAI)  
- Get File Video (HTTP Request)  
- Upload Video (Google Drive)  

**Node Details:**  

- **Get Url Video**  
  - Type: HTTP Request  
  - Role: Retrieves detailed video metadata including video URL using request_id.  
  - Configuration: GET request to `https://queue.fal.run/fal-ai/veo3/requests/{{ request_id }}` with header auth.  
  - Inputs: From “Completed?” node.  
  - Outputs: Passes video URL to “Generate title”.  
  - Edge cases: API errors, invalid request_id, malformed response.

- **Generate title**  
  - Type: OpenAI (LangChain)  
  - Role: Generates an SEO optimized YouTube video title based on the prompt from Google Sheet.  
  - Configuration: Model GPT-4o-mini; system prompt instructs SEO best practices and formatting.  
  - Inputs: Receives prompt from “Get Url Video”.  
  - Outputs: Passes title content to “Get File Video”.  
  - Edge cases: API quota limits, prompt malformation, language mismatches.

- **Get File Video**  
  - Type: HTTP Request  
  - Role: Downloads the video file from the URL provided by “Get Url Video”.  
  - Configuration: GET request to video URL.  
  - Inputs: Receives URL from “Generate title”.  
  - Outputs: Passes binary video data to both “Upload Video” and “Upload Video to Postiz”.  
  - Edge cases: Download failures, URL expiry, file size issues.

- **Upload Video**  
  - Type: Google Drive  
  - Role: Uploads the downloaded video file to a specific Google Drive folder for archival and access.  
  - Configuration: Uses OAuth2 credentials; names file with timestamp and original filename; uploads to a specified folder.  
  - Inputs: Receives binary data from “Get File Video”.  
  - Outputs: Passes metadata to “Update result”.  
  - Edge cases: Google Drive API quota, insufficient permissions, folder not found.

---

#### 2.5 TikTok Upload via Postiz

**Overview:**  
Uploads the video file to TikTok using Postiz API, including the generated title as the video caption.

**Nodes Involved:**  
- Upload Video to Postiz (HTTP Request)  
- TikTok (Postiz node)  

**Node Details:**  

- **Upload Video to Postiz**  
  - Type: HTTP Request (multipart/form-data)  
  - Role: Uploads the video binary to Postiz’s upload endpoint.  
  - Configuration: POST to `https://api.postiz.com/public/v1/upload`, multipart form with binary field “file”, header authentication.  
  - Inputs: Receives video binary from “Get File Video”.  
  - Outputs: Passes uploaded file metadata (id, path) to “TikTok” node.  
  - Edge cases: API key errors, network timeouts, file size limits.

- **TikTok**  
  - Type: Postiz integration node  
  - Role: Creates a TikTok post using the uploaded video and generated title.  
  - Configuration: Uses Postiz API credentials; posts with current datetime; includes video id and path; integrationId set to user’s TikTok channel; short link enabled.  
  - Inputs: Receives file metadata and title content from “Upload Video to Postiz” and “Generate title”.  
  - Outputs: Finalizes the posting process.  
  - Edge cases: Invalid channel ID, API quota, content restrictions on TikTok, failed post creation.

---

#### 2.6 Google Sheet Update

**Overview:**  
Updates the original Google Sheet row with the generated video URL to track completion.

**Nodes Involved:**  
- Update result (Google Sheets)  

**Node Details:**  

- **Update result**  
  - Type: Google Sheets (update)  
  - Role: Updates the “VIDEO” column in the original Google Sheet row with the URL of the uploaded video on Google Drive.  
  - Configuration: Matches rows by “row_number” from the sheet; updates “VIDEO” column with video URL from “Get Url Video”.  
  - Inputs: Receives video URL from “Upload Video” node output.  
  - Outputs: None (workflow end).  
  - Edge cases: Row mismatch, API quota, permission issues.

---

### 3. Summary Table

| Node Name              | Node Type                | Functional Role                               | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                  |
|------------------------|--------------------------|-----------------------------------------------|-----------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger           | Manual start trigger                          |                             | Get new video                  |                                                                                              |
| Schedule Trigger        | Schedule Trigger          | Periodic trigger                              |                             | (Implicit start)               | ## STEP 4 - MAIN FLOW: Recommended 5 min intervals                                          |
| Get new video           | Google Sheets             | Fetch new video requests where VIDEO empty   | When clicking ‘Test workflow’| Set data                      |                                                                                              |
| Set data               | Set                      | Prepare prompt payload for video creation    | Get new video                | Create Video                  |                                                                                              |
| Create Video            | HTTP Request             | Submit video generation request to Veo3 Fast | Set data                    | Wait 60 sec.                  | Set API Key created in Step 2                                                                |
| Wait 60 sec.            | Wait                     | Delay 60 seconds between status polls        | Create Video / Completed?    | Get status                    |                                                                                              |
| Get status              | HTTP Request             | Poll Veo3 Fast API for video generation status | Wait 60 sec.                | Completed?                    | Set API Key created in Step 2                                                                |
| Completed?              | If                       | Check if video generation is complete        | Get status                  | Get Url Video / Wait 60 sec.  |                                                                                              |
| Get Url Video           | HTTP Request             | Retrieve video URL and metadata               | Completed?                  | Generate title                |                                                                                              |
| Generate title          | OpenAI (LangChain)       | Generate YouTube SEO optimized video title   | Get Url Video               | Get File Video                |                                                                                              |
| Get File Video          | HTTP Request             | Download video file from URL                   | Generate title              | Upload Video / Upload Video to Postiz |                                                                                              |
| Upload Video            | Google Drive             | Upload video to Google Drive for storage     | Get File Video              | Update result                 |                                                                                              |
| Update result           | Google Sheets            | Update Google Sheet row with video URL       | Upload Video                |                               |                                                                                              |
| Upload Video to Postiz  | HTTP Request             | Upload video binary to Postiz platform        | Get File Video              | TikTok                       | ## STEP 3 - Upload video on TikTok with Postiz; set API Key and ChannelId                    |
| TikTok                 | Postiz                   | Create and post TikTok video via Postiz API  | Upload Video to Postiz      |                               | ## STEP 3 - Upload video on TikTok with Postiz; set API Key and ChannelId                    |
| Sticky Note3            | Sticky Note              | Overview and workflow purpose                  |                             |                               | # Generate Cheaper AI Videos (with audio), using Veo3 Fast and Upload to TikTok              |
| Sticky Note4            | Sticky Note              | Instructions for Google Sheet setup            |                             |                               | ## STEP 1 - GOOGLE SHEET; link to example sheet                                             |
| Sticky Note5            | Sticky Note              | Instruction on triggering workflow             |                             |                               | ## STEP 4 - MAIN FLOW: Start manually or schedule trigger                                   |
| Sticky Note6            | Sticky Note              | Instructions for API key setup                  |                             |                               | ## STEP 2 - GET API KEY (YOURAPIKEY); link to fal.ai                                        |
| Sticky Note7            | Sticky Note              | Reminder to set API Key                          |                             |                               | Set API Key created in Step 2                                                               |
| Sticky Note8            | Sticky Note              | Instructions for TikTok upload via Postiz      |                             |                               | ## STEP 3 - Upload video on TikTok; setup Postiz account and ChannelId                       |
| Sticky Note             | Sticky Note              | Reminder for ChannelId setup                     |                             |                               | Set ChannelId Step 3                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named “When clicking ‘Test workflow’”.  
   - Add a **Schedule Trigger** node named “Schedule Trigger”, set to run every 1 minute (or preferred interval).

2. **Google Sheets Node to Get New Videos**  
   - Add a **Google Sheets** node named “Get new video”.  
   - Configure to read from your Google Sheet (document ID and sheet name).  
   - Use OAuth2 credentials for Google Sheets API.  
   - Set a filter to select rows where the “VIDEO” column is empty (indicating new requests).

3. **Set Node to Prepare Prompt**  
   - Add a **Set** node named “Set data”.  
   - Create a new string field named `prompt`.  
   - Set its value using an expression combining PROMPT and DURATION columns:  
     ```
     {{$json.PROMPT}}\n\nDuration of the video: {{$json.DURATION}}
     ```
   - Connect “Get new video” output to this node.

4. **HTTP Request Node to Create Video**  
   - Add an **HTTP Request** node named “Create Video”.  
   - POST to `https://queue.fal.run/fal-ai/veo3/fast`.  
   - Set header “Content-Type: application/json” and use HTTP Header Auth with your API key from fal.ai.  
   - Set JSON body to:  
     ```
     {
       "prompt": "{{$json.prompt}}"
     }
     ```
   - Connect “Set data” output here.

5. **Wait Node for 60 Seconds**  
   - Add a **Wait** node named “Wait 60 sec.”.  
   - Set to wait for 60 seconds.  
   - Connect “Create Video” output here.

6. **HTTP Request Node to Get Status**  
   - Add an **HTTP Request** node named “Get status”.  
   - GET from `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}/status`.  
   - Use same HTTP Header Auth credentials.  
   - Connect “Wait 60 sec.” output here.

7. **If Node to Check Completion**  
   - Add an **If** node named “Completed?”.  
   - Condition: Check if `{{$json.status}}` equals “COMPLETED”.  
   - Connect “Get status” output here.  
   - If true, connect to next step “Get Url Video”.  
   - If false, connect back to “Wait 60 sec.” to poll again.

8. **HTTP Request Node to Get Video URL**  
   - Add an **HTTP Request** node named “Get Url Video”.  
   - GET from `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}`.  
   - Use same HTTP Header Auth credentials.  
   - Connect “Completed?” true output here.

9. **OpenAI Node to Generate Title**  
   - Add a **LangChain OpenAI** node named “Generate title”.  
   - Select model `gpt-4o-mini`.  
   - Set system prompt instructing SEO optimized YouTube title generation per the guidelines in the original workflow.  
   - Pass input prompt from “Get Url Video” node.  
   - Connect “Get Url Video” output here.

10. **HTTP Request Node to Download Video File**  
    - Add an **HTTP Request** node named “Get File Video”.  
    - GET from the URL `{{$json.video.url}}` received from “Get Url Video”.  
    - Connect “Generate title” output here.

11. **Google Drive Node to Upload Video**  
    - Add a **Google Drive** node named “Upload Video”.  
    - Use OAuth2 credentials for Google Drive.  
    - Configure upload folder ID to your target folder.  
    - Name file with timestamp + original filename:  
      ```
      {{$now.format('yyyyLLddHHmmss')}}-{{$json.video.file_name}}
      ```  
    - Connect “Get File Video” output here.

12. **HTTP Request Node to Upload Video to Postiz**  
    - Add an **HTTP Request** node named “Upload Video to Postiz”.  
    - POST to `https://api.postiz.com/public/v1/upload` with multipart form-data, binary field named “file” containing the video.  
    - Use HTTP Header Auth with your Postiz API key.  
    - Connect “Get File Video” output here.

13. **Postiz Node to Post to TikTok**  
    - Add a **Postiz** node named “TikTok”.  
    - Authenticate with Postiz API credentials.  
    - Configure post content with the title generated by “Generate title”.  
    - Set current date/time.  
    - Set integrationId to your TikTok channel ID (replace “XXX” with your actual channel).  
    - Connect “Upload Video to Postiz” output here.

14. **Google Sheets Node to Update Video Column**  
    - Add a **Google Sheets** node named “Update result”.  
    - Configure to update the row number matched by original sheet row number.  
    - Update the “VIDEO” column with the Google Drive URL from “Upload Video”.  
    - Connect “Upload Video” output here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                           | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow uses the Google Veo3 Fast AI model for video generation, which is a cost-effective alternative for AI video creation.                                                                                     | Workflow overview sticky note                                                                                   |
| Google Sheet setup is critical; see example sheet here: https://docs.google.com/spreadsheets/d/1pcoY9N_vQp44NtSRR5eskkL5Qd0N0BGq7Jh_4m-7VEQ/edit?usp=sharing                                                               | Sticky Note4                                                                                                    |
| Obtain API key from fal.ai (https://fal.ai/) and configure HTTP header authentication with “Authorization: Key YOURAPIKEY” in HTTP Request nodes.                                                                      | Sticky Note6                                                                                                    |
| For TikTok uploads, create a Postiz account (https://postiz.com/?ref=n3witalia) and connect your TikTok channel in Postiz. Use the channel ID in the TikTok node configuration.                                        | Sticky Note8                                                                                                    |
| This workflow requires community nodes only supported in self-hosted n8n instances.                                                                                                                                    | Sticky Note3                                                                                                    |
| Recommended workflow triggering interval is every 5 minutes for production stability.                                                                                                                                 | Sticky Note5                                                                                                    |
| Use consistent credential storage for Google Sheets, Google Drive, fal.ai API, and Postiz API to avoid authentication errors.                                                                                          | General best practice                                                                                            |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created using n8n, an integration and automation tool. All processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.