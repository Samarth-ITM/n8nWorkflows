Download Facebook Videos to Google Drive with Automated Logging in Sheets

https://n8nworkflows.xyz/workflows/download-facebook-videos-to-google-drive-with-automated-logging-in-sheets-6923


# Download Facebook Videos to Google Drive with Automated Logging in Sheets

---

### 1. Workflow Overview

This workflow automates the process of downloading Facebook videos as MP4 files, uploading them to Google Drive, and logging both successful and failed attempts in Google Sheets. It is designed for users who submit a Facebook video URL via a form and want to receive a downloadable video link stored safely on Google Drive, with clear logging for tracking purposes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Capture user-submitted Facebook video URLs via a web form.
- **1.2 Facebook Video API Request**: Query a third-party API to retrieve downloadable MP4 video links.
- **1.3 Error Handling Decision**: Check for API response errors and branch accordingly.
- **1.4 Video Download**: Download the MP4 video binary using the obtained media URL.
- **1.5 Upload to Google Drive**: Store the downloaded video on Google Drive.
- **1.6 Set File Sharing Permissions**: Make the uploaded video publicly accessible.
- **1.7 Logging in Google Sheets**: Log successful downloads and failed attempts distinctly.
- **1.8 Delay for Failure Logging**: Introduce a wait before logging failures to avoid rapid successive writes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Captures the Facebook video URL from the user via a web form, which triggers the workflow execution.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point collecting user input  
  - *Configuration:* Presents a form titled "Facebook to MP4" with a single required field "URL" (placeholder: https://facebook.com/).  
  - *Expressions:* The entered URL is accessed downstream as `{{$json.URL}}`.  
  - *Input:* None (trigger node)  
  - *Output:* JSON containing the submitted URL  
  - *Potential Failures:* Form not submitted, empty URL (guarded by required field setting)  
  - *Notes:* Triggers the entire workflow  

#### 2.2 Facebook Video API Request

**Overview:**  
Sends a POST request to a Facebook Video Downloader API to retrieve downloadable video links for the provided Facebook URL.

**Nodes Involved:**  
- Facebook RapidAPI Request

**Node Details:**  

- **Facebook RapidAPI Request**  
  - *Type:* HTTP Request  
  - *Role:* Fetch direct MP4 download links from Facebook video URL  
  - *Configuration:*  
    - Method: POST  
    - URL: `https://facebook-video-downloader11.p.rapidapi.com/index.php`  
    - Headers: `x-rapidapi-host` and `x-rapidapi-key` (API key required, replace `"your key"`)  
    - Body: multipart form-data with field `url` set from the form input `{{$json.URL}}`  
  - *Expressions:* URL dynamically set using the submitted form value.  
  - *Input:* Output from On form submission node  
  - *Output:* JSON response expected to contain media URLs or error info  
  - *Version:* v4.2 or above recommended for multipart form-data support  
  - *Failure Modes:* API key invalid or missing, API rate limits, network issues, malformed URL input  
  - *Error Handling:* Configured to continue workflow even on error (`onError: continueRegularOutput`)  

#### 2.3 Error Handling Decision

**Overview:**  
Checks the API response for an error field. If no error, the workflow proceeds to download; otherwise, it handles failure logging.

**Nodes Involved:**  
- If

**Node Details:**  

- **If**  
  - *Type:* Conditional logic node  
  - *Role:* Branch workflow based on presence of error in API response  
  - *Configuration:* Checks if `$json.error` equals `false` (strict boolean check)  
  - *Input:* Output from Facebook RapidAPI Request  
  - *Outputs:*  
    - True path (no error): Proceed to MP4 Downloader  
    - False path (error present): Proceed to Wait node for failure handling  
  - *Edge Cases:* API response structure changes, missing `error` field, unexpected data types  

#### 2.4 Video Download

**Overview:**  
Downloads the MP4 video file from the media URL received in the API response.

**Nodes Involved:**  
- MP4 Downloader

**Node Details:**  

- **MP4 Downloader**  
  - *Type:* HTTP Request  
  - *Role:* Download binary video data  
  - *Configuration:*  
    - URL: Extracted from the first media URL in the API response at `{{$json.medias[0].url}}`  
    - Method: GET (default)  
    - No special headers or authentication required  
  - *Input:* True output from If node  
  - *Output:* Binary data of the MP4 video file  
  - *Failure Modes:* Invalid media URL, network timeout, large file download issues  

#### 2.5 Upload to Google Drive

**Overview:**  
Uploads the downloaded MP4 video binary to Google Drive in a specified folder.

**Nodes Involved:**  
- Upload To Google Drive

**Node Details:**  

- **Upload To Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Upload file to Google Drive folder  
  - *Configuration:*  
    - Drive: "My Drive"  
    - Folder: Root folder (default)  
    - File content: Binary data from MP4 Downloader  
  - *Input:* Output from MP4 Downloader  
  - *Output:* Metadata about the uploaded file, including file ID  
  - *Credentials:* OAuth2 Google Drive account with permission to upload  
  - *Failure Modes:* Auth token expiration, insufficient permissions, quota limits  

#### 2.6 Set File Sharing Permissions

**Overview:**  
Sets the uploaded Google Drive file's sharing permissions to public view and obtains a shareable link.

**Nodes Involved:**  
- Google Drive Set Permission

**Node Details:**  

- **Google Drive Set Permission**  
  - *Type:* Google Drive node  
  - *Role:* Modify file permissions for public access  
  - *Configuration:*  
    - File ID: Taken dynamically from uploaded file metadata (`{{$json.id}}`)  
    - Operation: Share file with "Anyone with the link can view" permissions  
  - *Input:* Output from Upload To Google Drive  
  - *Output:* Provides `webViewLink` for sharing  
  - *Credentials:* Same OAuth2 Google Drive account as upload  
  - *Failure Modes:* Permission update failure, API quota limits  

#### 2.7 Logging in Google Sheets

**Overview:**  
Logs the original Facebook URL and the Google Drive shareable link into a Google Sheet for successful downloads.

**Nodes Involved:**  
- Google Sheets

**Node Details:**  

- **Google Sheets**  
  - *Type:* Google Sheets node  
  - *Role:* Append row with success data  
  - *Configuration:*  
    - Operation: Append  
    - Document: Specific Google Sheets document by URL  
    - Sheet name: Default first sheet (gid=0)  
    - Columns:  
      - `URL`: Original Facebook video URL from form submission  
      - `Drive_URL`: The Google Drive public link from permission setting  
  - *Input:* Output from Google Drive Set Permission  
  - *Output:* Confirmation of row append  
  - *Credentials:* Google Sheets service account with append privileges  
  - *Failure Modes:* Sheet access denied, rate limiting  

#### 2.8 Delay for Failure Logging

**Overview:**  
Introduces a pause before logging failed download attempts to Google Sheets to avoid rapid consecutive writes.

**Nodes Involved:**  
- Wait

**Node Details:**  

- **Wait**  
  - *Type:* Wait node  
  - *Role:* Delay execution  
  - *Configuration:* Default wait time (unspecified, defaults to a short pause)  
  - *Input:* False branch from If node indicating failure  
  - *Output:* Passes control to failure logging node  
  - *Failure Modes:* Workflow timeout if wait too long  

#### 2.9 Logging Failed Attempts in Google Sheets

**Overview:**  
Logs failed download attempts including the original URL and a placeholder `N/A` for the Drive URL.

**Nodes Involved:**  
- Google Sheets Append Row

**Node Details:**  

- **Google Sheets Append Row**  
  - *Type:* Google Sheets node  
  - *Role:* Append row with failure data  
  - *Configuration:*  
    - Operation: Append  
    - Document & Sheet: Same as success logging  
    - Columns:  
      - `URL`: Original Facebook video URL from form submission  
      - `Drive_URL`: Set as `"N/A"` to indicate failure  
  - *Input:* Output from Wait node  
  - *Output:* Confirmation of row append for failure  
  - *Credentials:* Google Sheets service account  
  - *Failure Modes:* Same as successful logging  

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                            | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                       |
|--------------------------|-------------------------|--------------------------------------------|-------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger            | Capture Facebook video URL input           | (Trigger)                     | Facebook RapidAPI Request         | 🟢 **1. On form submission** - Trigger, collects user URL input                                 |
| Facebook RapidAPI Request| HTTP Request            | Fetch downloadable MP4 video links         | On form submission             | If                               | 🌐 **2. Facebook RapidAPI Request** - API call to get MP4 links                                  |
| If                       | If                      | Check for API error presence                | Facebook RapidAPI Request      | MP4 Downloader (true), Wait (false) | 🔍 **3. If** - Branch on API success or failure                                                 |
| MP4 Downloader            | HTTP Request            | Download MP4 video binary                   | If (true)                     | Upload To Google Drive            | ⬇️ **4. MP4 Downloader** - Downloads the video file                                            |
| Upload To Google Drive    | Google Drive            | Upload MP4 to Google Drive folder           | MP4 Downloader                | Google Drive Set Permission       | ☁️ **5. Upload To Google Drive** - Uploads video file                                           |
| Google Drive Set Permission| Google Drive           | Make uploaded file publicly accessible      | Upload To Google Drive         | Google Sheets                    | 🔑 **6. Google Drive Set Permission** - Sets public sharing and returns shareable link          |
| Google Sheets             | Google Sheets           | Log successful download with URLs           | Google Drive Set Permission    | (End)                           | 📄 **7. Google Sheets** - Logs success in spreadsheet                                           |
| Wait                      | Wait                    | Delay before logging failure                 | If (false)                    | Google Sheets Append Row          | ⏱️ **8. Wait** - Delay before failure logging                                                   |
| Google Sheets Append Row  | Google Sheets           | Log failed download attempts with N/A       | Wait                         | (End)                           | 📑 **9. Google Sheets Append Row** - Logs failures with N/A                                    |
| Sticky Note1              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note2              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note3              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note4              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note5              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note6              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note7              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note8              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note9              | Sticky Note             | Documentation                               | None                         | None                            | See above sticky note content                                                                   |
| Sticky Note               | Sticky Note             | Documentation                               | None                         | None                            | Workflow overview and node-by-node explanation                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Name: "On form submission"  
   - Type: Form Trigger  
   - Configure form:  
     - Title: "Facebook to MP4"  
     - Description: "Facebook to MP4 Converter"  
     - Fields: Add one required field "URL" with placeholder "https://facebook.com/"  
   - This node triggers the workflow upon form submission with the Facebook URL.

2. **Add HTTP Request node for Facebook API:**  
   - Name: "Facebook RapidAPI Request"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://facebook-video-downloader11.p.rapidapi.com/index.php`  
   - Authentication: None (API key passed via headers)  
   - Headers:  
     - `x-rapidapi-host`: `facebook-video-downloader11.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key (replace `"your key"`)  
   - Content Type: multipart/form-data  
   - Body Parameters: Add parameter `url` with value `={{ $json.URL }}` from form submission  
   - On Error: Continue execution (to allow error handling)  
   - Connect "On form submission" output to this node.

3. **Add If node for error checking:**  
   - Name: "If"  
   - Type: If  
   - Condition: Check if `{{$json.error}}` equals boolean `false` (strict comparison)  
   - Connect "Facebook RapidAPI Request" to this node.

4. **Add HTTP Request node for video download:**  
   - Name: "MP4 Downloader"  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: Set to `={{ $json.medias[0].url }}` from API response  
   - Connect True output from "If" to this node.

5. **Add Google Drive Upload node:**  
   - Name: "Upload To Google Drive"  
   - Type: Google Drive  
   - Operation: Upload file  
   - Drive: "My Drive"  
   - Folder: Root (or specify desired folder ID)  
   - File Content: Attach binary data output from "MP4 Downloader"  
   - Credentials: Google Drive OAuth2, authorize and select proper account  
   - Connect output of "MP4 Downloader" to this node.

6. **Add Google Drive Set Permission node:**  
   - Name: "Google Drive Set Permission"  
   - Type: Google Drive  
   - Operation: Share file  
   - File ID: Use expression `={{ $json.id }}` from upload output  
   - Permissions: Set to "Anyone with the link can view"  
   - Credentials: Same Google Drive OAuth2 account  
   - Connect output of "Upload To Google Drive" to this node.

7. **Add Google Sheets node for success logging:**  
   - Name: "Google Sheets"  
   - Type: Google Sheets  
   - Operation: Append  
   - Sheet: Use your Google Sheet URL and sheet name (e.g., first sheet)  
   - Columns:  
     - `URL`: `={{ $('On form submission').item.json.URL }}`  
     - `Drive_URL`: `={{ $json.webViewLink }}` from permission node output  
   - Credentials: Google Sheets service account with access to the document  
   - Connect output of "Google Drive Set Permission" to this node.

8. **Add Wait node for failure delay:**  
   - Name: "Wait"  
   - Type: Wait  
   - Leave default wait time (or customize delay if needed)  
   - Connect False output from "If" node to this node.

9. **Add Google Sheets node for failure logging:**  
   - Name: "Google Sheets Append Row"  
   - Type: Google Sheets  
   - Operation: Append  
   - Sheet: Same as success logging node  
   - Columns:  
     - `URL`: `={{ $('On form submission').item.json.URL }}`  
     - `Drive_URL`: Set fixed value `"N/A"`  
   - Credentials: Same Google Sheets service account  
   - Connect output of "Wait" node to this node.

10. **Add Sticky Notes (optional):**  
    - Add sticky notes adjacent to groups of nodes to describe their function, using the content from the sticky notes in the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow uses **[Facebook Video Downloader API](https://rapidapi.com/skdeveloper/api/facebook-video-downloader11)** for video extraction. | API documentation and usage details for fetching Facebook video links                                          |
| Requires Google Drive OAuth2 credentials with sufficient permissions for file upload and permission changes.                    | Credential setup in n8n for Google Drive                                                                       |
| Requires Google Sheets service account with permission to append rows in the target spreadsheet.                                | Credential setup in n8n for Google Sheets service account                                                       |
| The workflow includes a delay node before logging failures to avoid flooding Google Sheets with rapid consecutive entries.      | Best practice for rate limiting and avoiding API quota issues                                                  |
| Make sure to replace `"your key"` in the HTTP Request node with a valid RapidAPI key for the Facebook Video Downloader API.     | Critical for API authentication                                                                                 |
| The workflow logs both successful and failed attempts separately in the same Google Sheet for easy tracking and audit purposes. | Ensures complete visibility of all user-submitted URLs and outcomes                                            |
| The Google Drive Set Permission node sets the file to "Anyone with the link can view" to ensure accessibility of the video link. | Critical for sharing downloadable links publicly                                                               |
| Form Trigger node requires n8n instance to be accessible via the internet or through proper tunneling for form submission.       | Deployment consideration for end-user access                                                                   |

---

**Disclaimer:** The content provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.

---