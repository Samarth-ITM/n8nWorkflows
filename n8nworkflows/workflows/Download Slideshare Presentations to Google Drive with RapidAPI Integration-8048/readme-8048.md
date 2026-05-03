Download Slideshare Presentations to Google Drive with RapidAPI Integration

https://n8nworkflows.xyz/workflows/download-slideshare-presentations-to-google-drive-with-rapidapi-integration-8048


# Download Slideshare Presentations to Google Drive with RapidAPI Integration

### 1. Workflow Overview

This workflow automates downloading presentations from Slideshare and uploading them to Google Drive, integrating RapidAPI for Slideshare content extraction. Its main use case is to provide users with a simple form to submit a Slideshare URL and receive a downloadable PDF stored in Google Drive with public sharing enabled. Failed downloads are logged in Google Sheets for audit and troubleshooting.

The logical blocks are:

- **1.1 Input Reception:** Receiving user input via a form.
- **1.2 Slideshare Download Link Retrieval:** Calling the RapidAPI Slideshare Downloader API to get the PDF link.
- **1.3 API Response Validation:** Checking if the API returned a successful response.
- **1.4 PDF Download:** Downloading the PDF file from the URL provided by the API.
- **1.5 Upload to Google Drive:** Uploading the downloaded PDF into a Google Drive folder.
- **1.6 Setting Google Drive Permissions:** Making the uploaded file publicly viewable.
- **1.7 Error Handling and Logging:** Handling API failures by waiting and logging failed attempts in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user submits a form containing a Slideshare URL.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point to receive user input interactively.  
    - Configuration: Presents a form titled "Slideshare Downloader" with one required field labeled `URL`.  
    - Expressions/Variables: Passes the entered `URL` field onward as `$json.URL`.  
    - Input: External user submission.  
    - Output: JSON object with the URL field.  
    - Version: 2.2  
    - Edge Cases: User submits invalid or empty URL (mitigated by required field).  
    - Sub-workflow: None.

#### 1.2 Slideshare Download Link Retrieval

- **Overview:**  
  Sends the submitted URL to the RapidAPI Slideshare Downloader Pro service to retrieve a downloadable PDF link.

- **Nodes Involved:**  
  - Slideshare downloader

- **Node Details:**

  - **Slideshare downloader**  
    - Type: HTTP Request  
    - Role: Calls external API to get download info.  
    - Configuration:  
      - Method: POST  
      - URL: https://slideshare-downloader-pro.p.rapidapi.com/slideshare.php  
      - Body: multipart-form-data including the submitted URL (`$json.URL`).  
      - Headers include `x-rapidapi-host` and `x-rapidapi-key` (user must supply their RapidAPI key).  
    - Expressions: URL in body is dynamically set from input JSON.  
    - Input: JSON with `URL` from form.  
    - Output: JSON response from API with status and downloadable link.  
    - Version: 4.2  
    - Edge Cases: API key invalid or missing (auth error), network timeout, malformed response.  
    - On error: Continues with regular output (does not stop workflow).  
    - Sub-workflow: None.

#### 1.3 API Response Validation

- **Overview:**  
  Evaluates if the API response indicates success (status code 200). Routes workflow accordingly.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - Type: If  
    - Role: Conditional branching based on API response status.  
    - Configuration: Checks if `$json.status` equals 200 (number, strict type).  
    - Input: API JSON response.  
    - Output:  
      - True path (status 200): proceeds to download PDF.  
      - False path: proceeds to error handling (wait and log).  
    - Version: 2.2  
    - Edge Cases: Missing `status` field, non-numeric status, API errors.  
    - Sub-workflow: None.

#### 1.4 PDF Download

- **Overview:**  
  Downloads the PDF from the URL provided by the successful API response.

- **Nodes Involved:**  
  - Download Pdf

- **Node Details:**

  - **Download Pdf**  
    - Type: HTTP Request  
    - Role: Downloads the raw PDF binary data.  
    - Configuration:  
      - URL set dynamically as `$json.url` from API response.  
      - Default GET method.  
      - No special headers or options.  
    - Input: JSON with `url` field from API response.  
    - Output: Binary PDF data payload for upload.  
    - Version: 4.2  
    - Edge Cases: Broken URL, network timeout, non-PDF content.  
    - Sub-workflow: None.

#### 1.5 Upload to Google Drive

- **Overview:**  
  Uploads the downloaded PDF file into a specified Google Drive folder.

- **Nodes Involved:**  
  - Upload To Google Drive

- **Node Details:**

  - **Upload To Google Drive**  
    - Type: Google Drive  
    - Role: Uploads file binary to Drive.  
    - Configuration:  
      - Drive ID: "My Drive" (user's root drive).  
      - Folder ID: "root" (root folder).  
      - Authentication: OAuth2 credentials (configured with Google Drive account).  
    - Input: Binary PDF from Download Pdf node.  
    - Output: JSON with uploaded file metadata including file ID.  
    - Version: 3  
    - Edge Cases: Permission errors, quota exceeded, invalid folder ID.  
    - Sub-workflow: None.

#### 1.6 Setting Google Drive Permissions

- **Overview:**  
  Sets the uploaded file’s permissions to "Anyone with the link can view," enabling public access.

- **Nodes Involved:**  
  - Google Drive Set Permission

- **Node Details:**

  - **Google Drive Set Permission**  
    - Type: Google Drive  
    - Role: Updates sharing permissions on the uploaded file.  
    - Configuration:  
      - File ID: dynamically retrieved from upload node output `$json.id`.  
      - Operation: Share (set permissions).  
      - Permissions: default to "Anyone with the link can view."  
      - Authentication: Same OAuth2 Google Drive credentials.  
    - Input: JSON output from Upload To Google Drive node.  
    - Output: Provides sharing details including `webViewLink`.  
    - Version: 3  
    - Edge Cases: Permission update failures, invalid file ID.  
    - Sub-workflow: None.

#### 1.7 Error Handling and Logging

- **Overview:**  
  Handles API failures by delaying execution and logging failed URLs to Google Sheets to avoid rapid repeated writes.

- **Nodes Involved:**  
  - Wait  
  - Google Sheets Append Row

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Introduces delay before logging errors.  
    - Configuration: Default wait (duration unspecified, likely default or configured in node).  
    - Input: False path from If node (failed API response).  
    - Output: Triggers Google Sheets append after delay.  
    - Version: 1.1  
    - Edge Cases: Long delay could cause workflow backlog.  
    - Sub-workflow: None.

  - **Google Sheets Append Row**  
    - Type: Google Sheets  
    - Role: Logs failed conversions for auditing.  
    - Configuration:  
      - Operation: Append row.  
      - Sheet Name & Document ID: Provided via URL mode (user must configure).  
      - Authentication: Service account credentials (Google Docs account).  
      - Data appended includes:  
        - `URL` (original Slideshare link)  
        - `Drive_URL`: set to "N/A" indicating failure  
    - Input: JSON from Wait node.  
    - Output: Confirmation of row append.  
    - Version: 4.6  
    - Edge Cases: Permission issues, sheet full or missing, invalid document ID.  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name                | Node Type         | Functional Role                     | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                              |
|--------------------------|-------------------|-----------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger      | Receive user Slideshare URL       | -                      | Slideshare downloader     | 🟢 **1. On form submission**: Trigger, form with URL field                                               |
| Slideshare downloader    | HTTP Request      | Fetch Slideshare download link    | On form submission      | If                       | 🌐 **2. SlideShare Downloader**: Calls RapidAPI to get PDF link                                         |
| If                       | If                | Check API response status          | Slideshare downloader   | Download Pdf (true), Wait (false) | 🔍 **3. If**: Checks status 200; routes to download or error handling                                   |
| Download Pdf             | HTTP Request      | Download PDF binary                | If (true)               | Upload To Google Drive    | ⬇️ **4. Download pdf**: Downloads PDF using received URL                                               |
| Upload To Google Drive   | Google Drive      | Upload PDF to Drive                | Download Pdf            | Google Drive Set Permission | ☁️ **5. Upload To Google Drive**: Uploads PDF to Drive                                                 |
| Google Drive Set Permission | Google Drive    | Make file publicly accessible     | Upload To Google Drive  | -                        | 🔑 **6. Google Drive Set Permission**: Sets "Anyone with link can view" permission                      |
| Wait                     | Wait              | Delay before logging failure      | If (false)              | Google Sheets Append Row  | ⏱️ **8. Wait**: Delays to prevent rapid sheet writes                                                   |
| Google Sheets Append Row | Google Sheets     | Log failures to Google Sheets     | Wait                    | -                        | 📑 **9. Google Sheets Append Row**: Logs failed URLs with "N/A" Drive_URL                              |
| Sticky Note1             | Sticky Note       | Documentation                     | -                      | -                        | See content in section 2                                                                                |
| Sticky Note2             | Sticky Note       | Documentation                     | -                      | -                        | See content in section 2                                                                                |
| Sticky Note3             | Sticky Note       | Documentation                     | -                      | -                        | See content in section 2                                                                                |
| Sticky Note4             | Sticky Note       | Documentation                     | -                      | -                        | See content in section 2                                                                                |
| Sticky Note5             | Sticky Note       | Documentation                     | -                      | -                        | See content in section 2                                                                                |
| Sticky Note6             | Sticky Note       | Documentation                     | -                      | -                        | See content in section 2                                                                                |
| Sticky Note8             | Sticky Note       | Documentation                     | -                      | -                        | See content in section 2                                                                                |
| Sticky Note9             | Sticky Note       | Documentation                     | -                      | -                        | See content in section 2                                                                                |
| Sticky Note              | Sticky Note       | Documentation                     | -                      | -                        | Overview description of the entire workflow                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger (n8n-nodes-base.formTrigger)  
   - Configure:  
     - Form Title: `Slideshare Downloader`  
     - Form Fields: One field named `URL`, required, labeled "URL"  
     - Form Description: `Slideshare Downloader`  
   - This node acts as the workflow trigger.

2. **Create "Slideshare downloader" node**  
   - Type: HTTP Request  
   - Configure:  
     - Method: POST  
     - URL: `https://slideshare-downloader-pro.p.rapidapi.com/slideshare.php`  
     - Content-Type: multipart-form-data  
     - Body Parameters: Add parameter `url` with value `={{ $json.URL }}`  
     - Headers:  
       - `x-rapidapi-host`: `slideshare-downloader-pro.p.rapidapi.com`  
       - `x-rapidapi-key`: Your RapidAPI key (must be configured by user)  
   - Set "On Error" to "Continue" to prevent workflow failure on API errors.

3. **Connect "On form submission" → "Slideshare downloader"**

4. **Create "If" node**  
   - Type: If  
   - Configure:  
     - Condition: Check if `$json.status` equals number `200` (strict type)  
   - Connect "Slideshare downloader" → "If"

5. **Create "Download Pdf" node**  
   - Type: HTTP Request  
   - Configure:  
     - Method: GET (default)  
     - URL: `={{ $json.url }}` (dynamic URL from API response)  
   - Connect "If" (true output) → "Download Pdf"

6. **Create "Upload To Google Drive" node**  
   - Type: Google Drive  
   - Configure:  
     - Drive ID: Select "My Drive"  
     - Folder ID: Select "root" or desired folder  
     - Authentication: OAuth2 credentials linked to a Google Drive account  
   - Connect "Download Pdf" → "Upload To Google Drive"

7. **Create "Google Drive Set Permission" node**  
   - Type: Google Drive  
   - Configure:  
     - Operation: Share (set permissions)  
     - File ID: `={{ $json.id }}` (from Upload node output)  
     - Permissions: Default to "Anyone with the link can view"  
     - Authentication: Same OAuth2 Google Drive credentials  
   - Connect "Upload To Google Drive" → "Google Drive Set Permission"

8. **Create "Wait" node**  
   - Type: Wait  
   - Configure: Keep default wait or set appropriate delay (e.g., 10 seconds)  
   - Connect "If" (false output) → "Wait"

9. **Create "Google Sheets Append Row" node**  
   - Type: Google Sheets  
   - Configure:  
     - Operation: Append row  
     - Document ID: Provide Google Sheets document ID via URL mode  
     - Sheet Name: Specify sheet name via URL mode  
     - Authentication: Service Account credentials for Google Sheets  
     - Data to append:  
       - `URL`: Pass original URL from input  
       - `Drive_URL`: Set to string `"N/A"` to indicate failure  
   - Connect "Wait" → "Google Sheets Append Row"

10. **Ensure credentials are properly created and assigned:**  
    - Google Drive OAuth2 credentials with proper scopes for file upload and permission changes.  
    - Google Sheets Service Account with write permissions to the target sheet.  
    - RapidAPI key for Slideshare Downloader API.

11. **Optional: Add sticky notes** in the workflow canvas for documentation and clarity, matching those described in the summary.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow gracefully handles API errors by logging failures instead of stopping execution.                    | Error handling strategy                                                                                 |
| RapidAPI key must be obtained and replaced in the HTTP Request node for Slideshare downloader API call.           | https://rapidapi.com/                                                                                   |
| Google Drive OAuth2 credentials require Drive API enabled and appropriate OAuth scopes (file upload, sharing).    | https://console.cloud.google.com/apis/library/drive.googleapis.com                                      |
| Google Sheets Service Account must have write access to the target spreadsheet.                                   | https://console.cloud.google.com/apis/library/sheets.googleapis.com                                    |
| To avoid rate limits and rapid writes in Sheets, a Wait node is introduced before logging errors.                 | Workflow performance and reliability best practice                                                    |
| This workflow assumes users input valid Slideshare URLs; no URL validation beyond required field is done.         | Input validation consideration                                                                         |

---

**Disclaimer:**  
The provided text derives exclusively from an automated workflow created in n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.