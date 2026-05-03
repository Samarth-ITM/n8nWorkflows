Bulk TikTok Video Download Without Watermark to Google Drive with Tracking

https://n8nworkflows.xyz/workflows/bulk-tiktok-video-download-without-watermark-to-google-drive-with-tracking-6171


# Bulk TikTok Video Download Without Watermark to Google Drive with Tracking

### 1. Workflow Overview

This workflow automates the bulk downloading of TikTok videos without watermarks, uploading them to Google Drive, and tracking their status via a Google Sheet. It is designed for users who manage multiple TikTok video URLs and want an efficient way to archive and share these videos publicly.

**Target Use Cases:**  
- Social media managers needing to archive TikTok videos.  
- Teams sharing TikTok content links centrally.  
- Automation enthusiasts looking to streamline repetitive downloads and uploads.

**Logical Blocks:**

- **1.1 Manual Trigger and Data Retrieval**  
  Starts the workflow manually and fetches TikTok URLs from a Google Sheet.

- **1.2 Iterative Processing Loop**  
  Splits data into batches and processes each TikTok URL sequentially.

- **1.3 TikTok Video Downloading**  
  Calls an external TikTok downloader API to obtain direct video links.

- **1.4 Rate Limiting and Download Handling**  
  Waits to avoid API rate limits, then downloads the video file.

- **1.5 Google Drive Upload and Sharing**  
  Uploads the video file to Google Drive and sets public sharing permissions.

- **1.6 Google Sheet Update and Loop Control**  
  Updates the Google Sheet with new Drive URLs and manages workflow pacing for subsequent items.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger and Data Retrieval

**Overview:**  
This block initiates the workflow manually and retrieves TikTok video URLs from a designated Google Sheet using a service account.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get Data From Google Sheets

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Config: No parameters; invoked manually.  
  - Input: None (entry point).  
  - Output: Triggers Google Sheets data retrieval.  
  - Failure Types: None typical; manual start only.

- **Get Data From Google Sheets**  
  - Type: Google Sheets (Read operation)  
  - Role: Reads all rows from a specified Google Sheet containing TikTok URLs.  
  - Config: Uses Service Account authentication; sheet and document IDs set via URL mode.  
  - Input: Trigger from manual node.  
  - Output: Passes rows to the Loop Over Items node.  
  - Failure Types: Authentication errors, invalid document ID, empty or malformed sheets.

---

#### 2.2 Iterative Processing Loop

**Overview:**  
Processes each TikTok URL individually by splitting the data into batches to handle them sequentially.

**Nodes Involved:**  
- Loop Over Items

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each row (TikTok URL) one by one for sequential processing.  
  - Config: Default batch size (one item at a time); no additional options set.  
  - Input: Data from Google Sheets node.  
  - Output: Passes single item to Call TikTok Downloader; triggers next batch after completion.  
  - Failure Types: Empty input data, batch processing interruptions.

---

#### 2.3 TikTok Video Downloading

**Overview:**  
Calls a third-party TikTok video downloader API to obtain direct video download URLs without watermarks.

**Nodes Involved:**  
- Call TikTok Downloader

**Node Details:**

- **Call TikTok Downloader**  
  - Type: HTTP Request (POST)  
  - Role: Sends TikTok URL to RapidAPI TikTok downloader and retrieves video metadata including download links.  
  - Config:  
    - URL: `https://tiktok-video-downloader23.p.rapidapi.com/index.php`  
    - Method: POST  
    - Headers: Includes `x-rapidapi-host` and `x-rapidapi-key` (API key placeholder).  
    - Body: Multipart form-data with TikTok URL from current batch item (`{{$json.url}}`).  
  - Input: Single TikTok URL from Loop Over Items.  
  - Output: JSON response containing video URLs (including `medias[1].url`).  
  - Failure Types: API rate limiting, invalid API key, network failures, invalid TikTok URLs.  
  - Notes: Requires valid RapidAPI key for TikTok downloader service.

---

#### 2.4 Rate Limiting and Download Handling

**Overview:**  
Implements a short wait to avoid hitting API or network limits before downloading the video file.

**Nodes Involved:**  
- Wait  
- Download File

**Node Details:**

- **Wait**  
  - Type: Wait (Delay)  
  - Role: Pauses workflow for an unspecified default time before video download to prevent rate limits.  
  - Config: No parameters set, defaults apply.  
  - Input: Response from TikTok downloader API.  
  - Output: Triggers Download File node.  
  - Failure Types: None typical; long waits may cause timeout if workflow duration limits exist.

- **Download File**  
  - Type: HTTP Request (GET)  
  - Role: Downloads video file from `medias[1].url` obtained from API response.  
  - Config: URL dynamically set to second media URL in the API response JSON.  
  - Input: Triggered after Wait node.  
  - Output: Binary file data passed to upload node.  
  - Failure Types: URL missing or invalid, network errors, timeouts.

---

#### 2.5 Google Drive Upload and Sharing

**Overview:**  
Uploads the downloaded video file to Google Drive and sets its permission to public for easy sharing.

**Nodes Involved:**  
- Upload File In Google Drive  
- Set Public Permission Google Drive

**Node Details:**

- **Upload File In Google Drive**  
  - Type: Google Drive (Upload)  
  - Role: Uploads the downloaded video file to Google Drive root folder.  
  - Config:  
    - Drive: "My Drive"  
    - Folder: root  
    - Authentication: Service Account  
  - Input: Binary video file from Download File.  
  - Output: File metadata including file ID and webViewLink.  
  - Failure Types: Authentication errors, quota limits, file size limits.

- **Set Public Permission Google Drive**  
  - Type: Google Drive (Share File)  
  - Role: Sets uploaded file’s sharing permissions to "anyone with the link" as writer (public access).  
  - Config:  
    - File ID from uploaded file metadata.  
    - Permissions: Role=writer, Type=anyone.  
    - Authentication: Service Account  
  - Input: File metadata from upload node.  
  - Output: Confirmation of permission update.  
  - Failure Types: Permission errors, API rate limits.

---

#### 2.6 Google Sheet Update and Loop Control

**Overview:**  
Updates the Google Sheet row matching the original TikTok URL with the new Google Drive public link, then pauses before processing the next item.

**Nodes Involved:**  
- Update Row In Google Sheet  
- Sleep

**Node Details:**

- **Update Row In Google Sheet**  
  - Type: Google Sheets (Update operation)  
  - Role: Updates the row with matching `url` column by adding the new `driveurl` (Google Drive public link).  
  - Config:  
    - Matching column: `url`  
    - Update columns: `url` and `driveurl`  
    - Uses service account authentication.  
    - Sheet and document via URL mode.  
  - Input: Metadata from Set Public Permission node and original URL from loop item.  
  - Output: Triggers Sleep node.  
  - Failure Types: Row not found, permission errors, API limits.

- **Sleep**  
  - Type: Wait (Delay)  
  - Role: Pauses briefly between batches to ensure stability and avoid rate limits.  
  - Config: Default wait parameters (unspecified duration).  
  - Input: Trigger from Update Row node.  
  - Output: Loops back to Loop Over Items to process next batch item.  
  - Failure Types: None typical.

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                                  | Input Node(s)                   | Output Node(s)                     | Sticky Note                                                                                  |
|--------------------------------|--------------------------|-------------------------------------------------|--------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger           | Workflow manual start point                      | None                           | Get Data From Google Sheets        | ## 🟢 When clicking ‘Execute workflow’<br>Manually triggers the workflow for testing or running on-demand. |
| Get Data From Google Sheets     | Google Sheets            | Fetch TikTok URLs from Google Sheet              | When clicking ‘Execute workflow’| Loop Over Items                   | ## 📄 Get Data From Google Sheets<br>Reads all rows from a specific Google Sheet containing TikTok video URLs. |
| Loop Over Items                 | SplitInBatches           | Iterate through each TikTok URL one by one      | Get Data From Google Sheets      | Call TikTok Downloader (main branch) | ## 🔁 Loop Over Items<br>Processes each row from the sheet one-by-one (batch looping).        |
| Call TikTok Downloader          | HTTP Request             | Get direct download link for TikTok video       | Loop Over Items                 | Wait                              | ## 🌐 Call TikTok Downloader<br>Sends each TikTok URL to the RapidAPI TikTok downloader to get a direct video download link. |
| Wait                           | Wait                     | Delay before downloading video                   | Call TikTok Downloader          | Download File                     | ## ⏳ Wait<br>Adds a short pause before downloading the file (helps prevent rate-limiting).   |
| Download File                  | HTTP Request             | Download TikTok video file                        | Wait                           | Upload File In Google Drive       | ## ⬇️ Download File<br>Downloads the video from the `medias[1].url` provided by the TikTok downloader API. |
| Upload File In Google Drive     | Google Drive             | Upload downloaded video to Google Drive          | Download File                  | Set Public Permission Google Drive| ## ☁️ Upload File In Google Drive<br>Uploads the downloaded video to Google Drive in the root folder. |
| Set Public Permission Google Drive | Google Drive          | Make uploaded file publicly accessible            | Upload File In Google Drive     | Update Row In Google Sheet         | ## 🔓 Set Public Permission Google Drive<br>Makes the uploaded video file publicly accessible by setting sharing permissions. |
| Update Row In Google Sheet      | Google Sheets            | Update sheet row with Google Drive public link   | Set Public Permission Google Drive| Sleep                          | ## ✏️ Update Row In Google Sheet<br>Updates the same row in the sheet by matching `url`, adding the new Drive `webViewLink` in the `driveurl` column. |
| Sleep                          | Wait                     | Pause before processing next batch item          | Update Row In Google Sheet       | Loop Over Items                   | ## 💤 Sleep<br>Pauses briefly and loops back to process the next TikTok URL in the list.      |
| Sticky Note                    | Sticky Note              | Documentation and explanation                     | None                           | None                            | # 📥 Bulk TikTok Video Downloader & Google Drive Uploader... (full content in node details)  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually.

2. **Create Google Sheets Node to Read Data**  
   - Type: Google Sheets (Read)  
   - Authentication: Service Account (configure credentials)  
   - Document ID: Set via URL mode to your Google Sheet ID containing TikTok URLs  
   - Sheet Name: Set via URL mode to your specific sheet/tab  
   - Operation: Read all rows

3. **Create SplitInBatches Node (Loop Over Items)**  
   - Type: SplitInBatches  
   - Purpose: Process one TikTok URL at a time  
   - Connect input from Google Sheets node output.

4. **Create HTTP Request Node for TikTok Downloader**  
   - Type: HTTP Request (POST)  
   - URL: `https://tiktok-video-downloader23.p.rapidapi.com/index.php`  
   - Authentication: None (API key in headers)  
   - Headers:  
     - `x-rapidapi-host`: `tiktok-video-downloader23.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key (replace placeholder)  
   - Body: Multipart form-data with parameter `url` set to `{{$json.url}}`  
   - Connect input from SplitInBatches node (main output).

5. **Create Wait Node (Wait before download)**  
   - Type: Wait  
   - Default delay (optional: set seconds if needed)  
   - Connect input from TikTok Downloader node output.

6. **Create HTTP Request Node (Download File)**  
   - Type: HTTP Request (GET)  
   - URL: Expression set to `{{$json.medias[1].url}}` (second media URL from TikTok API response)  
   - Connect input from Wait node.

7. **Create Google Drive Node (Upload File)**  
   - Type: Google Drive (Upload)  
   - Authentication: Service Account (configure credentials)  
   - Drive: My Drive  
   - Folder: root (or specify folder ID if desired)  
   - Connect input from Download File node.  
   - Configure to accept binary file input.

8. **Create Google Drive Node (Set Public Permission)**  
   - Type: Google Drive (Share File)  
   - File ID: Set via expression from uploaded file, e.g., `{{$json.id}}`  
   - Permissions: Role = writer, Type = anyone (public access)  
   - Connect input from Upload File node.

9. **Create Google Sheets Node (Update Row)**  
   - Type: Google Sheets (Update)  
   - Authentication: Service Account  
   - Document ID and Sheet Name: same as reading node  
   - Matching Column: `url`  
   - Update Columns:  
     - `url`: Set to `{{$json.url}}` (from Loop Over Items)  
     - `driveurl`: Set to `{{$json.webViewLink}}` (from Google Drive upload metadata)  
   - Connect input from Set Public Permission node.

10. **Create Wait Node (Sleep between batches)**  
    - Type: Wait  
    - Default or small delay (optional)  
    - Connect input from Update Row node.

11. **Connect Sleep node output back to SplitInBatches node (for next batch processing).**

12. **Add Sticky Notes to document each step clearly (optional but recommended).**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                     | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates TikTok video downloading, uploading to Google Drive, and tracking using Google Sheets. It significantly reduces manual effort and errors in managing multiple video URLs.                                                               | General workflow description                                                                    |
| The TikTok video downloader API requires a valid RapidAPI key. Obtain one at https://rapidapi.com/ and subscribe to the TikTok downloader API service.                                                                                                         | API key requirement                                                                             |
| Google Service Account credentials require appropriate permissions on both Google Sheets and Google Drive to read, write, upload, and set permissions.                                                                                                           | Google API setup                                                                                |
| Using SplitInBatches with Wait nodes helps avoid API rate limits by processing items sequentially and pausing as needed.                                                                                                                                         | Rate limiting best practice                                                                     |
| For troubleshooting, check API response payloads and error messages, especially around download URLs and Google API permissions.                                                                                                                                | Debugging tips                                                                                  |
| The workflow assumes the Google Sheet has at least two columns: `url` (original TikTok URL) and `driveurl` (to be updated with Google Drive link).                                                                                                             | Data structure requirement                                                                      |
| Sticky notes in the workflow provide detailed explanations for each functional segment. Review them for quick understanding.                                                                                                                                   | Internal documentation                                                                          |

---

**Disclaimer:** The text provided exclusively derives from an n8n automated workflow. It adheres strictly to content policies and does not contain illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.