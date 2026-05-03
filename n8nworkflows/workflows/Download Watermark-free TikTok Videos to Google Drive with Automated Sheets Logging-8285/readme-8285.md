Download Watermark-free TikTok Videos to Google Drive with Automated Sheets Logging

https://n8nworkflows.xyz/workflows/download-watermark-free-tiktok-videos-to-google-drive-with-automated-sheets-logging-8285


# Download Watermark-free TikTok Videos to Google Drive with Automated Sheets Logging

### 1. Workflow Overview

This workflow automates downloading watermark-free TikTok videos in MP4 format, uploading them to Google Drive, and logging the results in Google Sheets. It is designed to be triggered by a user submitting a TikTok video URL through a public form. The workflow handles success and failure cases distinctly, ensuring reliable logging and accessibility of downloaded videos.

The workflow is organized into these logical blocks:

- **1.1 Input Reception:** Triggered by form submission, capturing the TikTok URL.
- **1.2 TikTok Video Retrieval:** Sends the URL to a RapidAPI TikTok downloader to obtain a downloadable MP4 link.
- **1.3 Conditional Processing:** Checks whether the API response indicates success or failure.
- **1.4 Video Download and Upload:** Downloads the MP4 video, uploads it to Google Drive, and sets public sharing permissions.
- **1.5 Logging:** Logs successful downloads with URLs and Drive links or logs failures with an "N/A" marker in Google Sheets.
- **1.6 Failure Delay:** Introduces a wait before logging failures to prevent rapid consecutive writes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures the TikTok video URL via a user-submitted form to initiate the workflow.
- **Nodes Involved:**  
  - On form submission  
  - Sticky Note1

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point; displays a form titled "TikTok to MP4" with one required field for the TikTok URL.  
    - Key Configurations: Single text field labeled "URL", required.  
    - Input: User input via form submission webhook.  
    - Output: JSON with entered URL under `URL`.  
    - Potential Failures: Webhook accessibility issues or malformed URL input (not validated here).  
    - Sticky Note1 explains purpose and output.

---

#### 2.2 TikTok Video Retrieval

- **Overview:** Sends the submitted TikTok URL to a RapidAPI service to retrieve a downloadable MP4 video link without watermark.
- **Nodes Involved:**  
  - TikTok RapidAPI Request  
  - Sticky Note2

- **Node Details:**

  - **TikTok RapidAPI Request**  
    - Type: HTTP Request  
    - Role: Calls the TikTok downloader API via POST multipart/form-data.  
    - Configuration:  
      - URL: `https://tiktok-download-audio-video.p.rapidapi.com/mp3-4.php`  
      - Headers: `x-rapidapi-host` and `x-rapidapi-key` (API key required)  
      - Body: Sends the form-submitted URL as parameter `url`.  
    - Input: JSON with `URL` from form submission.  
    - Output: JSON response including downloadable media links or error status.  
    - OnError: Continues output to allow conditional handling downstream.  
    - Failure Modes: API key invalid/missing, network errors, malformed API response, rate limiting.  
    - Sticky Note2 outlines purpose and output.

---

#### 2.3 Conditional Processing

- **Overview:** Determines success or failure of the API call by checking the `status` field in the response.
- **Nodes Involved:**  
  - If  
  - Sticky Note3

- **Node Details:**

  - **If**  
    - Type: Conditional Node  
    - Role: Routes workflow based on whether API response's `status` equals `"success"`.  
    - Configuration: Checks `$json.status === "success"`.  
    - Inputs: API response JSON.  
    - Outputs:  
      - True path if status is "success" (proceed to video download).  
      - False path if not (to failure handling).  
    - Failure Modes: Missing or unexpected `status` field, expression evaluation errors.  
    - Sticky Note3 explains logic.

---

#### 2.4 Video Download and Upload

- **Overview:** Downloads the MP4 video via URL from API response, uploads it to Google Drive, and sets sharing permissions.
- **Nodes Involved:**  
  - MP4 Downloader  
  - Upload To Google Drive  
  - Google Drive Set Permission  
  - Sticky Note4, Sticky Note5, Sticky Note6

- **Node Details:**

  - **MP4 Downloader**  
    - Type: HTTP Request  
    - Role: Downloads raw MP4 binary.  
    - Configuration: Uses URL from API response's `"DOWNLOAD (WITHOUT WATERMARK)"` field.  
    - Input: JSON from If node's true branch.  
    - Output: Binary MP4 file data.  
    - Failure Modes: Download URL invalid, network timeout, file size too large.  
    - Sticky Note4 describes purpose.

  - **Upload To Google Drive**  
    - Type: Google Drive Node  
    - Role: Uploads MP4 binary to Google Drive root folder.  
    - Configuration:  
      - Drive: "My Drive"  
      - Folder: Root (`root`)  
      - Authentication: OAuth2 Google Drive credential  
    - Input: Binary MP4 from MP4 Downloader.  
    - Output: Google Drive file metadata including file ID.  
    - Failure Modes: Authentication failure, quota exceeded, API errors.  
    - Sticky Note5 details functionality.

  - **Google Drive Set Permission**  
    - Type: Google Drive Node  
    - Role: Sets file permission to "Anyone with the link can view".  
    - Configuration:  
      - File ID from Upload node output  
      - Permission: Share with anyone (public)  
      - Authentication: Same OAuth2 credential  
    - Input: File metadata from Upload node.  
    - Output: File metadata including `webViewLink` for sharing.  
    - Failure Modes: Permission errors, API limits.  
    - Sticky Note6 explains this step.

---

#### 2.5 Logging

- **Overview:** Logs both successful and failed download attempts in Google Sheets with relevant data.
- **Nodes Involved:**  
  - Google Sheets (success logging)  
  - Google Sheets Append Row (failure logging)  
  - Sticky Note7, Sticky Note9

- **Node Details:**

  - **Google Sheets** (success logging)  
    - Type: Google Sheets node (append operation)  
    - Role: Logs original TikTok URL and Google Drive share link.  
    - Configuration:  
      - Document ID and Sheet name configured via URL mode (values hidden in JSON)  
      - Appends row with columns:  
        - `URL` from form submission  
        - `Drive_URL` from Google Drive Set Permission's `webViewLink`  
      - Authentication: Service Account  
    - Input: File metadata with share link.  
    - Output: Confirmation of append operation.  
    - Failure Modes: Authentication issues, sheet not found, API rate limits.  
    - Sticky Note7 explains purpose.

  - **Google Sheets Append Row** (failure logging)  
    - Type: Google Sheets node (append operation)  
    - Role: Logs failed attempts with `Drive_URL` set to `"N/A"`.  
    - Configuration:  
      - Document ID and sheet name configured similarly  
      - Columns:  
        - `URL` from form submission  
        - `Drive_URL`: `"N/A"`  
      - Authentication: Different Service Account  
    - Input: Output from Wait node after failure.  
    - Output: Confirmation of append operation.  
    - Failure Modes: Same as above.  
    - Sticky Note9 describes this node.

---

#### 2.6 Failure Delay

- **Overview:** Introduces a deliberate pause before logging failed attempts to prevent rapid sheet writes.
- **Nodes Involved:**  
  - Wait  
  - Sticky Note8

- **Node Details:**

  - **Wait**  
    - Type: Wait Node  
    - Role: Delay execution after failure path from If node.  
    - Configuration: Default wait (duration not specified explicitly, likely default or zero).  
    - Input: Failure branch from If node.  
    - Output: Passes data to failure logging node.  
    - Failure Modes: None significant; if misconfigured, could cause unwanted delays.  
    - Sticky Note8 explains rationale.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                | Input Node(s)          | Output Node(s)              | Sticky Note                                                      |
|-------------------------|---------------------|------------------------------------------------|------------------------|----------------------------|-----------------------------------------------------------------|
| On form submission      | Form Trigger        | Receive TikTok URL input from user form       | —                      | TikTok RapidAPI Request     | 🟢 **1. On form submission** - Trigger with URL field            |
| TikTok RapidAPI Request | HTTP Request        | Request downloadable MP4 link from API        | On form submission     | If                         | 🌐 **2. Tiktok RapidAPI Request** - Fetch MP4 download link      |
| If                      | If Node             | Check if API response is success or failure   | TikTok RapidAPI Request | MP4 Downloader, Wait        | 🔍 **3. If** - Conditional branch on API response status         |
| MP4 Downloader           | HTTP Request        | Download MP4 video binary                       | If (true)              | Upload To Google Drive      | ⬇️ **4. MP4 Downloader** - Download video file                   |
| Upload To Google Drive   | Google Drive        | Upload MP4 video to Google Drive               | MP4 Downloader         | Google Drive Set Permission | ☁️ **5. Upload To Google Drive** - Store video                   |
| Google Drive Set Permission | Google Drive      | Set public sharing permissions for file        | Upload To Google Drive | Google Sheets               | 🔑 **6. Google Drive Set Permission** - Make file public         |
| Google Sheets            | Google Sheets       | Log successful conversions                      | Google Drive Set Permission | —                       | 📄 **7. Google Sheets** - Log download success                   |
| Wait                     | Wait                | Delay before logging failure                    | If (false)             | Google Sheets Append Row    | ⏱️ **8. Wait** - Delay to avoid rapid sheet writes              |
| Google Sheets Append Row | Google Sheets       | Log failed conversions                          | Wait                   | —                          | 📑 **9. Google Sheets Append Row** - Log download failure        |
| Sticky Note1             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |
| Sticky Note2             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |
| Sticky Note3             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |
| Sticky Note4             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |
| Sticky Note5             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |
| Sticky Note6             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |
| Sticky Note7             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |
| Sticky Note8             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |
| Sticky Note9             | Sticky Note         | Documentation                                   | —                      | —                          | See above                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the “On form submission” node:**  
   - Type: Form Trigger  
   - Configure form title: "TikTok to MP4"  
   - Add one field:  
     - Label: URL  
     - Placeholder: https://tiktok.com/  
     - Required: Yes  
   - This node acts as the webhook trigger.

2. **Add “TikTok RapidAPI Request” node:**  
   - Type: HTTP Request  
   - HTTP Method: POST  
   - URL: `https://tiktok-download-audio-video.p.rapidapi.com/mp3-4.php`  
   - Content Type: multipart-form-data  
   - Add header parameters:  
     - `x-rapidapi-host`: `tiktok-download-audio-video.p.rapidapi.com`  
     - `x-rapidapi-key`: [Your RapidAPI Key]  
   - Add body parameter:  
     - name: `url`  
     - value: `={{ $json.URL }}` (from form submission)  
   - Set onError to "continue" to allow failure handling.  
   - Connect output of form trigger to this node.

3. **Add “If” node:**  
   - Type: If  
   - Condition: Expression  
   - Expression: Check if `$json.status === "success"`  
   - Connect output of TikTok RapidAPI Request to this node.

4. **Add “MP4 Downloader” node:**  
   - Type: HTTP Request  
   - HTTP Method: GET (default)  
   - URL: `={{ $json.data["DOWNLOAD (WITHOUT WATERMARK)"] }}` (from API success response)  
   - Connect “If” node’s true output to this node.

5. **Add “Upload To Google Drive” node:**  
   - Type: Google Drive  
   - Operation: Upload File  
   - Drive ID: "My Drive"  
   - Folder ID: root (or specify target folder)  
   - Credentials: Configure OAuth2 Google Drive credential  
   - Connect “MP4 Downloader” output to this node.

6. **Add “Google Drive Set Permission” node:**  
   - Type: Google Drive  
   - Operation: Share File  
   - File ID: `={{ $json.id }}` from Upload To Google Drive output  
   - Permissions: Anyone with the link can view  
   - Credentials: Same as above  
   - Connect “Upload To Google Drive” output to this node.

7. **Add “Google Sheets” node for success logging:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Authentication: Service Account credential  
   - Document ID: Your Google Sheet ID (set in URL mode)  
   - Sheet name: Your target sheet name (set in URL mode)  
   - Columns to append:  
     - `URL`: `={{ $json.URL }}` (original from form submission)  
     - `Drive_URL`: `={{ $json.webViewLink }}` (from Set Permission node)  
   - Connect “Google Drive Set Permission” output to this node.

8. **Add “Wait” node for failure delay:**  
   - Type: Wait  
   - Leave default settings or specify a delay (e.g., 5 seconds)  
   - Connect “If” node’s false output to this node.

9. **Add “Google Sheets Append Row” node for failure logging:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Authentication: Service Account credential (can be different)  
   - Document ID and Sheet name as above or different as needed  
   - Columns to append:  
     - `URL`: `={{ $json.URL }}` (original URL)  
     - `Drive_URL`: `"N/A"` (string literal)  
   - Connect “Wait” node output to this node.

10. **Add sticky notes at relevant positions to document each block:**  
    - Use the summarized content from Sticky Note1 to Sticky Note9 for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses [RapidAPI TikTok Downloader](https://rapidapi.com/PrineshPatel/api/tiktok-download-audio-video) for MP4 links. | Official API provider for TikTok video download functionality.                                      |
| Google Drive OAuth2 credentials require setting up OAuth consent and enabling Drive API with appropriate scopes.                  | Google Cloud Console setup required.                                                                |
| Service Account credentials for Google Sheets should have editor access to the target spreadsheet.                                 | Necessary for appending rows programmatically.                                                     |
| The form trigger node exposes a webhook URL that should be embedded in a public form or frontend to collect URLs from users.     | Enables user access to start the conversion process.                                               |
| Failure handling includes a wait node to prevent rapid repeated writes to Google Sheets in case of multiple failures.            | Protects API rate limits and avoids data corruption.                                               |
| Ensure the RapidAPI key is kept secure and has sufficient quota to handle expected requests.                                       | Prevents unexpected failures due to rate limiting or invalid credentials.                           |
| The workflow logs both success and failures distinctly for auditing and debugging purposes.                                        | Improves operational transparency and troubleshooting.                                             |

---

**Disclaimer:** The content provided is extracted exclusively from an n8n workflow automation. It complies strictly with all applicable content policies and contains no illegal or offensive material. All data processed are publicly accessible and lawful.