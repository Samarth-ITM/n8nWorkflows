Automated IMDB Video Downloader with Google Drive Upload & Email Alerts

https://n8nworkflows.xyz/workflows/automated-imdb-video-downloader-with-google-drive-upload---email-alerts-9066


# Automated IMDB Video Downloader with Google Drive Upload & Email Alerts

### 1. Workflow Overview

This workflow automates the process of downloading a video from an IMDB URL submitted via a user form, uploading the video to Google Drive, and sending email notifications about success or failure. It is designed for use cases such as content creators, marketers, educators, or social media managers who need seamless access to IMDB videos with cloud storage and instant notifications.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Captures the IMDB video URL from a user-submitted form.
- **1.2 IMDB Video Metadata Retrieval**: Sends the URL to an external API to fetch video download metadata.
- **1.3 API Response Validation and Routing**: Checks if the API call was successful and routes accordingly.
- **1.4 Video Downloading**: Downloads the video file from the URL provided by the API.
- **1.5 Google Drive Upload and Sharing**: Uploads the video to Google Drive and sets sharing permissions.
- **1.6 Email Notifications**: Notifies the user via email whether the process succeeded or failed.
- **1.7 Failure Handling Delay**: Adds a delay before sending failure notification to allow retries or asynchronous process completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user submits an IMDB video URL via a form. It captures the input URL for downstream processing.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **Node:** On form submission  
    - Type: Form Trigger  
    - Configuration:  
      - Form title: "IMDB video downloader"  
      - One required field labeled "URL" capturing the IMDB video URL  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Fetch IMDB Video Info from API"  
    - Edge cases: Form might be submitted with invalid or malformed URLs; validation is minimal on the form itself.  
    - Sticky Note: Explains it triggers the workflow with the submitted URL.

#### 1.2 IMDB Video Metadata Retrieval

- **Overview:**  
  Sends a POST request to an IMDB downloader API to retrieve video metadata and download URLs using the submitted URL. Includes RapidAPI authentication.

- **Nodes Involved:**  
  - Fetch IMDB Video Info from API

- **Node Details:**  
  - **Node:** Fetch IMDB Video Info from API  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `https://imdb-downloader.p.rapidapi.com/imdb.php`  
      - Headers: Includes RapidAPI host and key (requires user to input valid API key)  
      - Body: multipart-form-data, includes the submitted URL as parameter named "url"  
      - On error: continue workflow to allow error handling downstream  
    - Inputs: From "On form submission" node  
    - Outputs: Connects to "Check API Response Status"  
    - Edge cases: API key missing or invalid, API downtime, malformed API responses  
    - Sticky Note: Notes the use of POST request and API authentication

#### 1.3 API Response Validation and Routing

- **Overview:**  
  Evaluates the HTTP response status code from the API call. If successful (status 200), proceeds to download the video; otherwise, routes to failure handling.

- **Nodes Involved:**  
  - Check API Response Status

- **Node Details:**  
  - **Node:** Check API Response Status  
    - Type: If Node  
    - Configuration:  
      - Condition: Check if `statusCode` equals 200  
    - Inputs: From "Fetch IMDB Video Info from API"  
    - Outputs:  
      - True branch: "Download Video File"  
      - False branch: "Processing Delay" (leads to failure notification)  
    - Edge cases: Status codes other than 200, unexpected API responses

#### 1.4 Video Downloading

- **Overview:**  
  Downloads the video file (MP4) from the URL obtained in the API response.

- **Nodes Involved:**  
  - Download Video File

- **Node Details:**  
  - **Node:** Download Video File  
    - Type: HTTP Request  
    - Configuration:  
      - URL: Extracted from API response JSON field `body.medias[0].url`  
      - Method: GET (default)  
    - Inputs: From "Check API Response Status" (True)  
    - Outputs: Connects to "Upload Video to Google Drive"  
    - Edge cases: Broken download link, slow response, large file size causing timeout  
    - Sticky Note: Explains it fetches the media content for upload

#### 1.5 Google Drive Upload and Sharing

- **Overview:**  
  Uploads the downloaded video file to Google Drive in the root folder and sets sharing permissions to make the file accessible.

- **Nodes Involved:**  
  - Upload Video to Google Drive  
  - Google Drive Set Permission

- **Node Details:**  
  - **Node:** Upload Video to Google Drive  
    - Type: Google Drive  
    - Configuration:  
      - Drive: "My Drive" (default user drive)  
      - Folder: Root folder (`root`)  
      - Authentication: OAuth2 with Google Drive credentials  
      - File content: The downloaded video file from previous node  
    - Inputs: From "Download Video File"  
    - Outputs: Connects to "Google Drive Set Permission"  
    - Edge cases: Google Drive API limits, quota exceeded, auth token expiration  
    - Sticky Note: Explains upload to root folder using OAuth2 credentials

  - **Node:** Google Drive Set Permission  
    - Type: Google Drive  
    - Configuration:  
      - Operation: Share file  
      - File ID: Dynamic from uploaded file's ID  
      - Permissions: Default sharing permissions for access  
      - Authentication: OAuth2  
    - Inputs: From "Upload Video to Google Drive"  
    - Outputs: Connects to "Success Notification Email with Drive Link"  
    - Edge cases: Permission setting failure, invalid file ID  
    - Sticky Note: Notes enabling recipient access via shareable link

#### 1.6 Email Notifications

- **Overview:**  
  Sends either a success email with the Google Drive link or a failure notification email, depending on the workflow outcome.

- **Nodes Involved:**  
  - Success Notification Email with Drive Link  
  - Failure Notification Email

- **Node Details:**  
  - **Node:** Success Notification Email with Drive Link  
    - Type: Email Send  
    - Configuration:  
      - Subject: "Your IMDB Video is Ready and Shared via Google Drive"  
      - To: `user@tes.cm` (configured recipient)  
      - From: `admin@test.com`  
      - Body: Includes dynamic Google Drive webContentLink from uploaded file metadata  
      - SMTP credentials configured for sending  
    - Inputs: From "Google Drive Set Permission"  
    - Outputs: None (terminal node)  
    - Edge cases: SMTP failure, invalid recipient email  
    - Sticky Note: Notes sending email with Drive shareable link

  - **Node:** Failure Notification Email  
    - Type: Email Send  
    - Configuration:  
      - Subject: "IMDB Video Download Failed"  
      - To: `user@test.com`  
      - From: `admin@test.com`  
      - Body: Explains failure, encourages retry or support contact  
      - SMTP credentials configured for sending  
    - Inputs: From "Processing Delay"  
    - Outputs: None (terminal node)  
    - Edge cases: SMTP failure, invalid recipient email  
    - Sticky Note: Describes failure notification content

#### 1.7 Failure Handling Delay

- **Overview:**  
  Adds a wait/delay before sending the failure notification, allowing asynchronous retries or processes to complete.

- **Nodes Involved:**  
  - Processing Delay

- **Node Details:**  
  - **Node:** Processing Delay  
    - Type: Wait  
    - Configuration: No explicit parameters set (default delay duration)  
    - Inputs: From "Check API Response Status" (False branch)  
    - Outputs: Connects to "Failure Notification Email"  
    - Edge cases: Delay duration may be too short or too long depending on context  
    - Sticky Note: Notes purpose to add delay before failure notification

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                       | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                                                     |
|----------------------------------|---------------------|------------------------------------|-----------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| On form submission               | Form Trigger        | Capture user-submitted IMDB URL     | -                           | Fetch IMDB Video Info from API  | Triggers the workflow when a user submits the IMDB video URL through a form. It captures the URL as input for further processing. |
| Fetch IMDB Video Info from API  | HTTP Request        | Retrieve video metadata/download URL| On form submission          | Check API Response Status       | Sends the submitted URL to the IMDB downloader API via a POST request to retrieve video metadata and download links. Handles API authentication with RapidAPI keys. |
| Check API Response Status       | If                  | Validate API response status        | Fetch IMDB Video Info from API | Download Video File / Processing Delay | Checks if the API call was successful by verifying the response status code equals 200. Routes workflow based on success or failure of the API request. |
| Download Video File             | HTTP Request        | Download video file from API URL    | Check API Response Status   | Upload Video to Google Drive    | Downloads the actual video file (MP4) from the URL provided by the API response. This node fetches the media content for later upload. |
| Upload Video to Google Drive    | Google Drive        | Upload downloaded video to Drive    | Download Video File         | Google Drive Set Permission     | Uploads the downloaded video file to a specified Google Drive folder (root drive in this case). Uses OAuth2 credentials to authenticate with Google Drive. |
| Google Drive Set Permission     | Google Drive        | Set sharing permissions on file     | Upload Video to Google Drive | Success Notification Email with Drive Link | Sets sharing permissions on the uploaded file to make it accessible via a shareable link. Enables the recipient to view or download the file. |
| Success Notification Email with Drive Link | Email Send          | Notify user of success with Drive link | Google Drive Set Permission | -                              | Sends an email to the user with a notification that their video is ready and includes the Google Drive link to access the video. |
| Processing Delay               | Wait                 | Delay before sending failure email | Check API Response Status (False branch) | Failure Notification Email    | Adds a wait/delay period before sending a failure notification. This can allow time for any asynchronous processes or retries. |
| Failure Notification Email     | Email Send            | Notify user of failure              | Processing Delay            | -                              | Sends an email informing the user that their video download failed, possibly due to an invalid URL or API issue, and suggests retrying or contacting support. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Form Trigger** node named "On form submission".  
   - Configure form title as "IMDB video downloader".  
   - Add a required form field labeled "URL" to capture the IMDB video URL.  
   - This node triggers the workflow on form submission.

2. **Add HTTP Request to Fetch IMDB Video Info:**  
   - Add an **HTTP Request** node named "Fetch IMDB Video Info from API".  
   - Set method to POST.  
   - Set URL to `https://imdb-downloader.p.rapidapi.com/imdb.php`.  
   - Under Headers, add:  
     - `x-rapidapi-host` = `imdb-downloader.p.rapidapi.com`  
     - `x-rapidapi-key` = `"your key"` (replace with valid RapidAPI key).  
   - Set body content type to `multipart-form-data`.  
   - Add body parameter named "url" with value set to `{{$json.URL}}` to pass submitted URL dynamically.  
   - Set On Error option to "Continue".  
   - Connect "On form submission" output to this node.

3. **Add Conditional Node for API Response Status:**  
   - Add an **If** node named "Check API Response Status".  
   - Configure condition to check if `{{$json.statusCode}}` equals 200.  
   - Connect output of "Fetch IMDB Video Info from API" to this node.

4. **Add HTTP Request to Download Video File:**  
   - Add an **HTTP Request** node named "Download Video File".  
   - Set URL to `{{$json.body.medias[0].url}}` to dynamically get video download link.  
   - Connect "Check API Response Status" True branch to this node.

5. **Add Google Drive Upload Node:**  
   - Add a **Google Drive** node named "Upload Video to Google Drive".  
   - Set operation to upload file.  
   - Set Drive to "My Drive" and Folder to "root" (or another folder as desired).  
   - Use OAuth2 credentials for Google Drive authentication (configure credential ahead of time).  
   - Connect output of "Download Video File" to this node.

6. **Add Google Drive Set Permission Node:**  
   - Add a **Google Drive** node named "Google Drive Set Permission".  
   - Set resource to "file" and operation to "share".  
   - Set fileId to `{{$json.id}}` from the uploaded file.  
   - Use OAuth2 credentials (same as upload).  
   - Connect output of "Upload Video to Google Drive" to this node.

7. **Add Success Notification Email Node:**  
   - Add an **Email Send** node named "Success Notification Email with Drive Link".  
   - Configure SMTP credentials for sending emails.  
   - Set subject to "Your IMDB Video is Ready and Shared via Google Drive".  
   - Set To email to the user's email (hardcoded or dynamic as per your form).  
   - Set From email accordingly.  
   - Email body includes link: `{{$node["Upload Video to Google Drive"].json["webContentLink"]}}`.  
   - Connect output of "Google Drive Set Permission" to this node.

8. **Add Wait Node for Failure Delay:**  
   - Add a **Wait** node named "Processing Delay".  
   - Configure delay as needed (defaults can be used).  
   - Connect False branch of "Check API Response Status" to this node.

9. **Add Failure Notification Email Node:**  
   - Add an **Email Send** node named "Failure Notification Email".  
   - Configure SMTP credentials.  
   - Set subject to "IMDB Video Download Failed".  
   - Set To and From emails.  
   - Include failure explanation in email body.  
   - Connect output of "Processing Delay" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow automates IMDB video downloads, Google Drive uploads, and user email notifications to streamline content acquisition workflows. | Project description and use case summary included in Sticky Note9 node for holistic understanding.    |
| RapidAPI key is required for the IMDB downloader API; users must replace placeholder key with valid credentials for successful API calls. | https://rapidapi.com/                                                                                   |
| Google Drive OAuth2 credentials must be preconfigured with appropriate scopes for file upload and permission changes.                     | Google Drive API documentation: https://developers.google.com/drive/api/v3/about-auth                  |
| SMTP credentials must be configured with a valid mail server for sending notification emails.                                               | Ensure SMTP server allows sending from configured From email addresses.                                |
| The workflow includes error handling with graceful continuation and delayed failure notification to improve user experience and reliability. |                                                                                                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created using n8n, a no-code integration and automation platform. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.