Scheduled YouTube Video Uploads with Google Sheets & Drive Integration

https://n8nworkflows.xyz/workflows/scheduled-youtube-video-uploads-with-google-sheets---drive-integration-4900


# Scheduled YouTube Video Uploads with Google Sheets & Drive Integration

### 1. Workflow Overview

This workflow automates the scheduled uploading of videos to YouTube by integrating Google Sheets and Google Drive services. It is designed to run multiple times a day (Monday to Friday at 9 am, 12 pm, and 3 pm), checking a Google Sheet for video metadata and related file information, downloading the video files from Google Drive, uploading them to YouTube, updating the Google Sheet with the upload status, and finally organizing the video files by moving them to a designated folder in Google Drive.

Logical blocks:

- **1.1 Scheduled Trigger**: Time-based initiation of the workflow at specified weekdays and times.
- **1.2 Data Retrieval from Google Sheets**: Fetching metadata and video file IDs from Google Sheets.
- **1.3 Video File Handling in Google Drive**: Retrieving video files and folder information from Google Drive, downloading video content, and moving files post-upload.
- **1.4 YouTube Upload and Status Update**: Uploading the video to YouTube and updating the Google Sheet with the upload status.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically on weekdays at 9 am, 12 pm, and 3 pm to regularly check for new videos to upload.

- **Nodes Involved:**  
  - M-F 9am,12pm,3pm (Schedule Trigger)

- **Node Details:**  
  - **Name:** M-F 9am,12pm,3pm  
  - **Type:** Schedule Trigger  
  - **Configuration:** Set to trigger on Monday through Friday at 9:00, 12:00, and 15:00 hours.  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Connected to “Google Sheets” node to start the data retrieval process.  
  - **Edge Cases:**  
    - Timezone misconfigurations may cause triggers to fire at incorrect local times.  
    - If n8n server is down during scheduled time, the trigger will not run, potentially skipping uploads.  
  - **Version Requirements:** n8n v0.141.0 or later recommended for advanced schedule options.

#### 1.2 Data Retrieval from Google Sheets

- **Overview:**  
  Reads video metadata and file identifiers from a configured Google Sheet to determine which videos to process.

- **Nodes Involved:**  
  - Google Sheets  
  - Get Video File Id

- **Node Details:**  
  - **Google Sheets**  
    - **Type:** Google Sheets node  
    - **Role:** Retrieves rows from a Google Sheet which presumably contains video titles, descriptions, and Google Drive file IDs.  
    - **Configuration:** Likely configured with spreadsheet ID and sheet name (not explicitly detailed).  
    - **Inputs:** Receives trigger from the schedule node.  
    - **Outputs:** Passes data to “Get Video File Id” node.  
    - **Edge Cases:**  
      - Authorization errors if OAuth credentials expire or are revoked.  
      - Empty or malformed rows could lead to missing file IDs or metadata.  
      - API rate limits from Google Sheets API.  
  - **Get Video File Id**  
    - **Type:** Google Drive node  
    - **Role:** Uses file IDs from Google Sheets to query Google Drive for video file metadata.  
    - **Configuration:** Possibly uses expressions to map file IDs from Google Sheets input.  
    - **Inputs:** Receives file ID data from Google Sheets node.  
    - **Outputs:** Passes file metadata to “Download Video Data” node.  
    - **Edge Cases:**  
      - File not found errors if IDs are incorrect or files deleted.  
      - Permission issues if OAuth token lacks Drive read access.

#### 1.3 Video File Handling in Google Drive

- **Overview:**  
  Downloads the video data from Google Drive and, after uploading to YouTube, moves the processed video file to a designated folder for organizational purposes.

- **Nodes Involved:**  
  - Download Video Data  
  - Get Folder Name  
  - Move Video File to Folder

- **Node Details:**  
  - **Download Video Data**  
    - **Type:** Google Drive node  
    - **Role:** Downloads the video binary data from Google Drive to prepare for YouTube upload.  
    - **Configuration:** Uses file metadata passed from the previous node to fetch the file content.  
    - **Inputs:** From “Get Video File Id” node.  
    - **Outputs:** Sends binary video data to “Upload to YouTube” node.  
    - **Edge Cases:**  
      - Download failures due to network issues or file access restrictions.  
      - Large file size could cause timeout or memory constraints.  
  - **Get Folder Name**  
    - **Type:** Google Drive node  
    - **Role:** Retrieves the target folder metadata (likely folder ID or name) to which the uploaded video file should be moved.  
    - **Configuration:** Possibly uses expressions or static configuration to specify folder.  
    - **Inputs:** From “Update Status” node after successful upload to YouTube.  
    - **Outputs:** Passes folder info to “Move Video File to Folder” node.  
    - **Edge Cases:**  
      - Folder not found or permission denied errors.  
  - **Move Video File to Folder**  
    - **Type:** Google Drive node  
    - **Role:** Moves the uploaded video file to the specified folder for organization.  
    - **Configuration:** Uses file ID and folder ID to execute the move operation.  
    - **Inputs:** From “Get Folder Name” node.  
    - **Outputs:** None (final step).  
    - **Edge Cases:**  
      - Move failures due to file locks or permission issues.  
      - Folder ID or file ID missing or invalid.

#### 1.4 YouTube Upload and Status Update

- **Overview:**  
  Uploads the video binary to YouTube using metadata from Google Sheets, then updates the Google Sheet to reflect the upload status.

- **Nodes Involved:**  
  - Upload to YouTube  
  - Update Status

- **Node Details:**  
  - **Upload to YouTube**  
    - **Type:** YouTube node  
    - **Role:** Uploads the video file with associated metadata (title, description, tags) to the YouTube channel linked by credentials.  
    - **Configuration:** Configured to receive binary file data and metadata fields. Uses OAuth2 credentials for YouTube API.  
    - **Inputs:** Receives binary video data from “Download Video Data” node.  
    - **Outputs:** Passes upload result info to “Update Status” node.  
    - **Edge Cases:**  
      - Upload failures due to quota limits or invalid credentials.  
      - Network issues causing incomplete uploads.  
      - Metadata validation errors (e.g., title length, tags).  
  - **Update Status**  
    - **Type:** Google Sheets node  
    - **Role:** Updates the Google Sheet row with the status of the upload (e.g., success, video URL).  
    - **Configuration:** Uses row ID or unique identifier to update specific row cells.  
    - **Inputs:** From “Upload to YouTube” node.  
    - **Outputs:** Passes data to “Get Folder Name” node for next step in file organization.  
    - **Edge Cases:**  
      - Write failures if sheet is locked or credentials expired.  
      - Race conditions if multiple workflow instances write concurrently.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                          | Input Node(s)                | Output Node(s)              | Sticky Note                                 |
|-------------------------|--------------------|----------------------------------------|-----------------------------|-----------------------------|---------------------------------------------|
| M-F 9am,12pm,3pm        | Schedule Trigger    | Time-based initiation of workflow      | None                        | Google Sheets                |                                             |
| Google Sheets           | Google Sheets      | Retrieves video metadata & file IDs    | M-F 9am,12pm,3pm            | Get Video File Id            |                                             |
| Get Video File Id       | Google Drive       | Fetches video file metadata by ID      | Google Sheets               | Download Video Data          |                                             |
| Download Video Data     | Google Drive       | Downloads video binary data             | Get Video File Id           | Upload to YouTube            |                                             |
| Upload to YouTube       | YouTube            | Uploads video to YouTube channel        | Download Video Data         | Update Status                |                                             |
| Update Status           | Google Sheets      | Updates upload status in Google Sheets  | Upload to YouTube           | Get Folder Name              |                                             |
| Get Folder Name         | Google Drive       | Retrieves target folder for organizing  | Update Status               | Move Video File to Folder    |                                             |
| Move Video File to Folder| Google Drive       | Moves uploaded video file to folder     | Get Folder Name             | None                        |                                             |
| Sticky Note             | Sticky Note        | (Empty)                                | None                        | None                        |                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `M-F 9am,12pm,3pm`  
   - Type: Schedule Trigger  
   - Configure to trigger Monday through Friday at 09:00, 12:00, and 15:00 hours local time.

2. **Add a Google Sheets node**  
   - Name: `Google Sheets`  
   - Operation: Read Rows (Get Data)  
   - Configure with Google Sheets credentials (OAuth2)  
   - Specify Spreadsheet ID and Sheet Name containing video metadata and Drive file IDs.  
   - Connect output from `M-F 9am,12pm,3pm` trigger.

3. **Add a Google Drive node**  
   - Name: `Get Video File Id`  
   - Operation: Get File Metadata by ID  
   - Use expressions to get file IDs from Google Sheets node data (e.g., `{{$json["fileId"]}}`)  
   - Connect input from `Google Sheets` node.

4. **Add another Google Drive node**  
   - Name: `Download Video Data`  
   - Operation: Download File  
   - Use file ID from previous node to download the video binary data  
   - Connect input from `Get Video File Id`.

5. **Add a YouTube node**  
   - Name: `Upload to YouTube`  
   - Operation: Upload Video  
   - Configure with YouTube OAuth2 credentials  
   - Map metadata fields (title, description, tags) from Google Sheets data  
   - Attach binary video data from `Download Video Data` node  
   - Connect input from `Download Video Data`.

6. **Add a Google Sheets node**  
   - Name: `Update Status`  
   - Operation: Update Row  
   - Configure with same spreadsheet and sheet as in step 2  
   - Use unique identifiers or row numbers to update upload status and video URL from YouTube upload response  
   - Connect input from `Upload to YouTube`.

7. **Add a Google Drive node**  
   - Name: `Get Folder Name`  
   - Operation: Get Folder Metadata by Name or ID  
   - Configure to specify the target folder where uploaded video files will be moved  
   - Connect input from `Update Status`.

8. **Add a final Google Drive node**  
   - Name: `Move Video File to Folder`  
   - Operation: Move File  
   - Use file ID from earlier nodes and folder ID from `Get Folder Name` to move the video file  
   - Connect input from `Get Folder Name`.

9. **(Optional) Add Sticky Note node**  
   - For annotations or documentation within the workflow editor.

**General Notes:**  
- Ensure all Google APIs and YouTube API OAuth2 credentials have the necessary scopes enabled:  
  - Google Sheets API: `spreadsheets.readonly` and `spreadsheets` for read/write  
  - Google Drive API: `drive.readonly` and `drive.file`  
  - YouTube Data API: `youtube.upload`  
- Use expressions to map data between nodes carefully, especially file IDs and row identifiers.  
- Test the workflow with a small dataset before scheduling.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                         |
|------------------------------------------------------------------------------------------------|----------------------------------------|
| Workflow automates YouTube video uploads triggered at multiple times on weekdays.             | Workflow Scheduling Concept             |
| Requires OAuth2 credentials for Google Sheets, Google Drive, and YouTube API.                  | Credential Setup in n8n Documentation  |
| For Google Drive API, ensure the OAuth token has sufficient permission scopes for file moves. | Google Drive API Documentation          |
| YouTube upload quota limits may apply; monitor YouTube API quota usage regularly.              | https://developers.google.com/youtube/v3/getting-started#quota |
| Proper handling of errors is recommended: implement error workflows or retries for failures.  | n8n Error Handling Best Practices       |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and includes no illegal, offensive, or protected material. All data handled is legal and public.