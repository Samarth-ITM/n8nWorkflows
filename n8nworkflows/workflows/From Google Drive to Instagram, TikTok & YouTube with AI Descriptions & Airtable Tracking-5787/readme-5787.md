From Google Drive to Instagram, TikTok & YouTube with AI Descriptions & Airtable Tracking

https://n8nworkflows.xyz/workflows/from-google-drive-to-instagram--tiktok---youtube-with-ai-descriptions---airtable-tracking-5787


# From Google Drive to Instagram, TikTok & YouTube with AI Descriptions & Airtable Tracking

### 1. Workflow Overview

This workflow automates the process of uploading videos placed in a specific Google Drive folder to social media platforms Instagram, TikTok, and YouTube. It leverages AI to generate engaging video descriptions based on extracted audio content, tracks upload status in Airtable, and provides error notification via Telegram. The workflow is designed for content creators, digital marketers, or brands aiming to streamline multi-platform video publishing with descriptive AI content and centralized tracking.

Logical blocks:

- **1.1 Input Reception and Initialization**: Detect new video files in Google Drive and initialize variables.
- **1.2 Video Download and Audio Extraction**: Download video, extract audio for transcription.
- **1.3 AI Description Generation**: Use OpenAI to create video descriptions from audio transcription.
- **1.4 Airtable Record Management**: Create and update Airtable records with video metadata and descriptions.
- **1.5 Video Reading and Uploading**: Prepare video files for each platform and upload via API.
- **1.6 Status Update in Airtable**: Update Airtable records with upload status and URLs.
- **1.7 Error Handling and Notification**: Catch errors and notify via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
Monitors a specific Google Drive folder for new video files and sets up essential variables like Airtable app and table IDs and user credentials for uploads.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - Google Drive (download)  
  - Set Variables  
  - Create Airtable Record

- **Node Details:**

  - **Google Drive Trigger**  
    - Type: Trigger node for Google Drive  
    - Role: Watches a specific folder for newly created files every minute  
    - Config: Folder ID monitored is `"18m0i341QLQuyWuHv_FBdz8-r-QDtofYm"`  
    - Input: None (trigger)  
    - Output: JSON metadata of new files  
    - Potential Failures: Google Drive API auth errors, network issues, polling delays  

  - **Google Drive (download)**  
    - Type: Standard Google Drive node  
    - Role: Downloads the newly created file using its ID from trigger  
    - Config: OAuth2 authentication, fileId dynamically set from trigger output  
    - Input: File metadata from trigger  
    - Output: Binary video file plus metadata  
    - Potential Failures: File access permission issues, network failures, retry enabled with 5s intervals  

  - **Set Variables**  
    - Type: Set node  
    - Role: Stores static configuration variables such as Airtable app ID, table ID, and upload user name  
    - Config:  
      - `airtable_app_id`, `airtable_table_id` to be replaced by user with actual IDs  
      - `upload_post_user` set to `"test2"` by default  
    - Input: None (start of flow)  
    - Output: Variables as JSON for downstream nodes  
    - Edge Cases: User must replace placeholder values before running  

  - **Create Airtable Record**  
    - Type: Airtable node  
    - Role: Creates a new record for the video in Airtable to track the upload lifecycle  
    - Config:  
      - Uses app and table IDs from "Set Variables" node  
      - Adds initial field: "Video Name"  
      - Uses API token authentication  
    - Input: Variables with video metadata  
    - Output: Airtable record ID and metadata  
    - Potential Failures: Airtable API rate limits, invalid credentials  

#### 1.2 Video Download and Audio Extraction

- **Overview:**  
Downloads the video locally, extracts audio from the video, and transcribes it to text using OpenAI.

- **Nodes Involved:**  
  - Read video from Google Drive (write binary file)  
  - Get Audio from Video (OpenAI transcription)  

- **Node Details:**

  - **Read video from Google Drive**  
    - Type: Write Binary File  
    - Role: Saves video binary content locally, filename sanitized by replacing spaces with underscores  
    - Input: Binary video output from Google Drive node  
    - Output: File saved locally with filename usable by subsequent nodes  
    - Potential Failures: File system permissions, disk space  

  - **Get Audio from Video**  
    - Type: OpenAI node (Langchain)  
    - Role: Transcribes audio from the video file to text  
    - Config:  
      - Resource set to "audio" and operation to "transcribe"  
      - Uses OpenAI API key credential  
    - Input: Local video file binary (implicitly handled)  
    - Output: Transcribed text in JSON field `text`  
    - Edge Cases: Audio too noisy or low quality, API limits or errors, retry enabled  

#### 1.3 AI Description Generation

- **Overview:**  
Generates an engaging social media description for the video based on the transcribed audio text using GPT-4o.

- **Nodes Involved:**  
  - Generate Description for Videos (OpenAI)

- **Node Details:**

  - **Generate Description for Videos**  
    - Type: OpenAI node (Langchain)  
    - Role: Requests GPT-4o to create a social media post description using examples and the extracted audio text  
    - Config:  
      - System message: Defines role as expert assistant  
      - User message: Provides example Instagram descriptions and injects the transcribed audio text dynamically from `Get Audio from Video`  
      - Response expected: Plain text description only  
      - Uses OpenAI API key  
    - Input: Audio transcription text  
    - Output: Generated description text in `message.content`  
    - Edge Cases: API failure, unexpected response format, retry enabled  

#### 1.4 Airtable Record Management

- **Overview:**  
Updates the Airtable record with the generated description, video metadata, and upload timestamp.

- **Nodes Involved:**  
  - Edit Airtable Fields1 (Set)  
  - Update Airtable with Description (Airtable)

- **Node Details:**

  - **Edit Airtable Fields1**  
    - Type: Set node  
    - Role: Prepares fields to update Airtable record: ID, Description, Video Name, Google Drive Links, Upload Date  
    - Config: Values dynamically set from previous nodes including description text and video metadata  
    - Input: Airtable record ID and description from previous nodes  
    - Output: Fields JSON for Airtable update  
    - Edge Cases: Missing data from previous nodes  

  - **Update Airtable with Description**  
    - Type: Airtable node  
    - Role: Updates the existing Airtable record with the prepared fields  
    - Config: App and table IDs from variables, update by record ID  
    - Input: Fields from "Edit Airtable Fields1"  
    - Output: Updated Airtable record metadata  
    - Potential Failures: API errors, missing or invalid record ID  

#### 1.5 Video Reading and Uploading

- **Overview:**  
Reads the video file for each platform, uploads it using the upload-post.com API, and prepares the upload status for Airtable.

- **Nodes Involved:**  
  - Read Video for TikTok (Read Binary File)  
  - Upload Video to TikTok (HTTP Request)  
  - Edit Airtable Fields (Set)

  - Read Video for Instagram (Read Binary File)  
  - Upload Video to Instagram (HTTP Request)  
  - Edit Airtable Fields 2 (Set)

  - Read Video for YouTube (Read Binary File)  
  - Upload Video to YouTube (HTTP Request)  
  - Edit Airtable Fields 3 (Set)

- **Node Details:**

  - **Read Video for TikTok / Instagram / YouTube**  
    - Type: Read Binary File  
    - Role: Reads the locally saved video file for uploading  
    - Config: Uses filename from "Read video from Google Drive" with spaces replaced by underscores  
    - Input: Local file path  
    - Output: Binary data in `datavideo` property  
    - Edge Cases: File not found, permission issues  

  - **Upload Video to TikTok / Instagram / YouTube**  
    - Type: HTTP Request  
    - Role: Uploads video to upload-post.com API specifying platform and title  
    - Config:  
      - POST to `https://api.upload-post.com/api/upload`  
      - Multipart form data includes:  
        - Title: generated description (YouTube truncated to 70 chars)  
        - Platform array with one platform name  
        - Video binary data field name `datavideo`  
        - User from variables  
      - Auth: HTTP Header with API key from credentials  
    - Input: Binary video data and description text  
    - Output: JSON response with upload success and URL  
    - Edge Cases: API auth failure, network errors, file size limits  

  - **Edit Airtable Fields / 2 / 3**  
    - Type: Set node  
    - Role: Prepares fields to update Airtable upload status and URLs for each platform  
    - Config:  
      - Includes record ID, status string with success boolean, and URL or "error" fallback  
      - Reads from upload API response JSON  
    - Input: Upload results  
    - Output: JSON for Airtable status update  

#### 1.6 Status Update in Airtable

- **Overview:**  
Updates the Airtable record with the upload status and URLs for each platform after upload completion.

- **Nodes Involved:**  
  - Update TikTok Status  
  - Update Instagram Status  
  - Update YouTube Status - Success

- **Node Details:**

  - **Update TikTok / Instagram / YouTube Status**  
    - Type: Airtable node  
    - Role: Update Airtable record fields corresponding to platform upload status and URLs  
    - Config:  
      - Uses record ID from previous steps  
      - Fields updated: e.g., "TikTok Status", "Tiktok URL" etc.  
      - Credentials: Airtable API token  
    - Input: JSON prepared by corresponding Edit Airtable Fields nodes  
    - Output: Updated Airtable record metadata  
    - Potential Failures: API limits, invalid record ID, network errors  

#### 1.7 Error Handling and Notification

- **Overview:**  
Captures workflow errors and sends a notification message via Telegram unless the error is a specific DNS server offline message.

- **Nodes Involved:**  
  - Error Trigger  
  - If (condition)  
  - Telegram

- **Node Details:**

  - **Error Trigger**  
    - Type: Error Trigger node  
    - Role: Listens for any execution errors in the workflow  
    - Output: Error object with message  
    - Input: N/A (triggered on errors)  

  - **If**  
    - Type: Conditional node  
    - Role: Filters errors to exclude those containing "The DNS server returned an error, perhaps the server is offline"  
    - Input: Error message from Error Trigger  
    - Output: Passes only relevant errors onward  

  - **Telegram**  
    - Type: Telegram node  
    - Role: Sends a notification text "🔔 ERROR SUBIENDO VIDEOS" to a configured Telegram bot/channel  
    - Config: Uses Telegram API credentials  
    - Edge Cases: Telegram API rate limits or failures  

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                                | Input Node(s)                   | Output Node(s)                            | Sticky Note                                                                                                                        |
|-------------------------------|---------------------------------|------------------------------------------------|---------------------------------|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger           | Google Drive Trigger             | Detects new video files in Google Drive folder | None                            | Google Drive                             |                                                                                                                                   |
| Google Drive                  | Google Drive                    | Downloads the new video file                     | Google Drive Trigger            | Read video from Google Drive             |                                                                                                                                   |
| Read video from Google Drive  | Write Binary File               | Saves video locally                              | Google Drive                   | Get Audio from Video                      |                                                                                                                                   |
| Get Audio from Video          | OpenAI (Langchain)              | Transcribes audio from video                      | Read video from Google Drive   | Generate Description for Videos           | Extract the audio from video for generate the description                                                                          |
| Generate Description for Videos | OpenAI (Langchain)             | Generates AI social media description            | Get Audio from Video           | Set Variables                            | Request to OpenAi for generate description with the audio extracted from the video                                                |
| Set Variables                | Set                            | Stores configuration variables                   | Generate Description for Videos | Create Airtable Record                   |                                                                                                                                   |
| Create Airtable Record        | Airtable                       | Creates initial Airtable record                   | Set Variables                  | Edit Airtable Fields1                    |                                                                                                                                   |
| Edit Airtable Fields1         | Set                            | Prepare Airtable fields for description update  | Create Airtable Record         | Update Airtable with Description         |                                                                                                                                   |
| Update Airtable with Description | Airtable                     | Updates Airtable with description and metadata  | Edit Airtable Fields1          | Read Video for TikTok, Instagram, YouTube |                                                                                                                                   |
| Read Video for TikTok          | Read Binary File               | Reads local video for TikTok upload              | Update Airtable with Description | Upload Video to TikTok                   |                                                                                                                                   |
| Upload Video to TikTok         | HTTP Request                  | Uploads video to TikTok via API                   | Read Video for TikTok          | Edit Airtable Fields                     | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here)               |
| Edit Airtable Fields           | Set                            | Prepares upload status fields for TikTok         | Upload Video to TikTok         | Update TikTok Status                     |                                                                                                                                   |
| Update TikTok Status           | Airtable                       | Updates Airtable with TikTok upload status       | Edit Airtable Fields           |                                           |                                                                                                                                   |
| Read Video for Instagram       | Read Binary File               | Reads local video for Instagram upload           | Update Airtable with Description | Upload Video to Instagram                |                                                                                                                                   |
| Upload Video to Instagram      | HTTP Request                  | Uploads video to Instagram via API                | Read Video for Instagram       | Edit Airtable Fields 2                   | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here)               |
| Edit Airtable Fields 2         | Set                            | Prepares upload status fields for Instagram       | Upload Video to Instagram      | Update Instagram Status                  |                                                                                                                                   |
| Update Instagram Status        | Airtable                       | Updates Airtable with Instagram upload status    | Edit Airtable Fields 2         |                                           |                                                                                                                                   |
| Read Video for YouTube         | Read Binary File               | Reads local video for YouTube upload              | Update Airtable with Description | Upload Video to YouTube                  |                                                                                                                                   |
| Upload Video to YouTube        | HTTP Request                  | Uploads video to YouTube via API                   | Read Video for YouTube         | Edit Airtable Fields 3                   | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here)               |
| Edit Airtable Fields 3         | Set                            | Prepares upload status fields for YouTube          | Upload Video to YouTube        | Update YouTube Status - Success          |                                                                                                                                   |
| Update YouTube Status - Success | Airtable                     | Updates Airtable with YouTube upload status      | Edit Airtable Fields 3         |                                           |                                                                                                                                   |
| Error Trigger                 | Error Trigger                  | Listens for any workflow errors                    | N/A                           | If                                      |                                                                                                                                   |
| If                           | If                            | Filters specific DNS error from error notifications | Error Trigger                 | Telegram                                |                                                                                                                                   |
| Telegram                     | Telegram                      | Sends error notification message                   | If                            |                                           |                                                                                                                                   |
| Sticky Note                  | Sticky Note                   | Workflow description and usage instructions        | N/A                           | N/A                                     | ## Description\nThis automation allows you to upload a video to a configured Google Drive folder, and it will automatically create descriptions and upload it to Instagram, TikTok, and YouTube with Airtable tracking.\n\n## How to Use\n1. Configure your Airtable base and table IDs in the Set Variables node\n2. Set up Airtable fields: Video Name, Google Drive Link, File ID, Instagram Status, TikTok Status, YouTube Status, Upload Date, Description\n3. Generate an API token at upload-post.com and add to Upload nodes\n4. Configure your Google Drive folder\n5. Customize the OpenAI prompt for your specific use case\n6. Optional: Configure Telegram for error notifications\n\n## Requirements\n- Airtable account with configured base\n- upload-post.com account\n- Google Drive account\n- OpenAI API key |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Event: `fileCreated`  
   - Folder to Watch: Set to your Google Drive folder ID (e.g., `"18m0i341QLQuyWuHv_FBdz8-r-QDtofYm"`)  
   - Poll frequency: Every minute  
   - Credentials: Connect with your Google Drive OAuth2 account  

2. **Add Google Drive node**  
   - Type: Google Drive  
   - Operation: Download file  
   - File ID: Expression `={{ $json.id || $json.data[0].id }}` to get file ID from trigger  
   - Credentials: Same Google Drive OAuth2 account  
   - Enable retry on failure with 5 seconds interval  

3. **Add Write Binary File node** ("Read video from Google Drive")  
   - Type: Write Binary File  
   - File Name: Expression `={{ $json.originalFilename.replaceAll(" ", "_") }}`  
   - Connections: Connect Google Drive node output to this node  

4. **Add OpenAI node** ("Get Audio from Video")  
   - Type: OpenAI (Langchain)  
   - Resource: Audio  
   - Operation: Transcribe  
   - Credentials: OpenAI API key  
   - Connect from Write Binary File node  
   - Enable retry on failure with wait  

5. **Add OpenAI node** ("Generate Description for Videos")  
   - Type: OpenAI (Langchain)  
   - Model: GPT-4o (or latest available)  
   - Messages:  
     - System role: "You are an expert assistant in creating engaging social media video titles."  
     - User role: Provide examples of good Instagram descriptions and include dynamic audio transcription text from previous node as `{{ $('Get Audio from Video').item.json.text }}`  
   - Credentials: OpenAI API key  
   - Connect from "Get Audio from Video" node  
   - Enable retry on failure with wait  

6. **Add Set node** ("Set Variables")  
   - Type: Set  
   - Assign the following variables (replace placeholders):  
     - airtable_app_id: `"add_airtable_app_id"`  
     - airtable_table_id: `"add_airtable_table_id"`  
     - upload_post_user: e.g. `"test2"`  
   - Connect from "Generate Description for Videos" node  

7. **Add Airtable node** ("Create Airtable Record")  
   - Type: Airtable  
   - Operation: Append (create record)  
   - Application ID: Use expression `={{ $('Set Variables').item.json.airtable_app_id }}`  
   - Table ID: Use expression `={{ $('Set Variables').item.json.airtable_table_id }}`  
   - Fields: ["Video Name"] with value from Google Drive file original filename  
   - Credentials: Airtable API token  
   - Connect from "Set Variables" node  

8. **Add Set node** ("Edit Airtable Fields1")  
   - Type: Set  
   - Assign fields for update:  
     - ID: Airtable record ID from previous node  
     - Description: Content from generated description  
     - Video Name: From Google Drive node original filename  
     - Google Drive Links: From Google Drive node `webViewLink`  
     - Upload Date: current date/time with expression `$now`  
   - Connect from "Create Airtable Record" node  

9. **Add Airtable node** ("Update Airtable with Description")  
   - Type: Airtable  
   - Operation: Update record by ID  
   - Use app and table IDs from variables  
   - Fields: Description, Video Name, Google Drive Links, Upload Date from Set node  
   - Credentials: Airtable API token  
   - Connect from "Edit Airtable Fields1" node  

10. **Add Read Binary File nodes** ("Read Video for TikTok", "Read Video for Instagram", "Read Video for YouTube")  
    - Type: Read Binary File  
    - File Path: Use expression from saved file name (replace spaces with underscores)  
    - Data Property Name: `datavideo`  
    - Connect all three from "Update Airtable with Description" node  

11. **Add HTTP Request nodes** for uploading videos to TikTok, Instagram, and YouTube respectively  
    - Method: POST  
    - URL: `https://api.upload-post.com/api/upload`  
    - Authentication: HTTP Header Auth with API key from upload-post.com  
    - Content-Type: multipart/form-data  
    - Body Parameters:  
      - `title`: description text from generated description (YouTube truncated to 70 chars)  
      - `platform[]`: platform name (tiktok/instagram/youtube)  
      - `video`: form binary data field from corresponding Read Binary File node's `datavideo`  
      - `user`: from variables node `upload_post_user`  
    - Connect each Read Binary File node to corresponding HTTP Request node  

12. **Add Set nodes** ("Edit Airtable Fields", "Edit Airtable Fields 2", "Edit Airtable Fields 3")  
    - Prepare JSON fields for updating upload status and URLs for TikTok, Instagram, YouTube respectively  
    - Values extracted from each upload HTTP response JSON  
    - Connect each HTTP Request node to corresponding Set node  

13. **Add Airtable nodes** ("Update TikTok Status", "Update Instagram Status", "Update YouTube Status - Success")  
    - Operation: Update record by ID  
    - Fields: Status and URL fields corresponding to each platform  
    - Credentials: Airtable API token  
    - Connect each Set node to corresponding Airtable node  

14. **Add Error Trigger node**  
    - Listens for any workflow error  

15. **Add If node**  
    - Condition: Check error message does NOT contain `"The DNS server returned an error, perhaps the server is offline"` string to filter out common network noise errors  

16. **Add Telegram node**  
    - Sends message `"🔔 ERROR SUBIENDO VIDEOS"` to configured Telegram bot/channel  
    - Credentials: Telegram API token  
    - Connect If node's true branch to Telegram node  

17. **Add Sticky Note** (optional)  
    - Add workflow description, usage instructions, and requirements as documentation inside the editor  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| This workflow requires you to create an API token at [upload-post.com](https://upload-post.com) and configure it in the HTTP Request nodes for video upload authorization.                                                                                                                                                                                                                                                                                                                                                                                     | Upload-post.com API token setup                                            |
| Airtable base should have the following fields configured: Video Name, Google Drive Link, File ID, Instagram Status, TikTok Status, YouTube Status, Upload Date, Description.                                                                                                                                                                                                                                                                                                                                                                                      | Airtable base/table schema                                                  |
| OpenAI API key is required and should have access to GPT-4o or compatible model for best description generation quality.                                                                                                                                                                                                                                                                                                                                                                                                                                       | OpenAI API key and model requirements                                      |
| Telegram integration is optional but recommended for error notifications to monitor the workflow health.                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Telegram bot setup                                                         |
| The Google Drive folder ID to monitor and Airtable app/table IDs must be replaced with your actual IDs before running the workflow.                                                                                                                                                                                                                                                                                                                                                                                                                            | Configuration prerequisites                                                |
| Retry mechanisms are enabled on nodes interacting with external APIs (Google Drive, OpenAI) to improve reliability in case of transient failures.                                                                                                                                                                                                                                                                                                                                                                                                             | Reliability considerations                                                |
| The workflow handles potential DNS server errors by filtering them out from notifications to reduce noise.                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Error handling logic                                                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.