YouTube to MP3 Converter with RapidAPI - Upload to Drive & Email Link

https://n8nworkflows.xyz/workflows/youtube-to-mp3-converter-with-rapidapi---upload-to-drive---email-link-6318


# YouTube to MP3 Converter with RapidAPI - Upload to Drive & Email Link

---

### 1. Workflow Overview

This workflow automates the process of converting a YouTube video into an MP3 file, uploading the resulting audio file to Google Drive, and sending the download link to the user via email. It is designed to simplify and streamline the entire user experience by handling video-to-audio conversion, cloud storage, file sharing permissions, and email notification seamlessly.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception:** Captures user inputs (YouTube video URL and email) through a web form.
- **1.2 MP3 Conversion Request:** Sends the YouTube URL to a third-party API to initiate MP3 conversion.
- **1.3 Conversion Wait:** Pauses the workflow to allow the external API sufficient time to process the video.
- **1.4 MP3 Retrieval:** Fetches the converted MP3 file or its metadata from the API.
- **1.5 Cloud Storage:** Uploads the MP3 file to Google Drive for persistent storage.
- **1.6 Sharing Setup:** Sets file permissions on Google Drive to make the MP3 publicly accessible.
- **1.7 User Notification:** Sends an email to the user containing the download link for their MP3 file.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by collecting the YouTube URL and the recipient's email address using a web form trigger.

- **Nodes Involved:**  
  - `n8n Form Trigger`

- **Node Details:**

  - **n8n Form Trigger**  
    - Type: Form Trigger  
    - Role: Captures user-submitted form data (YouTube URL and email) to start the workflow.  
    - Configuration:  
      - Form titled "YouTube To Mp3" with two fields: "url" and "email".  
      - No additional options set.  
    - Key Expressions/Variables:  
      - `$json.url` for YouTube URL  
      - `$json.email` for user email  
    - Inputs: None (trigger node)  
    - Outputs: Connects to `HTTP Request` node  
    - Version Requirements: n8n version supporting Form Trigger v2.2 or newer  
    - Potential Failures:  
      - Missing or malformed URL/email inputs  
      - Form submission errors or webhook issues  
    - Sub-workflow: None

#### 1.2 MP3 Conversion Request

- **Overview:**  
  This block sends the submitted YouTube URL to a third-party API (via RapidAPI) to initiate conversion from video to MP3.

- **Nodes Involved:**  
  - `HTTP Request`

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Sends POST request to the YouTube-to-MP3 conversion API.  
    - Configuration:  
      - URL: `https://youtube-to-mp3-fast.p.rapidapi.com/output.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameter: `url` with value from `$json.url`  
      - Headers:  
        - `x-rapidapi-host`: `youtube-to-mp3-fast.p.rapidapi.com`  
        - `x-rapidapi-key`: API key placeholder `YOUR_API_KEY` (must be replaced with valid key)  
      - Sends body and headers accordingly  
    - Key Expressions:  
      - Uses `={{ $json.url }}` to dynamically insert the user-submitted URL  
    - Inputs: From `n8n Form Trigger`  
    - Outputs: Connects to `Wait` node  
    - Version Requirements: HTTP Request node v4.2 or higher for multipart-form-data support  
    - Potential Failures:  
      - Invalid or expired API key  
      - API rate limiting or downtime  
      - Network errors or timeouts  
      - Invalid URL causing API errors  
    - Sub-workflow: None

#### 1.3 Conversion Wait

- **Overview:**  
  Introduces a deliberate pause to allow the external API enough time to complete the MP3 conversion before attempting to retrieve the file.

- **Nodes Involved:**  
  - `Wait`

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for a predefined duration (default unspecified, typically a few seconds)  
    - Configuration: No parameters explicitly set (defaults apply)  
    - Inputs: From `HTTP Request`  
    - Outputs: Connects to `HTTP Downloader`  
    - Version Requirements: Wait node v1.1 or newer  
    - Potential Failures:  
      - Workflow timeout if wait duration is too long  
      - Premature continuation if wait set too short  
    - Sub-workflow: None

#### 1.4 MP3 Retrieval

- **Overview:**  
  Retrieves the MP3 file or its metadata from the URL provided by the API after conversion is complete.

- **Nodes Involved:**  
  - `HTTP Downloader`

- **Node Details:**

  - **HTTP Downloader (HTTP Request node)**  
    - Type: HTTP Request  
    - Role: Downloads the MP3 file from the URL received after conversion  
    - Configuration:  
      - URL: dynamically set as `={{ $json.url }}` from previous node output  
      - Headers:  
        - `user-agent`: Set to mimic browser (`Mozilla/5.0 (Windows NT 10.0; Win64; x64)`)  
        - `accept`: `*/*`  
      - Options:  
        - Timeout set very high (10,000,000 ms) to handle large files or slow responses  
        - Redirects allowed up to 10 times  
        - Response configured to not error on HTTP error codes and to return full response  
    - Inputs: From `Wait` node  
    - Outputs: Connects to `Google Drive` node  
    - Version Requirements: HTTP Request v4.2+ to support advanced options  
    - Potential Failures:  
      - Network errors or slow responses causing timeout despite long setting  
      - Invalid or expired download URL  
      - Redirect loops exceeding max redirects  
      - Partial or corrupted downloads  
    - Sub-workflow: None

#### 1.5 Cloud Storage

- **Overview:**  
  Uploads the MP3 file retrieved from the API to Google Drive for storage and further sharing.

- **Nodes Involved:**  
  - `Google Drive`

- **Node Details:**

  - **Google Drive**  
    - Type: Google Drive node  
    - Role: Uploads the MP3 file to Google Drive root folder  
    - Configuration:  
      - Operation: Upload (default)  
      - Drive: "My Drive" (default user drive)  
      - Folder ID: `root` (uploads to root directory)  
      - Authentication: Service Account (using Google API credentials)  
      - File name: dynamically set to `"="` interpreted as empty or default (likely should be dynamic to preserve filename)  
    - Inputs: From `HTTP Downloader` node (file binary data expected)  
    - Outputs: Connects to `Google Drive set permissions` node  
    - Version Requirements: Google Drive node v3+  
    - Potential Failures:  
      - Authentication errors (invalid or expired service account credentials)  
      - API quota exceeded  
      - File upload errors (file too large, network issues)  
    - Sub-workflow: None

#### 1.6 Sharing Setup

- **Overview:**  
  Sets the uploaded MP3 file’s permissions to be publicly viewable and downloadable via a link.

- **Nodes Involved:**  
  - `Google Drive set permissions`

- **Node Details:**

  - **Google Drive set permissions**  
    - Type: Google Drive node  
    - Role: Changes file permissions to allow anyone with the link to access the file  
    - Configuration:  
      - Operation: Share  
      - File ID: taken dynamically from previous upload step (`={{ $json.resource.id }}`)  
      - Permissions set to role: `reader`, type: `anyone`  
      - Authentication: Same Service Account as upload node  
    - Inputs: From `Google Drive` node  
    - Outputs: Connects to `Send Email` node  
    - Version Requirements: Google Drive node v3+  
    - Potential Failures:  
      - Permission update failures due to API limits or invalid file ID  
      - Authentication errors  
    - Sub-workflow: None

#### 1.7 User Notification

- **Overview:**  
  Sends an email to the user with the public Google Drive link so they can download their MP3 file.

- **Nodes Involved:**  
  - `Send Email`

- **Node Details:**

  - **Send Email**  
    - Type: Email Send  
    - Role: Sends a notification email with the MP3 download link to the user’s email address  
    - Configuration:  
      - To Email: dynamically set from user input (`={{ $json.email }}`)  
      - From Email: fixed as `your@email.com` (should be updated accordingly)  
      - Subject: "🎵 Your MP3 is ready!"  
      - Email Body: "Your download link: {{ $json.webViewLink }}" — uses Google Drive’s shared file webViewLink property from the previous node output  
      - SMTP Credentials: Requires configured SMTP account  
    - Inputs: From `Google Drive set permissions` node  
    - Outputs: None (end of workflow)  
    - Version Requirements: Email Send node v1+  
    - Potential Failures:  
      - SMTP authentication or connection errors  
      - Invalid recipient email address  
      - Missing or invalid `webViewLink` property  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                                 | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                   |
|---------------------------|----------------------|------------------------------------------------|------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| n8n Form Trigger          | Form Trigger         | Captures YouTube URL and email from user form  | None                   | HTTP Request                | This node captures user input from a web form, specifically the YouTube video URL and the user's email address.|
| HTTP Request              | HTTP Request         | Sends URL to external API for MP3 conversion   | n8n Form Trigger       | Wait                        | It sends the submitted YouTube URL to an external API which processes the video and converts it into an MP3.  |
| Wait                      | Wait                 | Pauses workflow to allow API processing         | HTTP Request           | HTTP Downloader             | This node introduces a pause in the workflow, ensuring the external API has enough time to complete conversion.|
| HTTP Downloader           | HTTP Request         | Downloads the converted MP3 or metadata         | Wait                   | Google Drive                | After waiting, this node fetches the resulting MP3 file or its metadata from the API, confirming readiness.    |
| Google Drive              | Google Drive         | Uploads MP3 file to Google Drive                 | HTTP Downloader        | Google Drive set permissions| Uploads the retrieved MP3 file to Google Drive, providing cloud storage and easy file management.              |
| Google Drive set permissions | Google Drive       | Sets public sharing permissions on uploaded file | Google Drive           | Send Email                  | Modifies the file’s permissions in Google Drive to allow anyone with the link to access or download the file.  |
| Send Email                | Email Send           | Emails user the download link                     | Google Drive set permissions | None                     | Sends an automated email to the user with the link to download their converted MP3 file, completing the flow. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Add `n8n Form Trigger` node.  
   - Set form title: `YouTube To Mp3`.  
   - Add two form fields: `url` (YouTube video URL) and `email` (user email).  
   - No additional options needed.  
   - This node starts the workflow.

2. **Create HTTP Request Node (Conversion Request)**  
   - Add `HTTP Request` node.  
   - Configure:  
     - URL: `https://youtube-to-mp3-fast.p.rapidapi.com/output.php`  
     - Method: POST  
     - Content-Type: `multipart/form-data`  
     - Body Parameter: Add a parameter named `url` with value `={{ $json.url }}` (dynamic from form input).  
     - Headers:  
       - `x-rapidapi-host`: `youtube-to-mp3-fast.p.rapidapi.com`  
       - `x-rapidapi-key`: Insert your valid RapidAPI key here.  
   - Connect `n8n Form Trigger` to this node.

3. **Add Wait Node**  
   - Add `Wait` node immediately after HTTP Request.  
   - No special parameters needed, but recommend setting a wait time (e.g., 30 seconds) to allow conversion to complete.  
   - Connect `HTTP Request` node output to `Wait`.

4. **Create HTTP Request Node (Downloader)**  
   - Add another `HTTP Request` node, rename to `HTTP Downloader`.  
   - Configure:  
     - URL: `={{ $json.url }}` (dynamic URL from previous API response).  
     - Method: GET (default).  
     - Headers:  
       - `user-agent`: `Mozilla/5.0 (Windows NT 10.0; Win64; x64)`  
       - `accept`: `*/*`  
     - Options:  
       - Timeout: Set very high (e.g., 10000000 ms) to avoid premature timeout.  
       - Follow redirects enabled, max 10 redirects.  
       - Set response to never error and return full response.  
   - Connect `Wait` node output to this node.

5. **Create Google Drive Upload Node**  
   - Add `Google Drive` node.  
   - Set operation to upload file.  
   - Drive: Select "My Drive" or appropriate drive.  
   - Folder ID: `root` (or choose a specific folder).  
   - Authentication: Use Service Account credentials with sufficient Drive permissions.  
   - File Name: Ideally set dynamically or leave default (in the source workflow it is set to `"="`, which likely defaults to original filename).  
   - Connect `HTTP Downloader` output to this node.

6. **Create Google Drive Permission Node**  
   - Add another `Google Drive` node, rename to `Google Drive set permissions`.  
   - Operation: Share  
   - File ID: Set to `={{ $json.resource.id }}` to get the uploaded file’s ID dynamically.  
   - Permissions: Role=`reader`, Type=`anyone` (public link access).  
   - Authentication: Same Google API Service Account credentials.  
   - Connect `Google Drive` upload node output to this node.

7. **Create Email Send Node**  
   - Add `Email Send` node.  
   - Configure SMTP credentials with valid SMTP server.  
   - Set `To Email` to `={{ $json.email }}` (from form input).  
   - Set `From Email` to your valid sender address (e.g., `your@email.com`).  
   - Subject: `🎵 Your MP3 is ready!`  
   - Text: `Your download link: {{ $json.webViewLink }}` — this uses the Google Drive file’s public URL from the permission node output.  
   - Connect `Google Drive set permissions` node output to this node.

8. **Connect Nodes in Order:**  
   `n8n Form Trigger` → `HTTP Request` → `Wait` → `HTTP Downloader` → `Google Drive` → `Google Drive set permissions` → `Send Email`

9. **Credentials Setup:**  
   - Configure RapidAPI credentials with valid API key for YouTube to MP3 API.  
   - Configure Google API credentials for Google Drive access with a service account authorized to upload and share files.  
   - Configure SMTP credentials for email sending (e.g., Gmail SMTP, SendGrid, or other SMTP server).

10. **Testing and Validation:**  
    - Submit test YouTube URL and email via the form URL generated by the `n8n Form Trigger`.  
    - Verify MP3 conversion completes, file uploads, permission sets correctly, and email is received with a working download link.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow enables a fully automated YouTube to MP3 conversion and delivery system without manual intervention, leveraging cloud services and APIs.                                                                                                                                                                                                           | General workflow goal                                                                                         |
| Replace `"YOUR_API_KEY"` with a valid RapidAPI key from https://rapidapi.com/ for the YouTube to MP3 conversion API.                                                                                                                                                                                                                                            | API Key setup                                                                                                 |
| Google Drive Service Account credentials must have permissions to upload files and alter sharing permissions in the target Drive.                                                                                                                                                                                                                                | Google Drive API setup                                                                                        |
| SMTP credentials must be correctly set up for email sending; consider using OAuth2 for Gmail or a reliable SMTP service.                                                                                                                                                                                                                                        | Email sending setup                                                                                           |
| The workflow includes a wait node to handle asynchronous processing delays of the conversion API, avoiding premature download attempts. Adjust the wait time based on observed API performance.                                                                                                                                                                   | Timing considerations                                                                                         |
| For improved user experience, customize email content and branding. Optionally add error handling nodes to catch and notify on failures (e.g., API errors, upload issues).                                                                                                                                                                                         | Enhancement suggestions                                                                                       |
| See the included sticky notes in the workflow for detailed functional explanations of each block and node.                                                                                                                                                                                                                                                     | Included documentation in workflow via sticky notes                                                          |
| Workflow tested on n8n version supporting Form Trigger v2.2, HTTP Request v4.2, Google Drive v3, and Email Send v1. Ensure your n8n instance is up to date to avoid compatibility issues.                                                                                                                                                                         | Version compatibility                                                                                         |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---