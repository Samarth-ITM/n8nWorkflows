Create & Publish Inspirational Videos with FFmpeg, Google Drive & YouTube

https://n8nworkflows.xyz/workflows/create---publish-inspirational-videos-with-ffmpeg--google-drive---youtube-11591


# Create & Publish Inspirational Videos with FFmpeg, Google Drive & YouTube

### 1. Workflow Overview

This workflow automates the creation and publishing of inspirational videos by combining video backgrounds, music, and quotes, leveraging Google Drive, Google Sheets, FFmpeg processing (via an external API), and YouTube uploading. It targets marketing teams or content creators who want to automate inspirational video generation and distribution across platforms efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Data Loading**: Retrieves video backgrounds, music files, and quote data from Google Sheets and Google Drive.
- **1.2 Random Content Selection**: Merges and processes retrieved data to randomly select a video background, a music track, and a quote for each video.
- **1.3 Video Generation & Processing**: Uploads the selected assets to a video generation service (likely FFmpeg-based), monitors processing status, and downloads the final video.
- **1.4 File Handling & Upload**: Saves the final video locally, uploads it back to Google Drive, and prepares it for publishing.
- **1.5 Publishing & Status Tracking**: Uploads the video to YouTube, moves it to a playlist, and updates statuses in Google Sheets. Optional disabled nodes suggest planned posting to Facebook, Instagram, and X (Twitter).
- **1.6 Trigger & Scheduling**: Manual and scheduled triggers to start the workflow and handle asynchronous waits, with Telegram notifications for status updates.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Data Loading

**Overview:**  
This block fetches all necessary input data for video creation — video backgrounds, music files, and inspirational quotes — from Google Sheets and Google Drive folders.

**Nodes Involved:**  
- Start AutoClip Workflow (Manual Trigger)  
- Schedule Trigger (Scheduled Trigger)  
- Retrieve Quote Data (Google Sheets)  
- Retrieve Video Background Data (Google Sheets)  
- Retrieve Music Background Data (Google Sheets)  
- List Video Background Files (Google Drive)  
- List Music Background Files (Google Drive)  
- Configure Music Background Folder ID (Set)

**Node Details:**  

- **Start AutoClip Workflow**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual execution of the workflow.  
  - *Connections:* Outputs to Retrieve Quote Data, List Video Background Files, Configure Music Background Folder ID.  
  - *Failure types:* None typical, manual initiation.

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Entry point for automatic scheduled runs.  
  - *Connections:* Outputs to Configure Music Background Folder ID, List Video Background Files, Retrieve Quote Data.  
  - *Failure types:* Cron misconfiguration or scheduling errors.

- **Configure Music Background Folder ID**  
  - *Type:* Set  
  - *Role:* Defines the Google Drive folder ID where music files are stored.  
  - *Configuration:* Static or parameterized folder ID.  
  - *Connections:* Outputs to List Music Background Files.  
  - *Failure types:* Incorrect folder ID leads to empty list or error.

- **Retrieve Quote Data**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves rows containing quotes for video captions.  
  - *Configuration:* Points to specific Google Sheet and range.  
  - *Connections:* Outputs to Merge File Selection Data.  
  - *Failure types:* Auth errors, rate limits, empty data.

- **Retrieve Video Background Data**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves metadata for video backgrounds.  
  - *Connections:* Outputs to Merge File Selection Data.  
  - *Failure types:* Same as above.

- **Retrieve Music Background Data**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves metadata for music tracks.  
  - *Connections:* Outputs to Merge File Selection Data.  
  - *Failure types:* Same as above.

- **List Video Background Files**  
  - *Type:* Google Drive  
  - *Role:* Lists actual video background files in specified Drive folder.  
  - *Connections:* Outputs to Retrieve Video Background Data.  
  - *Failure types:* Folder access errors.

- **List Music Background Files**  
  - *Type:* Google Drive  
  - *Role:* Lists music files.  
  - *Connections:* Outputs to Retrieve Music Background Data.  
  - *Failure types:* Same as above.

---

#### 2.2 Random Content Selection

**Overview:**  
Merges data from quotes, video backgrounds, and music, then randomly picks one item from each category to prepare for video generation.

**Nodes Involved:**  
- Merge File Selection Data (Merge)  
- Select Random Video, Music & Quote (Code)

**Node Details:**  

- **Merge File Selection Data**  
  - *Type:* Merge  
  - *Role:* Combines outputs from the three data retrieval nodes into a single dataset for random selection.  
  - *Connections:* Inputs from Retrieve Quote Data, Retrieve Video Background Data, Retrieve Music Background Data; outputs to Select Random Video, Music & Quote.  
  - *Failure types:* Data mismatch or empty inputs cause errors or empty merges.

- **Select Random Video, Music & Quote**  
  - *Type:* Code  
  - *Role:* Executes JavaScript to select one random video background file, one music track, and one quote from the merged data.  
  - *Key Logic:* Uses random index selection on arrays; prepares payload for video generation.  
  - *Connections:* Outputs to upload and gen video (video generation HTTP request).  
  - *Failure types:* Empty arrays, coding errors.

---

#### 2.3 Video Generation & Processing

**Overview:**  
Sends selected assets to an external video generation API, polls for completion, downloads the finished video.

**Nodes Involved:**  
- upload and gen video (HTTP Request)  
- check status (HTTP Request)  
- If (Conditional)  
- download video (HTTP Request)  
- Save Final Video (Read/Write File)  
- Wait (Wait)  
- If1 (Conditional)  
- Send a text message (Telegram)

**Node Details:**  

- **upload and gen video**  
  - *Type:* HTTP Request  
  - *Role:* Sends request to an external service (likely FFmpeg API) to generate the video combining selected assets.  
  - *Connections:* Output to check status.  
  - *Failure types:* Network errors, API auth failures, invalid payloads.

- **check status**  
  - *Type:* HTTP Request  
  - *Role:* Polls the external service for video generation completion status.  
  - *Connections:* Output to If1 conditional.  
  - *Failure types:* Timeout, 404 if job not found.

- **If1**  
  - *Type:* If  
  - *Role:* Determines if video generation is complete or still processing.  
  - *Connections:* True branch to Send a text message (notification), False branch to Wait node.  
  - *Failure types:* Expression errors.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses workflow to avoid aggressive polling.  
  - *Connections:* Output back to If node to re-check status.  
  - *Failure types:* None typical.

- **If**  
  - *Type:* If  
  - *Role:* Checks if the video file is ready to download.  
  - *Connections:* True branch to download video.  
  - *Failure types:* Expression errors.

- **download video**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the finished video file.  
  - *Connections:* Outputs to Save Final Video.  
  - *Failure types:* Network errors, file not found.

- **Save Final Video**  
  - *Type:* Read/Write File  
  - *Role:* Saves downloaded video to local storage.  
  - *Connections:* Outputs to Download File from Google Drive.  
  - *Failure types:* Disk write errors.

- **Send a text message**  
  - *Type:* Telegram  
  - *Role:* Sends notification messages on status updates.  
  - *Connections:* None downstream.  
  - *Failure types:* Telegram API errors, auth issues.

---

#### 2.4 File Handling & Upload

**Overview:**  
Uploads the saved video file to Google Drive for storage and further sharing.

**Nodes Involved:**  
- Read output file (Read/Write File)  
- Upload file (Google Drive)  
- Download File from Google Drive (HTTP Request)  
- Extract Filename Without Extension (Set)

**Node Details:**  

- **Read output file**  
  - *Type:* Read/Write File  
  - *Role:* Reads the saved video file from local disk to prepare for upload.  
  - *Connections:* Outputs to Upload file.  
  - *Failure types:* File read errors.

- **Upload file**  
  - *Type:* Google Drive  
  - *Role:* Uploads video file to configured Google Drive folder.  
  - *Connections:* Outputs to Download File from Google Drive.  
  - *Failure types:* Drive quota or permissions errors.

- **Download File from Google Drive**  
  - *Type:* HTTP Request  
  - *Role:* Downloads uploaded file metadata or link; used for further processing.  
  - *Connections:* Outputs to Extract Filename Without Extension.  
  - *Failure types:* API errors.

- **Extract Filename Without Extension**  
  - *Type:* Set  
  - *Role:* Parses filename to remove extension, likely for metadata or naming consistency.  
  - *Connections:* Outputs to Read output file.  
  - *Failure types:* String manipulation errors.

---

#### 2.5 Publishing & Status Tracking

**Overview:**  
Uploads the final video to YouTube, manages playlist addition, and updates status sheets. Disabled nodes show intended support for Facebook, Instagram, and X.

**Nodes Involved:**  
- Upload to YouTube1 (YouTube)  
- Move to playlist1 (YouTube)  
- Update Instagram status2 (Google Sheets)  
- Send a text message2 (Telegram)  
- Upload Video to Instagram (HTTP Request) [disabled]  
- Update Instagram status (Google Sheets) [disabled]  
- Upload Video to X (HTTP Request) [disabled]  
- Update X status (Google Sheets) [disabled]  
- Upload Video to FB1 (HTTP Request) [disabled]  
- Upload Video to FB (HTTP Request) [disabled]  
- Update FB status (Google Sheets) [disabled]

**Node Details:**  

- **Upload to YouTube1**  
  - *Type:* YouTube node  
  - *Role:* Uploads the video to YouTube with provided metadata, credentials via OAuth2.  
  - *Connections:* Outputs to Move to playlist1.  
  - *Failure types:* OAuth token expiry, API limits.

- **Move to playlist1**  
  - *Type:* YouTube node  
  - *Role:* Adds the uploaded video to a specified YouTube playlist.  
  - *Connections:* Outputs to Update Instagram status2.  
  - *Failure types:* API errors.

- **Update Instagram status2**  
  - *Type:* Google Sheets  
  - *Role:* Updates the Google Sheet tracking Instagram upload status.  
  - *Connections:* Outputs to Send a text message2 for notification.  
  - *Failure types:* Auth or rate limit errors.

- **Send a text message2**  
  - *Type:* Telegram  
  - *Role:* Notifies about the upload status.  
  - *Connections:* Terminal node.  
  - *Failure types:* Telegram API issues.

- **Disabled Social Upload Nodes**  
  - *Purpose:* Intended for posting videos to Instagram, Facebook, and X using HTTP APIs with tokens generated at upload-post.com.  
  - *Notes:* Currently disabled, with instructions to add API keys in headers.  
  - *Failure types:* Auth errors, API changes.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                          | Input Node(s)                              | Output Node(s)                            | Sticky Note                                                                                                 |
|-------------------------------|---------------------|----------------------------------------|--------------------------------------------|------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Start AutoClip Workflow        | Manual Trigger      | Manual workflow start                   |                                            | Retrieve Quote Data, List Video Background Files, Configure Music Background Folder ID |                                                                                                             |
| Schedule Trigger              | Schedule Trigger    | Scheduled workflow start                |                                            | Configure Music Background Folder ID, List Video Background Files, Retrieve Quote Data |                                                                                                             |
| Configure Music Background Folder ID | Set               | Define music folder ID                  | Start AutoClip Workflow, Schedule Trigger  | List Music Background Files               |                                                                                                             |
| List Video Background Files    | Google Drive        | List video background files             | Start AutoClip Workflow, Schedule Trigger  | Retrieve Video Background Data            |                                                                                                             |
| List Music Background Files    | Google Drive        | List music files                        | Configure Music Background Folder ID        | Retrieve Music Background Data             |                                                                                                             |
| Retrieve Quote Data            | Google Sheets       | Retrieve quotes                        | Start AutoClip Workflow, Schedule Trigger  | Merge File Selection Data                  |                                                                                                             |
| Retrieve Video Background Data | Google Sheets       | Retrieve video background metadata     | List Video Background Files                  | Merge File Selection Data                  |                                                                                                             |
| Retrieve Music Background Data | Google Sheets       | Retrieve music metadata                | List Music Background Files                   | Merge File Selection Data                  |                                                                                                             |
| Merge File Selection Data      | Merge               | Combine retrieved data                  | Retrieve Quote Data, Retrieve Video Background Data, Retrieve Music Background Data | Select Random Video, Music & Quote        |                                                                                                             |
| Select Random Video, Music & Quote | Code               | Randomly select assets                  | Merge File Selection Data                     | upload and gen video                       |                                                                                                             |
| upload and gen video           | HTTP Request        | Trigger video generation API            | Select Random Video, Music & Quote            | check status                              |                                                                                                             |
| check status                  | HTTP Request        | Poll video generation status            | upload and gen video                          | If1                                      |                                                                                                             |
| If1                          | If                  | Check if video is ready                  | check status                                  | Send a text message (true), Wait (false) |                                                                                                             |
| Send a text message           | Telegram            | Send notification on status             | If1 (true branch)                             | None                                     |                                                                                                             |
| Wait                         | Wait                | Wait before rechecking status           | If1 (false branch)                            | If                                       |                                                                                                             |
| If                           | If                  | Check if video should be downloaded     | Wait                                          | download video (true)                     |                                                                                                             |
| download video               | HTTP Request        | Download generated video                 | If (true branch)                              | Save Final Video                          |                                                                                                             |
| Save Final Video             | Read/Write File     | Save video locally                       | download video                                | Download File from Google Drive           |                                                                                                             |
| Download File from Google Drive | HTTP Request        | Get uploaded file metadata or download  | Save Final Video                              | Extract Filename Without Extension        |                                                                                                             |
| Extract Filename Without Extension | Set                 | Parse filename                          | Download File from Google Drive                | Read output file                          |                                                                                                             |
| Read output file             | Read/Write File     | Read final video for upload              | Extract Filename Without Extension             | Upload file                              |                                                                                                             |
| Upload file                 | Google Drive        | Upload final video to Drive              | Read output file                              | Download File from Google Drive           |                                                                                                             |
| Upload to YouTube1           | YouTube             | Upload video to YouTube                  | Download File from Google Drive                | Move to playlist1                        |                                                                                                             |
| Move to playlist1            | YouTube             | Add uploaded video to playlist           | Upload to YouTube1                            | Update Instagram status2                   |                                                                                                             |
| Update Instagram status2     | Google Sheets       | Update Instagram upload status           | Move to playlist1                            | Send a text message2                      |                                                                                                             |
| Send a text message2         | Telegram            | Notify upload completion                  | Update Instagram status2                      | None                                     |                                                                                                             |
| Upload Video to Instagram    | HTTP Request        | Upload video to Instagram (disabled)     | Download File from Google Drive                | Update Instagram status                   | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here) |
| Update Instagram status      | Google Sheets       | Update Instagram status (disabled)       | Upload Video to Instagram                      | None                                     |                                                                                                             |
| Upload Video to X            | HTTP Request        | Upload video to X (disabled)              | Download File from Google Drive                | Update X status                          | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here) |
| Update X status              | Google Sheets       | Update X upload status (disabled)         | Upload Video to X                              | None                                     |                                                                                                             |
| Upload Video to FB           | HTTP Request        | Upload video to Facebook (disabled)       | Download File from Google Drive                | Update FB status                        | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here) |
| Update FB status             | Google Sheets       | Update Facebook status (disabled)          | Upload Video to FB                            | None                                     |                                                                                                             |
| Upload Video to FB1          | HTTP Request        | Upload video to Facebook (disabled)       | Download File from Google Drive                | None                                     | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual & Scheduled Triggers**  
   - Add a **Manual Trigger** node named `Start AutoClip Workflow`.  
   - Add a **Schedule Trigger** node named `Schedule Trigger` with desired cron timing.

2. **Configure Music Folder ID**  
   - Add a **Set** node `Configure Music Background Folder ID` with a parameter to store the Google Drive folder ID for music files.

3. **List Video and Music Files**  
   - Add a **Google Drive** node `List Video Background Files` configured to list files in the video background folder.  
   - Add a **Google Drive** node `List Music Background Files` configured to list files in the music folder (uses output from `Configure Music Background Folder ID`).

4. **Retrieve Metadata from Google Sheets**  
   - Add three **Google Sheets** nodes:  
     - `Retrieve Quote Data` for quotes sheet and range.  
     - `Retrieve Video Background Data` for video metadata.  
     - `Retrieve Music Background Data` for music metadata.

5. **Merge Data**  
   - Add a **Merge** node `Merge File Selection Data` that merges outputs from the three Google Sheets nodes.

6. **Select Random Items**  
   - Add a **Code** node `Select Random Video, Music & Quote` with JavaScript to randomly select one row from each data set and prepare payload for video generation.

7. **Trigger Video Generation**  
   - Add an **HTTP Request** node `upload and gen video` configured to send the selected assets to the external video generation API.

8. **Poll Generation Status**  
   - Add an **HTTP Request** node `check status` to poll the API for completion.  
   - Add an **If** node `If1` to check if the video is ready.  
   - On false, add a **Wait** node to pause, then loop back to `check status`.  
   - On true, send notification via **Telegram** node `Send a text message`.

9. **Download Final Video**  
   - Add an **If** node `If` to check if download is possible.  
   - Add an **HTTP Request** node `download video` to download the video.  
   - Add a **Read/Write File** node `Save Final Video` to save the video locally.

10. **Upload Final Video to Google Drive**  
    - Add an **HTTP Request** node `Download File from Google Drive` to get metadata or prepare for upload.  
    - Add a **Set** node `Extract Filename Without Extension` to parse filename.  
    - Add a **Read/Write File** node `Read output file` to read the saved video.  
    - Add a **Google Drive** node `Upload file` to upload the video.

11. **Publish to YouTube**  
    - Add a **YouTube** node `Upload to YouTube1` configured with OAuth2 credentials for YouTube upload.  
    - Add a **YouTube** node `Move to playlist1` to add the video to a playlist.

12. **Update Status and Notify**  
    - Add a **Google Sheets** node `Update Instagram status2` to update the status sheet.  
    - Add a **Telegram** node `Send a text message2` to notify completion.

13. **Optional Disabled Social Upload Nodes**  
    - Add HTTP Request nodes with headers containing API keys for Instagram, X, and Facebook video uploads, currently disabled and configured as placeholders.

14. **Connect all nodes** as per the workflow logic described in section 2.

15. **Credential Setup:**  
    - Configure Google Drive and Google Sheets credentials with appropriate access.  
    - Configure YouTube OAuth2 credentials.  
    - Configure Telegram bot credentials.  
    - For HTTP request nodes targeting external services, configure API keys or tokens as per service requirements.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                 |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| To upload videos to Instagram, X, and Facebook, generate tokens via https://upload-post.com and add them to HTTP request headers under Authorization: Apikey (token here). | Inline notes on Upload Video to Instagram, X, FB nodes |
| This workflow uses FFmpeg processing via an external API (not included in n8n) for video generation. Ensure the API endpoint and payload match the service’s expectations. | Video Generation API integration                 |
| Telegram nodes are used for notifications and require a bot token configured in credentials.                 | Telegram Bot API documentation                    |
| YouTube uploading requires OAuth2 credentials with appropriate scopes for video upload and playlist management. | YouTube Data API v3                              |

---

_Disclaimer: The provided description and analysis are based solely on the exported n8n workflow JSON and respect all applicable content policies. No illegal or protected content is involved._