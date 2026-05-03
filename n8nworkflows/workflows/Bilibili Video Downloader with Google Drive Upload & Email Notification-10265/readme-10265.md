Bilibili Video Downloader with Google Drive Upload & Email Notification

https://n8nworkflows.xyz/workflows/bilibili-video-downloader-with-google-drive-upload---email-notification-10265


# Bilibili Video Downloader with Google Drive Upload & Email Notification

### 1. Workflow Overview

This workflow automates the process of downloading a video from Bilibili via a submitted URL, uploading it to Google Drive, and notifying the user by email with either a link to the uploaded video or a failure notification. It is designed for users who want to easily save Bilibili videos to their Google Drive and receive prompt email updates on the process outcome.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Captures the video URL input from a user-submitted web form.
- **1.2 Video Information Retrieval:** Calls an external API to fetch video metadata and media links based on the submitted URL.
- **1.3 Response Validation:** Checks the API response status to determine if the video info retrieval was successful.
- **1.4 Video Download:** Downloads the video file from the URL provided by the API.
- **1.5 Upload to Google Drive:** Uploads the downloaded video file to the user’s Google Drive account.
- **1.6 File Sharing Setup:** Sets sharing permissions on the uploaded Google Drive file to make it accessible externally.
- **1.7 Success Notification:** Sends an email to the user with a link to the uploaded video on Google Drive.
- **1.8 Failure Handling:** Introduces a delay before sending a failure notification email if any step fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Waits for a user to submit a form containing a Bilibili video URL, triggering the workflow start.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point waiting for user input via form submission  
  - Configuration: Form titled "Bilibili Video Downloader" with a required URL field labeled "URL" and placeholder text for a Bilibili video URL  
  - Inputs: None (trigger node)  
  - Outputs: To "Fetch Bilibili Video Info from API"  
  - Edge Cases: Form not submitted or invalid URL format (not validated at this node)  
  - Notes: Triggers the workflow when a user submits a form containing a Bilibili video URL

#### 1.2 Video Information Retrieval

**Overview:**  
Sends the submitted URL to a third-party Bilibili downloader API to retrieve video metadata and media resource URLs.

**Nodes Involved:**  
- Fetch Bilibili Video Info from API

**Node Details:**  
- **Fetch Bilibili Video Info from API**  
  - Type: HTTP Request  
  - Role: API call to get video details and media URLs  
  - Configuration:  
    - POST request to `https://bilibili-video-downloader.p.rapidapi.com/bili.php`  
    - Sends the URL as multipart form-data body parameter  
    - Includes required headers: `x-rapidapi-host` and `x-rapidapi-key` (placeholder for API key)  
    - Full HTTP response is captured (for status code checking)  
  - Inputs: From "On form submission"  
  - Outputs: To "Check API Response Status"  
  - Edge Cases: API key invalid or missing, API endpoint down, malformed URL causing API failure  
  - Notes: Sends the submitted URL to the Bilibili downloader API to retrieve video details and media links

#### 1.3 Response Validation

**Overview:**  
Checks that the API response status code is 200 to confirm successful retrieval of video info before proceeding.

**Nodes Involved:**  
- Check API Response Status

**Node Details:**  
- **Check API Response Status**  
  - Type: If (Conditional)  
  - Role: Branching logic based on API success  
  - Configuration: Checks if `$json.statusCode` equals 200  
  - Inputs: From "Fetch Bilibili Video Info from API"  
  - Outputs:  
    - On success (true branch): To "Download Video File"  
    - On failure (false branch): To "Processing Delay" (which leads to failure notification)  
  - Edge Cases: Status codes other than 200, unexpected response formats  
  - Notes: Verifies that the API response returned a 200 status code indicating success

#### 1.4 Video Download

**Overview:**  
Downloads the video file from the media URL extracted from the API response.

**Nodes Involved:**  
- Download Video File

**Node Details:**  
- **Download Video File**  
  - Type: HTTP Request  
  - Role: Downloads the actual video file content  
  - Configuration:  
    - GET request to URL extracted from `medias[0].resource_url` in API response body  
    - No special headers or authentication required  
  - Inputs: From "Check API Response Status" (success branch)  
  - Outputs: To "Upload Video to Google Drive"  
  - Edge Cases: Video URL inaccessible, network timeouts, large file size issues  
  - Notes: Downloads the Bilibili video file using the resource URL from the API response

#### 1.5 Upload to Google Drive

**Overview:**  
Uploads the downloaded video file to a folder in the connected Google Drive account.

**Nodes Involved:**  
- Upload Video to Google Drive

**Node Details:**  
- **Upload Video to Google Drive**  
  - Type: Google Drive  
  - Role: Uploads file to Google Drive  
  - Configuration:  
    - Uploads to "My Drive" root folder (`folderId` set to "root")  
    - Uses OAuth2 authentication with a pre-configured Google Drive account credential  
  - Inputs: From "Download Video File"  
  - Outputs: To "Google Drive Set Permission"  
  - Edge Cases: Drive quota exceeded, invalid credentials, upload failure  
  - Notes: Uploads the downloaded video file to the connected Google Drive account

#### 1.6 File Sharing Setup

**Overview:**  
Sets the sharing permissions of the uploaded video file to make it accessible to the user via a shareable link.

**Nodes Involved:**  
- Google Drive Set Permission

**Node Details:**  
- **Google Drive Set Permission**  
  - Type: Google Drive  
  - Role: Modifies file sharing settings to enable access  
  - Configuration:  
    - Uses the uploaded file ID to set sharing permissions (likely public or shareable link)  
    - OAuth2 authentication with the same Google Drive credential as upload node  
  - Inputs: From "Upload Video to Google Drive"  
  - Outputs: To "Success Notification Email with Drive Link"  
  - Edge Cases: Permission API failure, incorrect file ID, insufficient access rights  
  - Notes: Sets sharing permissions for the uploaded video file to make it accessible to the user

#### 1.7 Success Notification

**Overview:**  
Sends an email to the user containing the Google Drive link to the successfully uploaded video.

**Nodes Involved:**  
- Success Notification Email with Drive Link

**Node Details:**  
- **Success Notification Email with Drive Link**  
  - Type: Email Send  
  - Role: Notifies user of success with a download link  
  - Configuration:  
    - HTML email with message and embedded link to file (`{{ $('Upload Video to Google Drive').item.json.webContentLink }}`)  
    - Sends from `admin@test.com` to `user@tes.cm` (likely placeholders)  
    - Uses SMTP credentials configured in n8n  
  - Inputs: From "Google Drive Set Permission"  
  - Outputs: None (end node)  
  - Edge Cases: Email sending failure, invalid recipient email, SMTP authentication failure  
  - Notes: Sends an email to the user with the Google Drive link to the successfully uploaded video

#### 1.8 Failure Handling

**Overview:**  
Handles failure scenarios by introducing a delay and then sending a failure notification email.

**Nodes Involved:**  
- Processing Delay  
- Failure Notification Email

**Node Details:**  
- **Processing Delay**  
  - Type: Wait  
  - Role: Adds a short pause before failure notification to avoid rapid retries or overload  
  - Configuration: Default wait (no parameters explicitly set)  
  - Inputs: From "Check API Response Status" (failure branch)  
  - Outputs: To "Failure Notification Email"  
  - Edge Cases: Delay node failure is rare but possible if resource constrained  

- **Failure Notification Email**  
  - Type: Email Send  
  - Role: Notifies user that the video download/upload failed  
  - Configuration:  
    - Static HTML email informing about failure and recommending URL check  
    - Sends from `admin@test.com` to `user@test.com` (placeholders)  
    - Uses SMTP credentials configured in n8n  
  - Inputs: From "Processing Delay"  
  - Outputs: None (end node)  
  - Edge Cases: Email send failure, invalid recipient, SMTP issues  
  - Notes: Sends an email notifying the user that the video download process failed

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                            | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                                                      |
|----------------------------------|---------------------|------------------------------------------|-------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| On form submission               | Form Trigger        | Trigger workflow on user form submission | None                          | Fetch Bilibili Video Info from API | Triggers the workflow when a user submits a form containing a Bilibili video URL.                                               |
| Fetch Bilibili Video Info from API | HTTP Request       | Retrieve video info and media URLs       | On form submission             | Check API Response Status          | Sends the submitted URL to the Bilibili downloader API to retrieve video details and media links.                              |
| Check API Response Status        | If                  | Validate API success status code          | Fetch Bilibili Video Info from API | Download Video File (on success); Processing Delay (on failure) | Verifies that the API response returned a 200 status code indicating success.                                                  |
| Download Video File              | HTTP Request        | Download video file from API media URL    | Check API Response Status (true branch) | Upload Video to Google Drive        | Downloads the Bilibili video file using the resource URL from the API response.                                                 |
| Upload Video to Google Drive     | Google Drive        | Upload downloaded video to Google Drive   | Download Video File            | Google Drive Set Permission        | Uploads the downloaded video file to the connected Google Drive account.                                                       |
| Google Drive Set Permission      | Google Drive        | Set sharing permissions on uploaded file  | Upload Video to Google Drive   | Success Notification Email with Drive Link | Sets sharing permissions for the uploaded video file to make it accessible to the user.                                         |
| Success Notification Email with Drive Link | Email Send         | Email user success notification with link | Google Drive Set Permission    | None                             | Sends an email to the user with the Google Drive link to the successfully uploaded video.                                       |
| Processing Delay                | Wait                | Adds delay before failure notification    | Check API Response Status (false branch) | Failure Notification Email         | Adds a short delay in workflow execution before handling a failed download scenario.                                            |
| Failure Notification Email       | Email Send          | Email user failure notification           | Processing Delay              | None                             | Sends an email notifying the user that the video download process failed.                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "On form submission"**  
   - Type: Form Trigger  
   - Configure form title: "Bilibili Video Downloader"  
   - Add one required field: Label "URL", placeholder " https://www.bilibili.com/video/....."  
   - Save and position the node as entry point.

2. **Add HTTP Request Node: "Fetch Bilibili Video Info from API"**  
   - Set method to POST  
   - URL: `https://bilibili-video-downloader.p.rapidapi.com/bili.php`  
   - Content-Type: multipart-form-data  
   - Body parameter: name="url", value from expression `{{$json.URL}}`  
   - Header parameters:  
     - `x-rapidapi-host`: `bilibili-video-downloader.p.rapidapi.com`  
     - `x-rapidapi-key`: your API key (replace "your key")  
   - Enable full response capture for status code  
   - Connect output from "On form submission" to this node’s input.

3. **Add If Node: "Check API Response Status"**  
   - Condition: Check if `$json.statusCode` equals `200` (number)  
   - Connect output of "Fetch Bilibili Video Info from API" to this node.

4. **Add HTTP Request Node: "Download Video File"**  
   - Method: GET  
   - URL: Expression `{{$json.body.medias[0].resource_url}}`  
   - Connect the true (success) output of "Check API Response Status" to this node.

5. **Add Google Drive Node: "Upload Video to Google Drive"**  
   - Operation: Upload file  
   - Drive: "My Drive"  
   - Folder: root ("/")  
   - Authentication: OAuth2 with Google Drive credentials (set up or select existing)  
   - Connect output of "Download Video File" to this node.

6. **Add Google Drive Node: "Google Drive Set Permission"**  
   - Operation: Share file  
   - File ID: Expression `{{$json.id}}` (uses uploaded file’s ID)  
   - Permissions: Use default share settings to create a shareable link (ensure public or anyone with link access)  
   - Authentication: Same Google Drive OAuth2 credentials  
   - Connect output of "Upload Video to Google Drive" to this node.

7. **Add Email Send Node: "Success Notification Email with Drive Link"**  
   - From Email: `admin@test.com` (replace with your sender email)  
   - To Email: `user@tes.cm` (replace with user email or dynamic input if applicable)  
   - Subject: "Your Bilibili Video is Ready and Shared via Google Drive"  
   - HTML Body:  
     ```
     Hello,

     Your requested Bilibili video has been successfully downloaded and uploaded to Google Drive. It is now ready for you to access.

     You can download or view the video directly from your Google Drive using the shared link below:
     {{ $('Upload Video to Google Drive').item.json.webContentLink }}
     ```  
   - Credentials: SMTP configured with your email provider (e.g., SMTP account 2)  
   - Connect output of "Google Drive Set Permission" to this node.

8. **Add Wait Node: "Processing Delay"**  
   - No special parameters (default wait time)  
   - Connect the false (failure) output of "Check API Response Status" to this node.

9. **Add Email Send Node: "Failure Notification Email"**  
   - From Email: `admin@test.com`  
   - To Email: `user@test.com`  
   - Subject: "Bilibili Video Download Failed"  
   - HTML Body:  
     ```
     Hello,

     We regret to inform you that your Bilibili video download request could not be completed successfully.

     There might have been an issue with the URL provided or a problem with the video processing service.

     Please double-check the IMDB URL and try again. If the problem persists, feel free to contact our support team for assistance.

     We apologize for the inconvenience caused.
     ```  
   - Credentials: Same SMTP as success email  
   - Connect output of "Processing Delay" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow allows users to submit a Bilibili video URL, automatically downloads the video via an API, uploads to Google Drive, and emails the user with the shared link or error notification. | Overview of entire workflow purpose                                                                        |
| SMTP email credentials and Google Drive OAuth2 credentials need to be configured in n8n for email and Drive integration respectively. | Credential setup requirement                                                                                 |
| The Bilibili downloader API requires an API key from RapidAPI; replace placeholder `"your key"` with a valid key.                  | API key management for the external Bilibili downloader API                                                |
| The workflow includes graceful failure handling with delay and user notification to improve UX.                                     | Failure scenario management                                                                                 |
| Google Drive sharing permissions are set to allow user access via shared link after upload.                                          | Permissions management for uploaded files                                                                   |

---

**Disclaimer:** The provided text is exclusively generated from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.