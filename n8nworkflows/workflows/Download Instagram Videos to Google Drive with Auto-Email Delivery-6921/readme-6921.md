Download Instagram Videos to Google Drive with Auto-Email Delivery

https://n8nworkflows.xyz/workflows/download-instagram-videos-to-google-drive-with-auto-email-delivery-6921


# Download Instagram Videos to Google Drive with Auto-Email Delivery

### 1. Workflow Overview

This workflow automates the process of downloading Instagram videos as MP4 files, uploading them to Google Drive, and emailing the download link to the user. The key use case is to provide an easy-to-use web form where users submit an Instagram video URL and their email, and receive a downloadable MP4 link without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input (Instagram URL and email) via a web form.
- **1.2 Video Conversion Request:** Sends the Instagram URL to a RapidAPI service that processes and converts the video to MP4.
- **1.3 API Response Validation:** Checks the API response for errors to ensure successful video processing.
- **1.4 Video Download:** Downloads the converted MP4 video file from the API.
- **1.5 Google Drive Upload:** Uploads the downloaded MP4 file to Google Drive via a Service Account.
- **1.6 File Permission Setting:** Sets sharing permissions on the uploaded file to make it accessible via a public link.
- **1.7 Email Delivery:** Sends an automated email to the user with the Google Drive download link.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block obtains user inputs from a web form, specifically the Instagram video URL and the recipient's email address, to start the workflow.

- **Nodes Involved:**  
  - `n8n Form Trigger`

- **Node Details:**  
  - **Type:** Form Trigger  
  - **Configuration:**  
    - Form title: "Instagram To Mp4"  
    - Two fields:  
      - `url` (Instagram video URL)  
      - `email` (user email address)  
    - Webhook URL automatically generated to receive form submissions  
  - **Key Expressions/Variables:**  
    - `$json.url` to capture the Instagram video URL  
    - `$json.email` to capture the recipient’s email  
  - **Connections:**  
    - Output connected to `API Request` node  
  - **Edge Cases / Potential Failures:**  
    - Missing or invalid URL or email inputs may cause downstream errors  
    - Webhook misconfiguration or failure to receive requests  
  - **Version:** 2.2  

---

#### 1.2 Video Conversion Request

- **Overview:**  
  Sends the Instagram video URL to the RapidAPI Instagram Video Downloader API, which processes the video and returns an MP4 download link and metadata.

- **Nodes Involved:**  
  - `API Request`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://instagram-video-downloader13.p.rapidapi.com/index.php`  
    - Headers include:  
      - `x-rapidapi-host: instagram-video-downloader13.p.rapidapi.com`  
      - `x-rapidapi-key: YOUR_API_KEY` (replace with actual API key)  
    - Body: multipart-form-data with parameter `url` set to `={{ $json.url }}`  
    - Sends headers and body in request  
  - **Key Expressions:**  
    - Uses user-submitted URL via `$json.url`  
  - **Connections:**  
    - Input from `n8n Form Trigger`  
    - Output to `Check for API Error`  
  - **Edge Cases / Potential Failures:**  
    - Invalid or private Instagram URLs causing API errors  
    - API rate limits or key misconfiguration  
    - Network timeouts or unexpected API responses  
  - **Version:** 4.2  

---

#### 1.3 API Response Validation

- **Overview:**  
  Checks if the API response contains an error flag. If no error (`error === false`), the workflow continues; otherwise, execution stops.

- **Nodes Involved:**  
  - `Check for API Error`

- **Node Details:**  
  - **Type:** If node (conditional branching)  
  - **Configuration:**  
    - Condition: Checks that the field `error` in API response JSON is boolean `false`  
    - Only proceeds if no error  
  - **Key Expressions:**  
    - `={{ $json.error }}` evaluated as boolean false  
  - **Connections:**  
    - Input from `API Request`  
    - Output (true branch) to `Download Instagram Video`  
  - **Edge Cases / Potential Failures:**  
    - Unexpected API response structure causing condition to fail  
    - False negatives if API does not return an `error` field  
  - **Version:** 2.2  

---

#### 1.4 Video Download

- **Overview:**  
  Downloads the MP4 video file from the URL provided by the API. Uses headers mimicking a browser to avoid request blocks.

- **Nodes Involved:**  
  - `Download Instagram Video`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - URL: `={{ $json.medias[0].url }}` (the first media URL from API response)  
    - Method: GET (default)  
    - Headers:  
      - `User-Agent`: `Mozilla/5.0 (Windows NT 10.0; Win64; x64)`  
      - `Accept`: `*/*`  
    - Options:  
      - Timeout set very high (10000000 ms) to accommodate large downloads  
      - Follows redirects up to 10 times  
      - Never error on HTTP status, full response captured  
  - **Key Expressions:**  
    - Uses dynamic media URL from API response  
  - **Connections:**  
    - Input from `Check for API Error` (true branch)  
    - Output to `Upload To Google Drive`  
  - **Edge Cases / Potential Failures:**  
    - URL may expire or be inaccessible  
    - Download interruption or timeout  
    - Large file size exceeding limits  
  - **Version:** 4.2  

---

#### 1.5 Google Drive Upload

- **Overview:**  
  Uploads the downloaded MP4 file to Google Drive using a Google Service Account for authentication.

- **Nodes Involved:**  
  - `Upload To Google Drive`

- **Node Details:**  
  - **Type:** Google Drive node  
  - **Configuration:**  
    - Operation: Upload file  
    - Drive: "My Drive" (default drive)  
    - Folder: root folder  
    - File name: uses incoming data (`=` means dynamic)  
    - Authentication: Service Account  
  - **Credentials:**  
    - Google API credentials linked to a Service Account with Drive API access  
  - **Connections:**  
    - Input from `Download Instagram Video`  
    - Output to `Set permissions Google Drive`  
  - **Edge Cases / Potential Failures:**  
    - Insufficient permissions or expired credentials  
    - File upload failure due to size or network  
    - Naming conflicts or invalid file names  
  - **Version:** 3  

---

#### 1.6 File Permission Setting

- **Overview:**  
  Changes the uploaded file’s sharing permissions to “anyone with the link can view,” enabling public access.

- **Nodes Involved:**  
  - `Set permissions Google Drive `

- **Node Details:**  
  - **Type:** Google Drive node  
  - **Configuration:**  
    - Operation: Share file  
    - File ID: dynamic from previous node (`={{ $json.resource.id }}`)  
    - Permissions: Role = reader, Type = anyone  
    - Authentication: Service Account  
  - **Credentials:**  
    - Same Google API Service Account as upload node  
  - **Connections:**  
    - Input from `Upload To Google Drive`  
    - Output to `Deliver Download Link to User`  
  - **Edge Cases / Potential Failures:**  
    - Permission errors due to API or Drive settings  
    - Incorrect file ID reference  
  - **Version:** 3  

---

#### 1.7 Email Delivery

- **Overview:**  
  Sends an email to the user containing the public Google Drive link to the MP4 video file.

- **Nodes Involved:**  
  - `Deliver Download Link to User`

- **Node Details:**  
  - **Type:** Email Send node  
  - **Configuration:**  
    - To: `={{ $json.email }}` (user’s email from form input)  
    - From: `your@email.com` (replace with sender email)  
    - Subject: "🎵 Your Instagram MP4 is ready!"  
    - Text: "Your download link: {{ $json.webViewLink }}" (public Google Drive link)  
  - **Credentials:**  
    - SMTP credentials configured (e.g., Gmail or other SMTP server)  
  - **Connections:**  
    - Input from `Set permissions Google Drive`  
    - No outputs (end of workflow)  
  - **Edge Cases / Potential Failures:**  
    - Invalid email addresses causing send failure  
    - SMTP credential or network issues  
    - Email marked as spam or delayed delivery  
  - **Version:** 1  

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                       | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                      |
|----------------------------|---------------------|------------------------------------|-------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| n8n Form Trigger           | Form Trigger        | Captures user input (URL, email)    | —                       | API Request                 | **n8n Form Trigger**  This node captures user input from a web form, specifically the Instagram video URL and the user's email address, to initiate the workflow. |
| API Request                | HTTP Request        | Sends Instagram URL to RapidAPI     | n8n Form Trigger        | Check for API Error          | **API Request**  It sends the submitted instagram To URL to an rapid API which processes the video and converts it into an MP4 format.                         |
| Check for API Error        | If                  | Validates API response for errors   | API Request             | Download Instagram Video     | **Check for API Error**  This node validate error false if yes then will continue to next node else it will stop execution.                                     |
| Download Instagram Video   | HTTP Request        | Downloads MP4 video from API URL    | Check for API Error     | Upload To Google Drive       | **Download Instagram Video**  After waiting, this node fetches the resulting MP4 file or its metadata from the API, confirming the file is ready for the next steps. |
| Upload To Google Drive     | Google Drive        | Uploads MP4 file to Google Drive    | Download Instagram Video | Set permissions Google Drive | **Upload To Google Drive**  Uploads the retrieved MP4 file to Google Drive, providing cloud storage and easy file management.                                   |
| Set permissions Google Drive| Google Drive       | Sets sharing permissions to public  | Upload To Google Drive  | Deliver Download Link to User| **Set permissions Google Drive  (Share)**  Modifies the file’s permissions in Google Drive to allow anyone with the link to access or download the MP4 file.    |
| Deliver Download Link to User| Email Send         | Sends download link to user by email| Set permissions Google Drive | —                         | **Deliver Download Link to User**  Sends an automated email to the user with the link to download their converted MP4 file, completing the user experience.      |
| Sticky Note1               | Sticky Note         | Documentation comment                | —                       | —                           | **n8n Form Trigger**  This node captures user input from a web form, specifically the Instagram video URL and the user's email address, to initiate the workflow. |
| Sticky Note2               | Sticky Note         | Documentation comment                | —                       | —                           | **API Request**  It sends the submitted instagram To URL to an rapid API which processes the video and converts it into an MP4 format.                            |
| Sticky Note3               | Sticky Note         | Documentation comment                | —                       | —                           | **Check for API Error**  This node validate error false if yes then will continue to next node else it will stop execution.                                       |
| Sticky Note4               | Sticky Note         | Documentation comment                | —                       | —                           | **Download Instagram Video**  After waiting, this node fetches the resulting MP4 file or its metadata from the API, confirming the file is ready for the next steps.|
| Sticky Note5               | Sticky Note         | Documentation comment                | —                       | —                           | **Upload To Google Drive**  Uploads the retrieved MP4 file to Google Drive, providing cloud storage and easy file management.                                   |
| Sticky Note6               | Sticky Note         | Documentation comment                | —                       | —                           | **Set permissions Google Drive  (Share)**  Modifies the file’s permissions in Google Drive to allow anyone with the link to access or download the MP4 file.    |
| Sticky Note7               | Sticky Note         | Documentation comment                | —                       | —                           | **Deliver Download Link to User**  Sends an automated email to the user with the link to download their converted MP4 file, completing the user experience.      |
| Sticky Note                | Sticky Note         | Full workflow description           | —                       | —                           | See detailed workflow description and breakdown in section 5 below.                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `n8n Form Trigger` Node:**  
   - Type: Form Trigger  
   - Set form title: `Instagram To Mp4`  
   - Add two form fields:  
     - `url` (text input for Instagram video URL)  
     - `email` (text input for user email)  
   - Save to generate webhook URL.

2. **Create `API Request` Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://instagram-video-downloader13.p.rapidapi.com/index.php`  
   - Content-Type: multipart/form-data  
   - Body parameters: Add parameter `url` with value `={{ $json.url }}`  
   - Header parameters:  
     - `x-rapidapi-host`: `instagram-video-downloader13.p.rapidapi.com`  
     - `x-rapidapi-key`: `YOUR_API_KEY` (replace with your actual RapidAPI key)  
   - Connect output of `n8n Form Trigger` to input of this node.

3. **Create `Check for API Error` Node:**  
   - Type: If node  
   - Condition:  
     - Variable: `={{ $json.error }}`  
     - Operation: Boolean equals false  
   - Connect output of `API Request` to this node.

4. **Create `Download Instagram Video` Node:**  
   - Type: HTTP Request  
   - URL: `={{ $json.medias[0].url }}`  
   - Headers:  
     - `User-Agent`: `Mozilla/5.0 (Windows NT 10.0; Win64; x64)`  
     - `Accept`: `*/*`  
   - Options:  
     - Set timeout to a high value (e.g., 10000000 ms)  
     - Enable redirects (max 10)  
     - Set to never error on HTTP status, get full response  
   - Connect `Check for API Error` true output to this node.

5. **Create `Upload To Google Drive` Node:**  
   - Type: Google Drive  
   - Operation: Upload file  
   - Drive ID: Select "My Drive"  
   - Folder ID: `root` or desired folder  
   - File Name: Set to dynamic filename from downloaded data (e.g., `={{ $json.fileName }}` or default)  
   - Authentication: Use Google Service Account credentials with Drive API access  
   - Connect output of `Download Instagram Video` to this node.

6. **Create `Set permissions Google Drive` Node:**  
   - Type: Google Drive  
   - Operation: Share file  
   - File ID: `={{ $json.resource.id }}` (from previous upload node output)  
   - Permissions: Role = `reader`, Type = `anyone`  
   - Authentication: Same Google Service Account credentials  
   - Connect output of `Upload To Google Drive` to this node.

7. **Create `Deliver Download Link to User` Node:**  
   - Type: Email Send  
   - To Email: `={{ $json.email }}` (from form input)  
   - From Email: Your sending email address (set accordingly)  
   - Subject: "🎵 Your Instagram MP4 is ready!"  
   - Text: `Your download link: {{ $json.webViewLink }}` (link from Google Drive sharing node)  
   - Credentials: SMTP or Gmail credentials configured  
   - Connect output of `Set permissions Google Drive` to this node.

8. **Connect Nodes in Order:**  
   `n8n Form Trigger` → `API Request` → `Check for API Error` (true branch) → `Download Instagram Video` → `Upload To Google Drive` → `Set permissions Google Drive` → `Deliver Download Link to User`

9. **Credential Setup:**  
   - Obtain RapidAPI key and configure in HTTP Request node headers.  
   - Create Google Cloud Service Account with Drive API enabled; download credentials JSON and configure in n8n Google Drive credentials.  
   - Setup SMTP or Gmail credentials in n8n for email sending.

10. **Test Workflow:**  
    - Submit test Instagram URL and email via the form webhook URL.  
    - Verify video downloads, uploads, permission changes, and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                            |
|---------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| Workflow automates converting Instagram videos to MP4, uploads to Google Drive, and emails download link automatically.  | Workflow summary and purpose                                              |
| Requires RapidAPI Instagram Video Downloader subscription and API key.                                                    | https://rapidapi.com/                                                     |
| Requires Google Cloud Service Account with Drive API enabled for file upload and sharing permissions.                     | https://console.cloud.google.com/                                         |
| Email delivery requires SMTP or Gmail credentials configured within n8n.                                                  | n8n Email credentials documentation                                       |
| Set high HTTP request timeout and user-agent headers to ensure successful video download from Instagram URLs.            | Important for handling large files and avoiding blocks                    |
| Ensure form inputs are validated to avoid malformed URLs or email addresses before processing.                            | Prevents workflow errors                                                  |
| Replace placeholder email addresses and API keys with real credentials before deployment.                                 | Security best practice                                                    |
| Sticky notes in the workflow provide detailed contextual explanations for each block.                                     | Visible in n8n editor UI                                                  |

---

**Disclaimer:**  
The provided content originates exclusively from an automated n8n workflow. It strictly follows content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.