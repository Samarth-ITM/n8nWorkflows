Automated Instagram Reels Posting Using Google Drive, Cloudinary & Sheets

https://n8nworkflows.xyz/workflows/automated-instagram-reels-posting-using-google-drive--cloudinary---sheets-6217


# Automated Instagram Reels Posting Using Google Drive, Cloudinary & Sheets

### 1. Workflow Overview

This workflow automates the process of scheduling and posting Instagram Reels using Google Sheets, Google Drive, and Cloudinary. It is designed for users managing Instagram content calendars with videos stored in Google Drive, leveraging Cloudinary as a media host, and Google Sheets as a content scheduler. The workflow periodically checks for scheduled Reels, downloads the videos from Google Drive, uploads them to Cloudinary, then posts the videos to Instagram Reels, updating the content status accordingly.

**Logical blocks:**

- **1.1 Schedule Trigger & Content Retrieval:** Periodic triggering and fetching scheduled Instagram Reel posts from Google Sheets.
- **1.2 Video Retrieval from Google Drive:** Listing and downloading the scheduled video files from a shared Google Drive folder.
- **1.3 Cloudinary Upload:** Uploading the downloaded videos to Cloudinary to obtain a publicly accessible video URL.
- **1.4 Instagram Post Preparation and Publishing:** Setting Instagram credentials and metadata, creating a media container, waiting for processing, and publishing the Reel.
- **1.5 Status Update:** Updating the Google Sheet to mark the post as processed.
- **1.6 Notes & Instructions:** Sticky notes providing setup hints and important configuration reminders.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule Trigger & Content Retrieval

- **Overview:** This block periodically triggers the workflow and queries the Google Sheet to retrieve Reel posts marked as "Scheduled to post," filtering for Reels only.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Execution for Instagram contents

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger node  
    - Role: Starts workflow execution at regular intervals (every minute by default here)  
    - Configuration: Interval set to minutes (default 1 minute)  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Get Execution for Instagram contents" node  
    - Edge cases: Misconfiguration could cause too frequent runs; consider rate limits.

  - **Get Execution for Instagram contents**  
    - Type: Google Sheets node (Read)  
    - Role: Reads rows from Google Sheet where "Type" = "Reel" and "Status" = "Scheduled to post"  
    - Configuration: Filters applied on columns "Type" and "Status"; sheet and document IDs set to target specific Sheet  
    - Credentials: Google Sheets OAuth2  
    - Inputs: From Schedule Trigger  
    - Outputs: To "Get video list from a Google Drive folder"  
    - Edge cases: API quota limits, sheet schema changes, missing or malformed data.

---

#### Block 1.2: Video Retrieval from Google Drive

- **Overview:** Retrieves video file metadata from a specific Google Drive folder and downloads each video for upload.
- **Nodes Involved:**  
  - Get video list from a Google Drive folder  
  - Download Video (Reel) from Google Drive (Carousel)

- **Node Details:**

  - **Get video list from a Google Drive folder**  
    - Type: Google Drive node (List Files)  
    - Role: Lists all files inside a folder specified dynamically by the "Folder" field from the previous step  
    - Configuration: Folder ID passed dynamically from spreadsheet data; returns metadata including file ID for download  
    - Credentials: Google Drive OAuth2  
    - Inputs: From "Get Execution for Instagram contents"  
    - Outputs: To "Download Video (Reel) from Google Drive (Carousel)"  
    - Edge cases: Folder not shared publicly or permission errors, empty folders, large folder with pagination.

  - **Download Video (Reel) from Google Drive (Carousel)**  
    - Type: HTTP Request node  
    - Role: Downloads video file using Google Drive's direct download URL  
    - Configuration: URL constructed with `https://drive.google.com/uc?export=download&id={{ $json.id }}`; response type set to "file"  
    - Inputs: From Google Drive file list  
    - Outputs: To "Upload videos to Cloudinary"  
    - Edge cases: Download failures due to permissions, file not found, large file size causing timeout.

---

#### Block 1.3: Cloudinary Upload

- **Overview:** Uploads the downloaded video file to Cloudinary for hosting and public access.
- **Nodes Involved:**  
  - Upload videos to Cloudinary

- **Node Details:**

  - **Upload videos to Cloudinary**  
    - Type: HTTP Request node  
    - Role: Uploads video binary data to Cloudinary using a POST request with multipart/form-data  
    - Configuration: URL set to Cloudinary upload endpoint with placeholders `<your-cloud-name>` and `<your_upload_preset>` to be replaced by user; sends binary data under "file" parameter  
    - Inputs: Receives binary video file from previous node  
    - Outputs: To "Setup for Instagram (access token, ig_business_id)"  
    - Edge cases: Authentication failure if credentials invalid, upload preset misconfiguration, network issues, file size limits.

---

#### Block 1.4: Instagram Post Preparation and Publishing

- **Overview:** Prepares Instagram API parameters, creates a media container for the Reel, waits for processing, then publishes the Reel.
- **Nodes Involved:**  
  - Setup for Instagram (access token, ig_business_id)  
  - Create Media Container (Reels)  
  - Wait  
  - Publish Instagram Reels

- **Node Details:**

  - **Setup for Instagram (access token, ig_business_id)**  
    - Type: Set node  
    - Role: Defines Instagram API parameters including access token, Instagram user ID, video URL from Cloudinary, and caption from Google Drive  
    - Configuration: Values for `access_token` and `ig_user_id` must be manually replaced with valid Instagram credentials; `image_url` and `caption` assigned dynamically  
    - Inputs: From Cloudinary upload response  
    - Outputs: To "Create Media Container (Reels)"  
    - Edge cases: Incorrect tokens leading to API rejection, missing video URL or caption.

  - **Create Media Container (Reels)**  
    - Type: HTTP Request node  
    - Role: Calls Instagram Graph API to create a media container for the video Reel  
    - Configuration: POST request to Instagram API endpoint `/media` with parameters: video_url, caption, access_token, media_type=REELS  
    - Inputs: From Instagram setup node  
    - Outputs: To "Wait" node  
    - Edge cases: API rate limiting, invalid access tokens, malformed requests.

  - **Wait**  
    - Type: Wait node  
    - Role: Delays the workflow to allow Instagram to process the media container (usually a few seconds)  
    - Configuration: Default delay (time not explicitly defined here; could be configured as needed)  
    - Inputs: From media container creation  
    - Outputs: To "Publish Instagram Reels"  
    - Edge cases: If wait time too short, posting might fail due to incomplete media processing.

  - **Publish Instagram Reels**  
    - Type: HTTP Request node  
    - Role: Publishes the created media container as an Instagram Reel  
    - Configuration: POST request to Instagram API endpoint `/media_publish` with parameters: creation_id (from previous node response) and access_token  
    - Inputs: From Wait node  
    - Outputs: To "Update Execute to \"Processed\""  
    - Edge cases: API errors if creation_id invalid or expired, token expiration.

---

#### Block 1.5: Status Update

- **Overview:** Updates the Google Sheet to mark the processed Reel as "Processed" to avoid reposting.
- **Nodes Involved:**  
  - Update Execute to "Processed"

- **Node Details:**

  - **Update Execute to "Processed"**  
    - Type: Google Sheets node (Update)  
    - Role: Updates the row in the spreadsheet matching the post execution ID to set the "Status" field to "Processed"  
    - Configuration: Uses "ExecuteId" as matching column; updates "Status" to "Processed"  
    - Credentials: Google Sheets OAuth2  
    - Inputs: From "Publish Instagram Reels"  
    - Outputs: None (end of workflow)  
    - Edge cases: Mismatched or missing ExecuteId causing update failure, API limits.

---

#### Block 1.6: Notes & Instructions

- **Overview:** Contains sticky notes with important setup instructions and reminders for users.
- **Nodes Involved:**  
  - Sticky Note1  
  - Sticky Note  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**

  - **Sticky Note1**  
    - Content: Reminder that the Google Drive folder must be publicly shared with "Anyone with the link can view"  
    - Edge cases: If folder is private, video downloads will fail.

  - **Sticky Note**  
    - Content: Instructions to update Cloudinary `<your-cloud-name>` and `<your_upload_preset>` to match user account settings  
    - Edge cases: Misconfiguration leads to upload failures.

  - **Sticky Note2**  
    - Content: Full workflow overview with detailed explanation and example setup links  
    - Edge cases: Provides critical context but no direct workflow logic.

  - **Sticky Note3**  
    - Content: Reminder to update Instagram access token and user ID placeholders to valid credentials  
    - Edge cases: Failure to update causes Instagram API errors.

---

### 3. Summary Table

| Node Name                               | Node Type            | Functional Role                       | Input Node(s)                         | Output Node(s)                             | Sticky Note                                                                                                           |
|----------------------------------------|----------------------|------------------------------------|-------------------------------------|--------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                       | Schedule Trigger     | Starts workflow execution periodically | None                                | Get Execution for Instagram contents       |                                                                                                                       |
| Get Execution for Instagram contents  | Google Sheets        | Retrieves scheduled Reel posts      | Schedule Trigger                    | Get video list from a Google Drive folder  |                                                                                                                       |
| Get video list from a Google Drive folder | Google Drive         | Lists video files in specified folder | Get Execution for Instagram contents | Download Video (Reel) from Google Drive (Carousel) | Sticky Note1: The Google Drive folder needs to be shared publicly (Anyone with the link can view)                      |
| Download Video (Reel) from Google Drive (Carousel) | HTTP Request         | Downloads video file from Google Drive | Get video list from a Google Drive folder | Upload videos to Cloudinary                 |                                                                                                                       |
| Upload videos to Cloudinary            | HTTP Request         | Uploads video to Cloudinary         | Download Video (Reel) from Google Drive (Carousel) | Setup for Instagram (access token, ig_business_id) | Sticky Note: After creating account and folder on Cloudinary, update <your-cloud-name> and <your_upload_preset>        |
| Setup for Instagram (access token, ig_business_id) | Set                  | Prepares Instagram API parameters   | Upload videos to Cloudinary          | Create Media Container (Reels)              | Sticky Note3: Update your Instagram's access token and user_id placeholders                                           |
| Create Media Container (Reels)         | HTTP Request         | Creates Instagram media container   | Setup for Instagram (access token, ig_business_id) | Wait                                        |                                                                                                                       |
| Wait                                  | Wait                 | Delays to allow media processing    | Create Media Container (Reels)       | Publish Instagram Reels                      |                                                                                                                       |
| Publish Instagram Reels                | HTTP Request         | Publishes the Instagram Reel        | Wait                               | Update Execute to "Processed"               |                                                                                                                       |
| Update Execute to "Processed"          | Google Sheets        | Updates post status in Google Sheet | Publish Instagram Reels             | None                                         |                                                                                                                       |
| Sticky Note1                          | Sticky Note          | Shared folder access reminder       | None                                | None                                         | The Google Drive folder needs to be shared publicly (Anyone with the link can view)                                    |
| Sticky Note                           | Sticky Note          | Cloudinary setup instructions       | None                                | None                                         | After creating account and folder on Cloudinary, update <your-cloud-name> and <your_upload_preset>                     |
| Sticky Note2                         | Sticky Note          | Workflow overview and setup guide   | None                                | None                                         | Full workflow overview and example setup details                                                                      |
| Sticky Note3                         | Sticky Note          | Instagram credentials reminder      | None                                | None                                         | Update your Instagram's access token and user_id placeholders                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval to run every 1 minute (or desired frequency).

2. **Add Google Sheets Node - "Get Execution for Instagram contents"**  
   - Operation: Read rows  
   - Set filters to select rows where "Type" = "Reel" and "Status" = "Scheduled to post"  
   - Set document ID and sheet name to your target Google Sheet  
   - Use Google Sheets OAuth2 credentials connected to your Google account  
   - Connect Schedule Trigger output to this node input.

3. **Add Google Drive Node - "Get video list from a Google Drive folder"**  
   - Operation: List files from folder  
   - Set folder ID dynamically from the "Folder" column in the Google Sheets data  
   - Request fields: file id, name, thumbnailLink, webViewLink  
   - Use Google Drive OAuth2 credentials  
   - Connect output of Google Sheets node to this node.

4. **Add HTTP Request Node - "Download Video (Reel) from Google Drive (Carousel)"**  
   - Method: GET  
   - URL: `https://drive.google.com/uc?export=download&id={{ $json.id }}` (use expression to get file ID)  
   - Response format: File (binary)  
   - Connect Google Drive list node output to this node.

5. **Add HTTP Request Node - "Upload videos to Cloudinary"**  
   - Method: POST  
   - URL: `https://api.cloudinary.com/v1_1/<your-cloud-name>/video/upload` (replace `<your-cloud-name>`)  
   - Content-Type: multipart/form-data  
   - Body parameters:  
     - `file`: binary data input from previous node  
     - `upload_preset`: `<your_upload_preset>` (replace accordingly)  
   - Response format: JSON  
   - Connect download node output to this node.

6. **Add Set Node - "Setup for Instagram (access token, ig_business_id)"**  
   - Assign variables:  
     - `access_token`: your Instagram access token  
     - `ig_user_id`: your Instagram business user ID  
     - `image_url`: set to Cloudinary response URL (e.g., `={{ $json.url }}`)  
     - `caption`: set dynamically from Google Drive or Google Sheets content  
   - Connect Cloudinary upload node output to this node.

7. **Add HTTP Request Node - "Create Media Container (Reels)"**  
   - Method: POST  
   - URL: `https://graph.instagram.com/v23.0/{{ $json.ig_user_id }}/media`  
   - Content-Type: application/x-www-form-urlencoded  
   - Body parameters:  
     - `video_url`: `={{ $json.video_url }}`  
     - `caption`: `={{ $json.caption }}`  
     - `access_token`: `={{ $json.access_token }}`  
     - `media_type`: `REELS`  
   - Connect Set node output to this node.

8. **Add Wait Node**  
   - Use default delay or specify a delay (e.g., 5 seconds) to allow Instagram media processing  
   - Connect media container creation node output to this node.

9. **Add HTTP Request Node - "Publish Instagram Reels"**  
   - Method: POST  
   - URL: `https://graph.instagram.com/v23.0/{{ $json.ig_user_id }}/media_publish`  
   - Body parameters:  
     - `creation_id`: `={{ $json.body.id }}` (from media container creation response)  
     - `access_token`: `={{ $json.access_token }}`  
   - Connect Wait node output to this node.

10. **Add Google Sheets Node - "Update Execute to \"Processed\""**  
    - Operation: Update row  
    - Use "ExecuteId" column to match the correct row  
    - Update "Status" column to "Processed"  
    - Use the proper Google Sheets document and sheet IDs  
    - Connect Publish Instagram Reels node output to this node.

11. **Add Sticky Notes as needed** for reminders:  
    - Public sharing of Google Drive folder  
    - Cloudinary account setup (`<your-cloud-name>`, `<your_upload_preset>`)  
    - Instagram access token and user ID configuration  
    - Workflow overview and usage instructions (optional but recommended)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Google Drive folder must be shared publicly with "Anyone with the link can view" to enable video downloads.                                                                    | Critical for Google Drive download permissions.                                                                                |
| Cloudinary requires correct `<your-cloud-name>` and `<your_upload_preset>` values configured in the upload HTTP Request node.                                                | [Cloudinary Upload Presets](https://cloudinary.com/documentation/upload_presets)                                               |
| Instagram access token and Instagram Business user ID must be updated manually in the Set node for API calls to succeed.                                                     | Obtain via Facebook Developer portal and Instagram Graph API documentation.                                                    |
| Workflow overview and example setup details are provided in Sticky Note2 inside the workflow for user guidance.                                                               | Includes sample Google Sheets link: https://docs.google.com/spreadsheets/d/1TjZL_eWbs01DdRYs8pJNDr5UMXzYe8u311o6rVUwjdg        |
| Instagram API calls require a valid Business Instagram account linked to a Facebook Page with appropriate permissions granted to the access token.                            | See [Instagram Graph API](https://developers.facebook.com/docs/instagram-api)                                                  |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow designed for legal, public content handling respecting all current content policies. No illegal or protected elements are included.