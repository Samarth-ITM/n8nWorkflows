Download Spotify Music to Google Drive with Automatic Logging in Sheets

https://n8nworkflows.xyz/workflows/download-spotify-music-to-google-drive-with-automatic-logging-in-sheets-6684


# Download Spotify Music to Google Drive with Automatic Logging in Sheets

### 1. Workflow Overview

This workflow automates the process of downloading music tracks from Spotify based on user-submitted Spotify track URLs, storing the downloaded files on Google Drive, and maintaining detailed logs in Google Sheets. It is designed for users who want a seamless way to archive Spotify tracks into personal cloud storage while tracking download statuses and metadata.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the Spotify track URL submitted via a user form.
- **1.2 Validation and API Request:** Validates the input URL and sends a request to a third-party Spotify downloader API to fetch the music download link.
- **1.3 Download and Upload:** Downloads the music file from the obtained URL and uploads it to Google Drive.
- **1.4 Processing and Logging:** Calculates file size, waits for processing completion, and logs success or failure details into Google Sheets.
- **1.5 Error Handling:** Captures failed download attempts and logs them separately.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow upon Spotify track URL submission by a user via a web form.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point to receive user input from a form titled "Spotify Music Downloader" with one required field for the Spotify track link.  
  - Configuration: A form with a single required text field labeled "Link" expecting a Spotify track URL.  
  - Input: User-submitted form data.  
  - Output: JSON containing the submitted Spotify track URL under `$json.Link`.  
  - Edge Cases: Empty or invalid URLs are handled downstream; this node triggers regardless of input validity.  
  - No Sub-Workflow.

---

#### 2.2 Validation and API Request

**Overview:**  
This block validates the provided URL and requests an external Spotify downloader API to retrieve the direct music download URL.

**Nodes Involved:**  
- If (Link check)  
- HTTP Request  
- If1 (Success check)

**Node Details:**  
- **If (Link check)**  
  - Type: Conditional Check (If Node)  
  - Role: Ensures that the `Link` field from the form is not empty before proceeding.  
  - Configuration: Checks if `$json.Link` is not empty.  
  - Input: Output from "On form submission".  
  - Output: Passes data forward if condition true; otherwise, the workflow stops or branches accordingly.  
  - Edge Cases: Empty or malformed link inputs will prevent further processing.  
- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Calls the Spotify downloader API with the submitted link to get downloadable music links.  
  - Configuration:  
    - Method: POST  
    - URL: `https://spotify-downloader11.p.rapidapi.com/spotify-downloader.php`  
    - Headers: `x-rapidapi-host` and `x-rapidapi-key` (API key must be provided)  
    - Body: Multipart form with parameter `url` set to the Spotify track link from the form.  
    - Error Handling: On error, continues the workflow rather than stopping.  
  - Input: Spotify track URL from form submission.  
  - Output: JSON response expected to include a `status` field and a `download_url` if successful.  
  - Edge Cases: API key invalid, API downtime, malformed response, or rate limiting.  
- **If1 (Success check)**  
  - Type: Conditional Check (If Node)  
  - Role: Checks if the API response `status` equals `"success"`.  
  - Input: Output of HTTP Request node.  
  - Output:  
    - True branch: Proceeds to download the music.  
    - False branch: Proceeds to failure handling.  
  - Edge Cases: API may return unexpected statuses or malformed data.

---

#### 2.3 Download and Upload

**Overview:**  
This block downloads the music file from the download URL and uploads it to a specified Google Drive folder.

**Nodes Involved:**  
- Download music  
- Google Drive

**Node Details:**  
- **Download music**  
  - Type: HTTP Request  
  - Role: Downloads the music file using the URL received from the API (`$json.download_url`).  
  - Configuration:  
    - Method: GET (default)  
    - URL: Dynamic from API response.  
  - Input: Download URL from "If1" node's true branch.  
  - Output: Binary data of the music file to be used in the next node.  
  - Edge Cases: Download failures, invalid URLs, timeouts.  
- **Google Drive**  
  - Type: Google Drive node  
  - Role: Uploads the downloaded music file to a specific folder in Google Drive.  
  - Configuration:  
    - Drive ID and Folder ID are set (must correspond to the user’s Google Drive folder).  
    - Uploads the binary data named "data" from the previous node.  
    - Credentials: Google Drive OAuth2 credentials must be configured.  
  - Input: Binary music file from "Download music".  
  - Output: Metadata about the uploaded file, including file size and links.  
  - Edge Cases: OAuth token expiration, insufficient permissions, upload failures.

---

#### 2.4 Processing and Logging

**Overview:**  
This block calculates the file size, waits to ensure the upload is complete, and logs success details into Google Sheets.

**Nodes Involved:**  
- Code  
- Wait1  
- Google Sheets  
- Wait  
- Google Sheets1

**Node Details:**  
- **Code**  
  - Type: Code (JavaScript)  
  - Role: Converts the uploaded file size from bytes to megabytes for logging.  
  - Configuration: Custom JS code that reads file size from the Google Drive node output and formats it to MB (two decimal places).  
  - Input: Output from Google Drive node.  
  - Output: JSON containing file name and size in MB.  
  - Edge Cases: Missing or malformed file size data.  
- **Wait1**  
  - Type: Wait node  
  - Role: Introduces a delay after file size calculation before logging to Google Sheets.  
  - Configuration: Default wait time (not explicitly set; defaults to a short pause).  
  - Input: Output from Code node.  
  - Output: Passes data forward unchanged.  
  - Edge Cases: If wait is too short, subsequent nodes might run before data is fully ready.  
- **Google Sheets**  
  - Type: Google Sheets node  
  - Role: Logs the success record to the main Google Sheet.  
  - Configuration:  
    - Operation: Append or update a row in the sheet.  
    - Document ID and Sheet Name set to specified Google Sheet.  
    - Columns mapped with values including original URL, file size, status "Success", current date, and download/view links from Google Drive.  
    - Authentication: Service account credentials configured.  
  - Input: Data from Wait1 and original form submission.  
  - Output: Confirmation of sheet update.  
  - Edge Cases: Sheet access errors, Google API limits, authentication failures.  
- **Wait**  
  - Type: Wait node  
  - Role: Delay introduced after failure check before logging failure.  
  - Configuration: Default delay with no explicit duration set.  
  - Input: Output from If1 node's false branch (failed download).  
  - Output: Passes data forward unchanged.  
- **Google Sheets1**  
  - Type: Google Sheets node  
  - Role: Logs failure details for downloads that did not succeed.  
  - Configuration:  
    - Similar to Google Sheets node, but logs status as "failed", file size as "N/A", and placeholders for download links.  
    - Uses same Google Sheet and sheet name.  
    - Authentication via service account.  
  - Input: Output from Wait node (failed branch).  
  - Edge Cases: Same as Google Sheets node.

---

#### 2.5 Error Handling and Secondary Logging

**Overview:**  
Handles logging failed download attempts and updates the Google Sheet accordingly, ensuring all attempts are documented.

**Nodes Involved:**  
- Google Sheets2

**Node Details:**  
- **Google Sheets2**  
  - Type: Google Sheets node  
  - Role: Logs minimal information for failed attempts, primarily URL, to track failures separately.  
  - Configuration:  
    - Operates on the same Google Sheet and sheet name as others.  
    - Only the URL field is actively logged; other fields removed from the schema.  
    - Authentication through a service account.  
  - Input: Output from Google Drive node (used in failure path).  
  - Output: Confirmation of logging.  
  - Edge Cases: Authentication or API errors.

---

### 3. Summary Table

| Node Name          | Node Type             | Functional Role                        | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                         |
|--------------------|-----------------------|-------------------------------------|------------------------|------------------------|---------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger          | Captures Spotify track URL input     | —                      | If                     | **On form submission**: Triggered on user form submission of Spotify URL.                         |
| If                 | If Node               | Validates that the link is non-empty | On form submission     | HTTP Request           | **If (Link check)**: Validates form input is not empty before proceeding.                         |
| HTTP Request       | HTTP Request          | Calls Spotify downloader API         | If                     | If1                    | **HTTP Request**: Requests music download link from API using Spotify URL.                        |
| If1                | If Node               | Checks if API response is successful | HTTP Request           | Download music (true), Wait (false) | **If1 (Success check)**: Validates success status of API response.                                |
| Download music     | HTTP Request          | Downloads music file from URL        | If1 (true branch)      | Google Drive            | **Download music**: Downloads track from Spotify downloader API response.                         |
| Google Drive       | Google Drive          | Uploads the music file               | Download music         | Google Sheets2          | **Google Drive**: Uploads downloaded file to Google Drive folder.                                |
| Google Sheets2     | Google Sheets         | Logs minimal info for failed attempts| Google Drive           | Code                    | **Google Sheets2**: Logs failed download attempts separately.                                    |
| Code               | Code (JavaScript)     | Converts file size bytes to MB       | Google Sheets2         | Wait1                   | **Code**: Calculates and formats file size in MB for logging.                                   |
| Wait1              | Wait                  | Delays before final logging          | Code                   | Google Sheets            | **Wait1**: Delay for process synchronization before logging success.                            |
| Google Sheets      | Google Sheets         | Logs success details                 | Wait1                   | —                       | **Google Sheets**: Logs successful download details to Google Sheet.                            |
| Wait               | Wait                  | Delays before logging failure        | If1 (false branch)     | Google Sheets1           | **Wait**: Delay to ensure processing completion before failure logging.                         |
| Google Sheets1     | Google Sheets         | Logs failure details                 | Wait                   | —                       | **Google Sheets1**: Logs failed download info to Google Sheet with status "failed".              |
| Sticky Note        | Sticky Note           | Documentation and explanation        | —                      | —                       | Contains overall workflow description and node-by-node explanations.                            |
| Sticky Note1       | Sticky Note           | Explains "On form submission" node   | —                      | —                       | Explains purpose of form submission node.                                                       |
| Sticky Note2       | Sticky Note           | Explains "If" node (link check)      | —                      | —                       | Explains validation of link presence.                                                           |
| Sticky Note3       | Sticky Note           | Explains "HTTP Request" node         | —                      | —                       | Describes API request to Spotify downloader.                                                    |
| Sticky Note4       | Sticky Note           | Explains "If1" node (success check)  | —                      | —                       | Describes success verification of API response.                                                |
| Sticky Note5       | Sticky Note           | Explains "Download music" node       | —                      | —                       | Details about music file download.                                                              |
| Sticky Note6       | Sticky Note           | Explains "Google Drive" node         | —                      | —                       | Details about upload to Google Drive.                                                           |
| Sticky Note7       | Sticky Note           | Explains "Wait" node                  | —                      | —                       | Explains wait node for timing control between operations.                                      |
| Sticky Note8       | Sticky Note           | Explains "Google Sheets" node        | —                      | —                       | Describes logging of success details to Google Sheet.                                          |
| Sticky Note9       | Sticky Note           | Explains "Code" node                  | —                      | —                       | Explains file size calculation logic.                                                          |
| Sticky Note10      | Sticky Note           | Explains "Google Sheets1" node       | —                      | —                       | Describes logging successful downloads in sheet.                                               |
| Sticky Note11      | Sticky Note           | Explains "Wait1" node                 | —                      | —                       | Describes additional wait node for process timing.                                            |
| Sticky Note12      | Sticky Note           | Explains "Google Sheets2" node       | —                      | —                       | Describes logging of failed downloads separately.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node** ("On form submission")  
   - Type: Form Trigger  
   - Configure form with title "Spotify Music Downloader"  
   - Add one required text field labeled "Link" with placeholder "https://open.spotify.com/track/..."  
   - This node triggers the workflow when the user submits a track URL.

2. **Add an If Node** ("If")  
   - Condition: Check if `{{$json.Link}}` is not empty (string not empty).  
   - Connect output of "On form submission" to "If".  
   - True branch continues; False branch stops or ignores.

3. **Add HTTP Request Node** ("HTTP Request")  
   - Method: POST  
   - URL: `https://spotify-downloader11.p.rapidapi.com/spotify-downloader.php`  
   - Headers:  
     - `x-rapidapi-host`: `spotify-downloader11.p.rapidapi.com`  
     - `x-rapidapi-key`: *Your RapidAPI key here*  
   - Body parameters: multipart-form-data with field `url` set to `={{$json.Link}}`  
   - Connect True output of "If" to this node.  
   - On error: Continue workflow (do not stop).

4. **Add If Node** ("If1")  
   - Condition: Check if `{{$json.status}}` equals `"success"`.  
   - Connect "HTTP Request" output to "If1".  
   - True branch for success; False branch for failure.

5. **Add HTTP Request Node** ("Download music")  
   - Method: GET (default)  
   - URL: `={{$json.download_url}}`  
   - Connect True branch of "If1" to this node.

6. **Add Google Drive Node** ("Google Drive")  
   - Operation: Upload file  
   - Drive ID: Set your Google Drive ID  
   - Folder ID: Set your target folder ID inside the Drive where music files will be stored  
   - Input Data Field: Use binary data from "Download music" node  
   - Credentials: Set up Google Drive OAuth2 credentials  
   - Connect "Download music" output to "Google Drive".

7. **Add Google Sheets Node** ("Google Sheets2")  
   - Operation: Append or update row  
   - Document ID: Set your Google Sheets document ID  
   - Sheet Name: Set sheet to work with (e.g., "Sheet1")  
   - Mapping: Log only URL field from `On form submission` node  
   - Authentication: Use Google Sheets service account credentials  
   - Connect "Google Drive" output to "Google Sheets2".

8. **Add Code Node** ("Code")  
   - JavaScript code to convert file size in bytes to MB:  
     ```javascript
     function kbToMb(kb) {
         return (kb / 1024).toFixed(2);
     }
     function bytesToKb(bytes) {
         return bytes / 1024;
     }
     let fileSizeInBytes = $('Google Drive').first().json.size;
     let fileSizeInKb = bytesToKb(fileSizeInBytes);
     let fileSizeInMb = kbToMb(fileSizeInKb);
     return [{ json: { fileName: $input.first().json.name, fileSizeInMb } }];
     ```  
   - Connect "Google Sheets2" output to "Code".

9. **Add Wait Node** ("Wait1")  
   - Default wait (no parameters needed)  
   - Connect "Code" output to "Wait1".

10. **Add Google Sheets Node** ("Google Sheets")  
    - Operation: Append or update row  
    - Document ID and Sheet Name same as above  
    - Mapping fields:  
      - URL: `={{$('On form submission').item.json.Link}}`  
      - Size: `={{$('Code').item.json.fileSizeInMb}} MB`  
      - Status: `"Success"`  
      - Created at: `={{$now.format('dd-MM-yyyy')}}`  
      - Download Link: `={{$('Google Drive').item.json.webContentLink}}`  
      - Web View Link: `={{$('Google Drive').item.json.webViewLink}}`  
    - Authentication: Service account  
    - Connect "Wait1" to this node.

11. **Add Wait Node** ("Wait")  
    - Default wait  
    - Connect False branch of "If1" to this node.

12. **Add Google Sheets Node** ("Google Sheets1")  
    - Operation: Append or update row  
    - Same Doc ID and Sheet Name  
    - Mapping fields:  
      - URL: `={{$('On form submission').item.json.Link}}`  
      - Size: `"N/A"`  
      - Status: `"failed"`  
      - Created at: `={{$now.format('dd-MM-yyyy')}}`  
      - Download Link: `"N/A"`  
      - Web View Link: `"N/A"`  
    - Authentication: Service account  
    - Connect "Wait" to this node.

13. **Connect Nodes** accordingly to maintain the flow:  
    - On form submission → If  
    - If (true) → HTTP Request  
    - HTTP Request → If1  
    - If1 (true) → Download music → Google Drive → Google Sheets2 → Code → Wait1 → Google Sheets  
    - If1 (false) → Wait → Google Sheets1

14. **Credentials Setup:**  
    - Set up Google Drive OAuth2 credentials with proper scope to upload files.  
    - Set up Google Sheets Service Account credentials with permission to append/update rows in the designated sheet.  
    - Obtain RapidAPI key for Spotify downloader API.

15. **Final Notes:**  
    - Ensure folder and document IDs are correct and accessible with the credentials.  
    - Validate API key and endpoints.  
    - Set appropriate webhook URLs or expose form trigger for user access.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                   | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow allows users to download music from Spotify by pasting a track URL, saving the file to Google Drive, and logging details in Google Sheets. | Overall workflow purpose and high-level summary.                                                                |
| Use the RapidAPI Spotify Downloader API at https://rapidapi.com/ to fetch download links; ensure you have a valid API key.                     | External API dependency, requires user registration and key.                                                    |
| Google Drive OAuth2 credentials must have permissions to upload files to the specified Drive and folder.                                        | Credential requirements for Google Drive node.                                                                  |
| Google Sheets service account must have editor permissions on the target spreadsheet.                                                          | Credential requirements for Google Sheets nodes.                                                                |
| The workflow includes built-in error handling with conditional nodes and waits to ensure data integrity and smooth processing.                 | Workflow robustness and error management.                                                                        |
| The form trigger requires exposing the webhook URL to end users securely.                                                                      | Deployment consideration for the "On form submission" node.                                                     |
| This workflow can be extended to support playlists or albums by modifying input and API parameters accordingly.                               | Potential enhancement note.                                                                                       |

---

**Disclaimer:**  
The provided text is extracted exclusively from an n8n automated workflow. The processing complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.