Convert Pinterest Videos to MP4 with Google Drive Storage and Email Delivery

https://n8nworkflows.xyz/workflows/convert-pinterest-videos-to-mp4-with-google-drive-storage-and-email-delivery-6839


# Convert Pinterest Videos to MP4 with Google Drive Storage and Email Delivery

### 1. Workflow Overview

This workflow automates the process of converting Pinterest videos into MP4 format, storing them on Google Drive, and sending a download link via email to the user. It is designed for social media managers, content creators, and marketers who want a seamless, cloud-based solution to download and share Pinterest videos without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Capturing user input (Pinterest video URL and email) via a web form.
- **1.2 Video Conversion via API**: Sending the Pinterest URL to an external API to convert the video to MP4.
- **1.3 Processing Delay**: Introducing a wait period to allow the API to complete the conversion.
- **1.4 Video Download**: Downloading the converted MP4 video file from the API.
- **1.5 Cloud Storage Upload**: Uploading the MP4 file to Google Drive.
- **1.6 Permission Setting**: Making the uploaded file publicly accessible via a sharable link.
- **1.7 Email Delivery**: Sending an email to the user with the Google Drive download link.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the Pinterest video URL and user email address through a web form to initiate the workflow.

- **Nodes Involved:**  
  - n8n Form Trigger

- **Node Details:**  
  - **n8n Form Trigger**  
    - Type: Form Trigger  
    - Role: Entry point for user inputs, listens for web form submissions.  
    - Configuration:  
      - Form Title: "Pinterest To Mp4"  
      - Form Description: "Pinterest To Mp4 Downloader"  
      - Fields: "url" (Pinterest video URL), "email" (user email address)  
    - Key expressions: Uses `$json.url` and `$json.email` as input variables downstream.  
    - Input: HTTP request triggered by form submit  
    - Output: Passes submitted data to next node (HTTP Request)  
    - Potential failures: Missing or invalid input fields; malformed URL; form not submitted correctly.  
    - Version: 2.2

---

#### 2.2 Video Conversion via API

- **Overview:**  
  Sends the provided Pinterest URL to an external RapidAPI service that processes and converts the video to MP4 format.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: API client sending POST request to Pinterest Video Downloader API.  
    - Configuration:  
      - Method: POST  
      - URL: `https://pinterest-video-downloader6.p.rapidapi.com/index.php`  
      - Body: multipart-form-data with parameter "url" set to the Pinterest video URL provided by user (`={{ $json.url }}`)  
      - Headers:  
        - `x-rapidapi-host`: "pinterest-video-downloader6.p.rapidapi.com"  
        - `x-rapidapi-key`: API key placeholder `"YOUR_API_KEY"` (must be replaced with a valid key)  
    - Input: Pinterest video URL from Form Trigger  
    - Output: API response containing video metadata and downloadable MP4 links  
    - Potential failures: Invalid or expired API key, network errors, API rate limits, malformed URL input, unexpected API response format.  
    - Version: 4.2

---

#### 2.3 Processing Delay

- **Overview:**  
  Waits to ensure the external API has completed processing and the MP4 video file is ready.

- **Nodes Involved:**  
  - Wait

- **Node Details:**  
  - **Wait**  
    - Type: Wait node  
    - Role: Pauses workflow execution for a certain period (default or configured delay)  
    - Configuration: No explicit delay configured in the JSON; relies on default wait or manual setting  
    - Input: Output from HTTP Request node (API response)  
    - Output: Passes data to HTTP Downloader to fetch the video  
    - Potential failures: Too short or too long wait times causing premature download attempts or excessive delays.  
    - Version: 1.1

---

#### 2.4 Video Download

- **Overview:**  
  Downloads the MP4 video file or metadata from the API response after conversion is confirmed complete.

- **Nodes Involved:**  
  - HTTP Downloader

- **Node Details:**  
  - **HTTP Downloader**  
    - Type: HTTP Request (configured to download file)  
    - Role: Downloads the MP4 video from the URL provided in API response (`={{ $json.medias[0].url }}`)  
    - Configuration:  
      - URL: Extracted dynamically from API response JSON path `medias[0].url`  
      - Options:  
        - Timeout: 10,000,000 ms (very high to prevent premature termination)  
        - Max Redirects: 10  
        - Full Response: true (to access headers and content)  
      - Headers:  
        - User-Agent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"  
        - Accept: "*/*"  
    - Input: API response with media URLs after wait  
    - Output: Binary MP4 file data for upload  
    - Potential failures: Download timeout, invalid or expired URL, network failure, large file size handling, malformed response.  
    - Version: 4.2

---

#### 2.5 Cloud Storage Upload

- **Overview:**  
  Uploads the downloaded MP4 file to Google Drive for persistent cloud storage.

- **Nodes Involved:**  
  - Upload To Google Drive

- **Node Details:**  
  - **Upload To Google Drive**  
    - Type: Google Drive node  
    - Role: Uploads binary MP4 file to Google Drive root folder  
    - Configuration:  
      - Folder: "root" (Google Drive root directory)  
      - Authentication: Service Account OAuth2 credentials  
      - File Name: Dynamically set (empty string in JSON but typically should be set or taken from binary data)  
    - Input: Binary MP4 file from HTTP Downloader  
    - Output: Google Drive file metadata including file ID  
    - Potential failures: Authentication errors, quota exceeded, file size limits, service account permission issues.  
    - Version: 3

---

#### 2.6 Permission Setting

- **Overview:**  
  Sets the uploaded Google Drive file’s permissions to allow public access, enabling anyone with the link to view or download it.

- **Nodes Involved:**  
  - Set permissions Google Drive

- **Node Details:**  
  - **Set permissions Google Drive**  
    - Type: Google Drive node  
    - Role: Modifies file sharing settings to "anyone with the link" reader access  
    - Configuration:  
      - Operation: "share"  
      - Permissions: Role = "reader", Type = "anyone"  
      - File ID: Taken dynamically from previous upload node (`={{ $json.resource.id }}`)  
      - Authentication: Service Account OAuth2 credentials  
    - Input: Google Drive file metadata from upload node  
    - Output: Confirmation of permission changes and updated file metadata  
    - Potential failures: Permission denied, invalid file ID, API rate limits.  
    - Version: 3

---

#### 2.7 Email Delivery

- **Overview:**  
  Sends an email to the user containing the Google Drive download link for the converted MP4 file.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**  
  - **Send Email**  
    - Type: Email Send node  
    - Role: Sends an automated notification email with the download link  
    - Configuration:  
      - To: User’s email address from form (`={{ $json.email }}`)  
      - From: `your@email.com` (must be replaced with a valid sender email)  
      - Subject: "🎵 Your Pinterest MP4 is ready!"  
      - Text: "Your download link: {{ $json.webViewLink }}" (Google Drive sharable link)  
      - SMTP Credentials: Configured via SMTP account (OAuth2 or username/password)  
    - Input: Google Drive file metadata including `webViewLink` field after permission change  
    - Output: Email delivery status  
    - Potential failures: SMTP authentication failure, invalid email address, delivery errors, missing link in JSON.  
    - Version: 1

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                             | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                     |
|---------------------------|------------------------|---------------------------------------------|------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| n8n Form Trigger           | Form Trigger           | Captures Pinterest URL and email input      | -                      | HTTP Request               | **n8n Form Trigger**  \n   This node captures user input from a web form, specifically the Pinterest video URL and the user's email address, to initiate the workflow. |
| HTTP Request              | HTTP Request           | Sends URL to RapidAPI for MP4 conversion    | n8n Form Trigger       | Wait                       | **HTTP Request**  \n   It sends the submitted Pinterest To URL to an rapid API which processes the video and converts it into an MP4 format.                    |
| Wait                      | Wait                   | Pauses to allow video conversion completion | HTTP Request           | HTTP Downloader            | **Wait**  \n   This node introduces a pause in the workflow, ensuring the external API has enough time to complete the MP4 conversion before the workflow continues. |
| HTTP Downloader           | HTTP Request           | Downloads the MP4 video from API response   | Wait                   | Upload To Google Drive     | HTTP Downloader  \n   After waiting, this node fetches the resulting MP4 file or its metadata from the API, confirming the file is ready for the next steps.      |
| Upload To Google Drive     | Google Drive           | Uploads MP4 file to Google Drive             | HTTP Downloader        | Set permissions Google Drive | **Upload To Google Drive**  \n   Uploads the retrieved MP4 file to Google Drive, providing cloud storage and easy file management.                              |
| Set permissions Google Drive | Google Drive         | Sets public read permissions for the file   | Upload To Google Drive | Send Email                 | *Set permissions Google Drive  (Share)**  \n   Modifies the file’s permissions in Google Drive to allow anyone with the link to access or download the MP4 file.  |
| Send Email                | Email Send             | Sends email with Google Drive download link | Set permissions Google Drive | -                      | **Send Email**  \n   Sends an automated email to the user with the link to download their converted MP4 file, completing the user experience.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Add a **Form Trigger** node named "n8n Form Trigger".  
   - Configure form title: "Pinterest To Mp4"  
   - Configure form description: "Pinterest To Mp4 Downloader"  
   - Add two form fields:  
     - `url` (Pinterest video URL)  
     - `email` (user email address)  
   - Save.

2. **Create HTTP Request Node to call Pinterest Video Downloader API**  
   - Add **HTTP Request** node named "HTTP Request".  
   - Set Method to POST.  
   - Set URL to `https://pinterest-video-downloader6.p.rapidapi.com/index.php`.  
   - Under Body Parameters, add parameter:  
     - Name: `url`  
     - Value: `={{ $json.url }}` (from form input)  
   - Set Content Type to `multipart-form-data`.  
   - Under Headers, add:  
     - `x-rapidapi-host`: `pinterest-video-downloader6.p.rapidapi.com`  
     - `x-rapidapi-key`: Replace `"YOUR_API_KEY"` with your valid RapidAPI key.  
   - Connect "n8n Form Trigger" output to this node’s input.

3. **Add Wait Node**  
   - Add a **Wait** node named "Wait".  
   - Configure delay duration as needed (e.g., 10–20 seconds) to ensure API processing completes.  
   - Connect "HTTP Request" output to "Wait".

4. **Add HTTP Request Node to Download MP4**  
   - Add **HTTP Request** node named "HTTP Downloader".  
   - Set Method to GET (default).  
   - Set URL to `={{ $json.medias[0].url }}` (extract MP4 URL from previous API response).  
   - Under Options:  
     - Set Timeout to a high value (e.g., 10,000,000 ms).  
     - Enable redirects, max 10.  
     - Enable full response to access binary data.  
   - Under Headers, add:  
     - `user-agent`: `Mozilla/5.0 (Windows NT 10.0; Win64; x64)`  
     - `accept`: `*/*`  
   - Connect "Wait" output to this node.

5. **Add Google Drive Upload Node**  
   - Add **Google Drive** node named "Upload To Google Drive".  
   - Operation: Upload File.  
   - Folder ID: Set to `root` or specific folder ID.  
   - Set File Name parameter dynamically if possible (e.g., from binary data or a fixed string).  
   - Authentication: Use Google Service Account credentials with sufficient permissions.  
   - Connect "HTTP Downloader" output to this node.

6. **Add Google Drive Permission Setting Node**  
   - Add **Google Drive** node named "Set permissions Google Drive".  
   - Operation: Share.  
   - File ID: `={{ $json.resource.id }}` (taken from upload node output).  
   - Permissions: Role = "reader", Type = "anyone".  
   - Authentication: Use same Google Service Account credentials.  
   - Connect "Upload To Google Drive" output to this node.

7. **Add Email Send Node**  
   - Add **Email Send** node named "Send Email".  
   - To Email: `={{ $json.email }}` (from form input).  
   - From Email: Set your valid sender email address.  
   - Subject: "🎵 Your Pinterest MP4 is ready!"  
   - Text: "Your download link: {{ $json.webViewLink }}" (link from Google Drive sharing permissions node).  
   - Credentials: Configure SMTP or email provider credentials (e.g., SMTP account with OAuth2).  
   - Connect "Set permissions Google Drive" output to this node.

8. **Test Workflow**  
   - Save and activate the workflow.  
   - Submit a test form with a valid Pinterest video URL and email.  
   - Confirm video conversion, upload, permission setting, and email delivery function correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow leverages the [Pinterest Video Downloader API](https://rapidapi.com/skdeveloper/api/pinterest-video-downloader6) for video conversion. RapidAPI key is mandatory and must be obtained and inserted in the HTTP Request header.          | Pinterest Video Downloader API on RapidAPI                       |
| Google Drive integration requires a Service Account with proper Drive API permissions to upload files and modify sharing settings. Ensure the Service Account is shared with relevant Drive folders if needed.                                     | Google Drive API documentation                                   |
| SMTP credentials must be valid and configured properly for reliable email delivery. Use OAuth2 or other secure authentication methods where possible.                                                                                                | n8n Email Node documentation                                     |
| Proper error handling and timeouts should be considered if adapting the workflow, especially to handle API rate limits, network failures, or invalid inputs gracefully.                                                                               | n8n best practices                                               |
| Users must replace placeholder values such as API keys, email addresses, and credentials with their own secure data before deploying the workflow.                                                                                                   | Security best practices                                          |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.