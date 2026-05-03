Download Threads Videos & Log Results in Google Sheets

https://n8nworkflows.xyz/workflows/download-threads-videos---log-results-in-google-sheets-10153


# Download Threads Videos & Log Results in Google Sheets

---
### 1. Workflow Overview

This workflow automates the process of downloading videos from Threads URLs submitted via a web form. It downloads the video content, uploads it to a specified Google Drive folder, sets the file's sharing permissions to enable easy access, and logs the outcome (success or failure) in Google Sheets. The workflow is designed for users or systems needing an automated pipeline to archive Threads videos while maintaining an accessible log.

The workflow consists of the following logical blocks:

- **1.1 Input Reception**: Receives Threads URL submissions through a web form trigger.
- **1.2 Video Data Retrieval**: Sends the submitted URL to an external Threads Downloader API to fetch video metadata.
- **1.3 Video Availability Check**: Validates if a downloadable video URL was returned by the API.
- **1.4 Video Downloading**: Downloads the video file from the confirmed URL.
- **1.5 Google Drive Upload**: Uploads the downloaded video to a Google Drive folder.
- **1.6 Sharing Permissions Setup**: Configures the uploaded video's sharing settings for public accessibility.
- **1.7 Success Logging**: Logs successful downloads with URLs into a Google Sheets document.
- **1.8 Failure Handling and Logging**: Waits briefly before logging failed download attempts into Google Sheets with "N/A" as the Drive URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures user input from a web form submission, specifically a Threads URL, to trigger the workflow.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (description)

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger (Web form input collector)  
    - Configuration:  
      - Form Title: "Threads Downloader"  
      - Single required field labeled "URL" with placeholder "https://www.threads.net"  
      - Triggers workflow upon form submission  
    - Inputs: Webhook trigger via form URL  
    - Outputs: JSON object containing the submitted URL under `$json.URL`  
    - Edge Cases: Missing or malformed URL input; form validation handles required field  
    - Sticky Note Content: "Triggers the workflow when a user submits a URL through the form."

#### 1.2 Video Data Retrieval

- **Overview:**  
  Sends the submitted Threads URL to a third-party Threads Downloader API to retrieve video metadata including download URLs.

- **Nodes Involved:**  
  - Fetch Threads Video Data  
  - Sticky Note (description)

- **Node Details:**  
  - **Fetch Threads Video Data**  
    - Type: HTTP Request (POST)  
    - Configuration:  
      - URL: `https://threads-downloader1.p.rapidapi.com/threads.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body parameter: `"url"` set dynamically to the submitted URL (`{{$json.URL}}`)  
      - Headers: Required RapidAPI host and API key (key must be supplied)  
    - Inputs: Output of On form submission node  
    - Outputs: API response JSON with video URLs under `$json.video_urls`  
    - On Error: Continues workflow (does not fail on HTTP errors)  
    - Edge Cases: API key invalid/expired, API rate limits, malformed URL, network errors  
    - Sticky Note Content: "Sends the submitted URL to the Threads Downloader API to retrieve video data."

#### 1.3 Video Availability Check

- **Overview:**  
  Checks if the API response contains a non-empty downloadable video URL.

- **Nodes Involved:**  
  - Check If Video Exists  
  - Sticky Note (description)

- **Node Details:**  
  - **Check If Video Exists**  
    - Type: If node (conditional branching)  
    - Condition: Checks if the first video URL field `{{$json.video_urls[0].download_url}}` is not empty  
    - Branching:  
      - True (video URL exists): Continues to video download  
      - False (no video URL): Proceeds to failure logging path  
    - Inputs: Output of Fetch Threads Video Data  
    - Outputs:  
      - True: Download Threads Video File node  
      - False: Wait Before Logging Failure node  
    - Edge Cases: Missing or malformed API response, empty or invalid video URLs  
    - Sticky Note Content: "Checks if the API returned a valid video download URL."

#### 1.4 Video Downloading

- **Overview:**  
  Downloads the video file from the URL provided by the Threads Downloader API.

- **Nodes Involved:**  
  - Download Threads Video File  
  - Sticky Note (description)

- **Node Details:**  
  - **Download Threads Video File**  
    - Type: HTTP Request (GET)  
    - Configuration:  
      - URL dynamically assigned: `{{$json.video_urls[0].download_url}}`  
      - Default options (no specific headers or authentication)  
    - Inputs: True branch from Check If Video Exists  
    - Outputs: Binary video file data in response  
    - Edge Cases: HTTP errors, slow download, large file size, timeouts  
    - Sticky Note Content: "Downloads the video file from the provided download URL."

#### 1.5 Google Drive Upload

- **Overview:**  
  Uploads the downloaded video file to a configured Google Drive folder.

- **Nodes Involved:**  
  - Upload Video to Google Drive  
  - Sticky Note (description)

- **Node Details:**  
  - **Upload Video to Google Drive**  
    - Type: Google Drive node (File upload)  
    - Configuration:  
      - Drive: "My Drive" (default user drive)  
      - Folder: Root folder (`"/ (Root folder)"`) or specific folder if changed  
      - Uses binary data from previous node as file content  
    - Credentials: Google Drive OAuth2 credentials required  
    - Inputs: Output from Download Threads Video File (binary data)  
    - Outputs: Metadata about uploaded file including `id` and `webViewLink`  
    - Edge Cases: Credential expiration, quota limits, file size limits, network errors  
    - Sticky Note Content: "Uploads the downloaded video to a specified Google Drive folder."

#### 1.6 Sharing Permissions Setup

- **Overview:**  
  Sets sharing permissions on the uploaded video so it can be accessed via a shareable link.

- **Nodes Involved:**  
  - Set Google Drive Sharing Permissions  
  - Sticky Note (description)

- **Node Details:**  
  - **Set Google Drive Sharing Permissions**  
    - Type: Google Drive node (File permission update)  
    - Configuration:  
      - File ID: dynamically set to the uploaded file’s ID (`{{$json.id}}`)  
      - Operation: Share file, set permissions (defaults to accessible)  
      - Authentication: OAuth2 via Google Drive credentials  
    - Inputs: Output of Upload Video to Google Drive  
    - Outputs: Confirmation of permissions update  
    - Edge Cases: Permission update failure, API quota, incorrect file ID, expired credentials  
    - Sticky Note Content: "Sets the uploaded file’s sharing settings so it’s accessible via a link."

#### 1.7 Success Logging

- **Overview:**  
  Appends a new row to a Google Sheets document logging the original Threads URL and the Google Drive link of the uploaded video.

- **Nodes Involved:**  
  - Log Success to Google Sheets  
  - Sticky Note (description)

- **Node Details:**  
  - **Log Success to Google Sheets**  
    - Type: Google Sheets node (Append operation)  
    - Configuration:  
      - Operation: Append row  
      - Document ID and Sheet Name: URLs or IDs of the target Google Sheets document and sheet (must be configured)  
      - Columns:  
        - "URL": Original submitted URL, fetched from On form submission node  
        - "Drive_URL": Shareable Google Drive link from upload node  
      - Mapping mode: Defined below with matching columns for potential duplicates  
      - Authentication: Service Account credentials for Google Sheets  
    - Inputs: Output of Set Google Drive Sharing Permissions  
    - Outputs: Confirmation of append operation  
    - Edge Cases: Credential issues, quota limits, invalid document/sheet IDs, network errors  
    - Sticky Note Content: "Records the original URL and Google Drive link of successfully downloaded videos."

#### 1.8 Failure Handling and Logging

- **Overview:**  
  Introduces a wait/pause before logging failed video download attempts into Google Sheets with “N/A” as the Drive URL value.

- **Nodes Involved:**  
  - Wait Before Logging Failure  
  - Log Failed Download to Google Sheets  
  - Sticky Notes (descriptions)

- **Node Details:**  
  - **Wait Before Logging Failure**  
    - Type: Wait node  
    - Configuration: Default wait (no explicit duration set, likely immediate or default)  
    - Purpose: Mitigate timing issues before writing failure log  
    - Inputs: False branch from Check If Video Exists  
    - Outputs: Log Failed Download to Google Sheets  
    - Edge Cases: Workflow hanging if wait is configured too long or misconfigured  
    - Sticky Note Content: "Adds a pause before logging failed downloads to avoid timing issues."

  - **Log Failed Download to Google Sheets**  
    - Type: Google Sheets node (Append operation)  
    - Configuration:  
      - Operation: Append row  
      - Document ID and Sheet Name: Separate or same Google Sheets document as success logging (must be configured)  
      - Columns:  
        - "URL": Original submitted URL  
        - "Drive_URL": Static value "N/A" indicating failure  
      - Authentication: Service Account credentials  
    - Inputs: Output of Wait Before Logging Failure  
    - Outputs: Confirmation of append operation  
    - Edge Cases: Credential or document issues, network errors  
    - Sticky Note Content: "Logs the original URL with “N/A” for failed video downloads."

---

### 3. Summary Table

| Node Name                        | Node Type               | Functional Role                            | Input Node(s)              | Output Node(s)                          | Sticky Note                                                                                      |
|---------------------------------|-------------------------|--------------------------------------------|----------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger            | Receives URL input via web form             | —                          | Fetch Threads Video Data                | Triggers the workflow when a user submits a URL through the form.                               |
| Fetch Threads Video Data        | HTTP Request            | Sends URL to Threads Downloader API         | On form submission         | Check If Video Exists                   | Sends the submitted URL to the Threads Downloader API to retrieve video data.                   |
| Check If Video Exists           | If Node                 | Checks if a downloadable video URL exists   | Fetch Threads Video Data    | Download Threads Video File, Wait Before Logging Failure | Checks if the API returned a valid video download URL.                                         |
| Download Threads Video File     | HTTP Request            | Downloads the video file                      | Check If Video Exists (true) | Upload Video to Google Drive            | Downloads the video file from the provided download URL.                                       |
| Upload Video to Google Drive    | Google Drive            | Uploads video to Google Drive folder          | Download Threads Video File | Set Google Drive Sharing Permissions    | Uploads the downloaded video to a specified Google Drive folder.                               |
| Set Google Drive Sharing Permissions | Google Drive            | Sets sharing permissions on uploaded file     | Upload Video to Google Drive | Log Success to Google Sheets            | Sets the uploaded file’s sharing settings so it’s accessible via a link.                       |
| Log Success to Google Sheets    | Google Sheets           | Logs successful download URLs and Drive links | Set Google Drive Sharing Permissions | —                                      | Records the original URL and Google Drive link of successfully downloaded videos.              |
| Wait Before Logging Failure     | Wait                    | Pauses before logging failed downloads        | Check If Video Exists (false) | Log Failed Download to Google Sheets    | Adds a pause before logging failed downloads to avoid timing issues.                           |
| Log Failed Download to Google Sheets | Google Sheets           | Logs failed download attempts with "N/A"     | Wait Before Logging Failure | —                                      | Logs the original URL with “N/A” for failed video downloads.                                   |
| Sticky Note                    | Sticky Note             | Descriptive notes for user clarity            | —                          | —                                      | Various notes describing node functions (multiple nodes covered).                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node:**
   - Name: `On form submission`
   - Configure webhook with a unique ID.
   - Set Form Title: "Threads Downloader"
   - Add one form field:
     - Label: "URL"
     - Placeholder: "https://www.threads.net"
     - Required: Yes

3. **Add an HTTP Request node:**
   - Name: `Fetch Threads Video Data`
   - Connect input from `On form submission`.
   - Set URL: `https://threads-downloader1.p.rapidapi.com/threads.php`
   - Method: POST
   - Content-Type: multipart/form-data
   - Body parameters:  
     - Name: "url"  
     - Value: Expression: `{{$json.URL}}`
   - Headers:  
     - `x-rapidapi-host`: `threads-downloader1.p.rapidapi.com`  
     - `x-rapidapi-key`: Your valid RapidAPI key  
   - On error: Continue workflow without failure

4. **Add an If node:**
   - Name: `Check If Video Exists`
   - Connect input from `Fetch Threads Video Data`
   - Condition: Check if expression `{{$json.video_urls[0].download_url}}` is not empty

5. **Add an HTTP Request node:**
   - Name: `Download Threads Video File`
   - Connect input from `Check If Video Exists` true branch
   - Method: GET
   - URL: Expression: `{{$json.video_urls[0].download_url}}`
   - No special headers or authentication required

6. **Add a Google Drive node:**
   - Name: `Upload Video to Google Drive`
   - Connect input from `Download Threads Video File`
   - Operation: Upload file to Google Drive
   - Drive: "My Drive"
   - Folder: Root folder or specify target folder ID
   - Credentials: Configure with Google Drive OAuth2 credentials with upload permissions

7. **Add a Google Drive node:**
   - Name: `Set Google Drive Sharing Permissions`
   - Connect input from `Upload Video to Google Drive`
   - Operation: Share file
   - Resource: File
   - File ID: Expression `{{$json.id}}` from previous node
   - Credentials: Use same Google Drive OAuth2 credentials

8. **Add a Google Sheets node:**
   - Name: `Log Success to Google Sheets`
   - Connect input from `Set Google Drive Sharing Permissions`
   - Operation: Append row
   - Document ID: Set to your Google Sheets document ID
   - Sheet Name: Set to target sheet name
   - Columns:  
     - URL: Expression `{{$node["On form submission"].json.URL}}`  
     - Drive_URL: Expression `{{$node["Upload Video to Google Drive"].json.webViewLink}}`
   - Authentication: Google Sheets Service Account credentials

9. **Add a Wait node:**
   - Name: `Wait Before Logging Failure`
   - Connect input from `Check If Video Exists` false branch
   - Configure default wait (no delay unless needed)

10. **Add a Google Sheets node:**
    - Name: `Log Failed Download to Google Sheets`
    - Connect input from `Wait Before Logging Failure`
    - Operation: Append row
    - Document ID: Your Google Sheets document ID (can be same or different from success log)
    - Sheet Name: Target sheet name
    - Columns:  
      - URL: Expression `{{$node["On form submission"].json.URL}}`  
      - Drive_URL: Static value `"N/A"`
    - Authentication: Google Sheets Service Account credentials

11. **Add Sticky Notes for documentation:**
    - Add descriptive sticky notes near each logical block explaining purpose, referencing the content described in section 2.

12. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                  |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The RapidAPI key must be valid and have quota to access the Threads Downloader API.                 | API usage requirement                            |
| Google Drive OAuth2 credentials require permission to upload files and manage sharing settings.    | Google Drive API setup                           |
| Google Sheets Service Account must have edit access to target documents to append rows successfully.| Google Sheets API setup                          |
| Form Trigger webhook URL must be shared with users or embedded in a form to collect URL submissions.| User interface for triggering the workflow      |
| The workflow continues on HTTP errors during API call to avoid complete failure; consider error handling.| Robustness design                                |
| Video downloads may be large; ensure n8n server has sufficient resources and timeout settings.      | Performance consideration                         |
| This workflow is suitable for automating archival or backup of publicly shared Threads videos.      | Use case                                         |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.