Automatically Create and Upload YouTube Videos with Quotes in Thai Using FFmpeg

https://n8nworkflows.xyz/workflows/automatically-create-and-upload-youtube-videos-with-quotes-in-thai-using-ffmpeg-3328


# Automatically Create and Upload YouTube Videos with Quotes in Thai Using FFmpeg

### 1. Workflow Overview

This workflow automates the creation and upload of short-form YouTube videos featuring inspirational quotes in Thai, combining multimedia assets and text overlays with FFmpeg on a self-hosted n8n instance. It targets digital content creators, marketers, and social media managers who want to efficiently produce multilingual, visually appealing video clips with minimal manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Data Preparation & File Selection:**  
  Retrieves quotes, background videos, and music files from Google Sheets and Google Drive, then randomly selects one item from each category.

- **1.2 File Download & Video Processing:**  
  Downloads the selected video and music files locally, prepares the overlay text (quote and author) with appropriate fonts, and generates the final video clip using FFmpeg.

- **1.3 Video Upload & Post-Processing:**  
  Uploads the generated video to YouTube via the YouTube API and updates the Google Sheet with the upload status and YouTube URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Preparation & File Selection

**Overview:**  
This block fetches source data for quotes, video backgrounds, and music from Google Sheets and Google Drive, merges them, and randomly selects one quote, one background video, and one music file for video creation.

**Nodes Involved:**  
- Start AutoClip Workflow  
- Retrieve Quote Data  
- List Video Background Files  
- Retrieve Video Background Data  
- Configure Music Background Folder ID  
- List Music Background Files  
- Retrieve Music Background Data  
- Merge File Selection Data  
- Select Random Video, Music & Quote  
- Sticky Note (Data Preparation & File Selection)

**Node Details:**

- **Start AutoClip Workflow**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Triggers downstream nodes.  
  - Edge Cases: None.

- **Retrieve Quote Data**  
  - Type: Google Sheets  
  - Role: Reads quotes and authors from a Google Sheet, filtering by `CreateStatus` to select only pending quotes.  
  - Configuration: Reads from sheet with `gid=0` (Quotes_status), uses OAuth2 credentials.  
  - Key Expressions: Filters on `CreateStatus` column to exclude already processed quotes.  
  - Inputs: Trigger from Start node.  
  - Outputs: Quote data JSON objects.  
  - Edge Cases: Authentication errors, empty result sets if all quotes are processed.

- **List Video Background Files**  
  - Type: Google Drive  
  - Role: Lists video background files from a specific Google Drive folder (folderId: `1mLOqFJZvUm563mJ7LvTsIcKrAoakX-h2`).  
  - Configuration: Retrieves file metadata including `webViewLink`, `id`, and `name`.  
  - Inputs: Trigger from Start node.  
  - Outputs: List of video background files.  
  - Edge Cases: Folder access permission errors, empty folder.

- **Retrieve Video Background Data**  
  - Type: Google Sheets  
  - Role: Appends or updates video background URLs and statuses in a Google Sheet (sheetId: `90817124`).  
  - Configuration: Maps `BackgroundURL` and `BackgroudStatus` columns, uses OAuth2 credentials.  
  - Inputs: From List Video Background Files node.  
  - Outputs: Updated video background data.  
  - Edge Cases: Sheet permission issues, data mapping errors.

- **Configure Music Background Folder ID**  
  - Type: Set  
  - Role: Stores the Google Drive folder ID for music backgrounds as a workflow variable (`MusicBackgroundFolderID`).  
  - Configuration: Sets string value `12T6ABEuR7WlZ2i88GqcB3U4DmuKVM4iR`.  
  - Inputs: Trigger from Start node.  
  - Outputs: Passes folder ID downstream.  
  - Edge Cases: None.

- **List Music Background Files**  
  - Type: Google Drive  
  - Role: Lists music files from the configured music folder in Google Drive.  
  - Configuration: Uses folder ID from previous Set node, retrieves file metadata.  
  - Inputs: From Configure Music Background Folder ID node.  
  - Outputs: List of music files.  
  - Edge Cases: Folder access permission errors, empty folder.

- **Retrieve Music Background Data**  
  - Type: Google Sheets  
  - Role: Appends or updates music background URLs and statuses in a Google Sheet (sheetId: `1264732774`).  
  - Configuration: Maps `MusicURL` and `MusicStatus` columns, uses OAuth2 credentials.  
  - Inputs: From List Music Background Files node.  
  - Outputs: Updated music background data.  
  - Edge Cases: Sheet permission issues, data mapping errors.

- **Merge File Selection Data**  
  - Type: Merge  
  - Role: Combines the outputs of quote data, video background data, and music background data into a single data stream.  
  - Configuration: Set to accept 3 inputs.  
  - Inputs: From Retrieve Quote Data, Retrieve Video Background Data, Retrieve Music Background Data nodes.  
  - Outputs: Merged data array.  
  - Edge Cases: Mismatched input lengths, empty inputs.

- **Select Random Video, Music & Quote**  
  - Type: Code (JavaScript)  
  - Role: Randomly selects one video background, one music file, and one quote from the merged data.  
  - Configuration: Custom JS code filters inputs by presence of keys (`BackgroundURL`, `MusicURL`, `Qoute`), throws error if any category is empty.  
  - Key Expressions: Uses Math.random to select random items.  
  - Inputs: From Merge File Selection Data node.  
  - Outputs: Single combined JSON object with selected video, music, and quote.  
  - Edge Cases: Empty input arrays, expression errors, missing keys.

- **Sticky Note (Data Preparation & File Selection)**  
  - Content: Explains this block’s purpose to retrieve and merge source data and randomly select media and quote.  
  - Position: Visual aid only.

---

#### 2.2 File Download & Video Processing

**Overview:**  
Downloads the selected video and music files locally, prepares the overlay text with Thai font support, and generates the final video clip using FFmpeg with specified video dimensions and audio volume.

**Nodes Involved:**  
- Download Selected Video Background  
- Save Video Background Locally  
- Download Selected Music Background  
- Save Music Background Locally  
- Prepare Overlay Text (Quote & Author)  
- Generate Final Video Clip  
- Sticky Note1 (File Download & Video Processing)

**Node Details:**

- **Download Selected Video Background**  
  - Type: Google Drive (Download)  
  - Role: Downloads the selected video background file using its Google Drive URL.  
  - Configuration: Uses `BackgroundURL` from selected video JSON, downloads binary data as `data`.  
  - Inputs: From Select Random Video, Music & Quote node.  
  - Outputs: Binary video file data.  
  - Edge Cases: File not found, permission denied, download timeout.

- **Save Video Background Locally**  
  - Type: Read/Write File  
  - Role: Writes the downloaded video binary data to local disk as `video1.mp4`.  
  - Configuration: Write operation, filename fixed as `video1.mp4`.  
  - Inputs: From Download Selected Video Background node.  
  - Outputs: File path and metadata.  
  - Edge Cases: Disk write permission errors, insufficient space.

- **Download Selected Music Background**  
  - Type: Google Drive (Download)  
  - Role: Downloads the selected music file using its Google Drive URL.  
  - Configuration: Uses `MusicURL` from selected music JSON, downloads binary data as `data`.  
  - Inputs: From Save Video Background Locally node (sequential flow).  
  - Outputs: Binary music file data.  
  - Edge Cases: File not found, permission denied, download timeout.

- **Save Music Background Locally**  
  - Type: Read/Write File  
  - Role: Writes the downloaded music binary data to local disk as `music1.mp3`.  
  - Configuration: Write operation, filename fixed as `music1.mp3`, no append.  
  - Inputs: From Download Selected Music Background node.  
  - Outputs: File path and metadata.  
  - Edge Cases: Disk write permission errors, insufficient space.

- **Prepare Overlay Text (Quote & Author)**  
  - Type: Code (JavaScript)  
  - Role: Generates FFmpeg `drawtext` filter commands to overlay the quote and author text on the video using the Thai font "Kanit-Italic.ttf".  
  - Configuration:  
    - Quote font size: 70  
    - Author font size: 50  
    - Font color: white  
    - Video width: 1080px  
    - Margin: 40px  
    - Text is centered horizontally and vertically, author aligned right below quote block  
  - Key Expressions: Splits quote into lines based on estimated max chars per line, escapes single quotes for FFmpeg, constructs multiple `drawtext` filters concatenated by commas.  
  - Inputs: Uses merged data from "Merge File Selection Data" node for quote and author text.  
  - Outputs: JSON with `drawText` string for FFmpeg filter.  
  - Edge Cases: Missing quote or author throws error, font file must exist on FFmpeg environment, text encoding issues.

- **Generate Final Video Clip**  
  - Type: Execute Command  
  - Role: Runs FFmpeg command to create the final video with scaled and cropped video, semi-transparent black overlay, text overlay, and mixed audio.  
  - Configuration:  
    - Input video: `video1.mp4`  
    - Input audio: `music1.mp3`  
    - Video scaled to 1080x1920 (9:16 aspect ratio) with cropping  
    - Black overlay at 30% opacity for 10 seconds  
    - Text overlay from `drawText` filter string  
    - Audio volume set to 0.8  
    - Output file: `output.mp4` (overwrites if exists)  
    - Video codec: libx264, audio codec: aac  
  - Inputs: From Prepare Overlay Text node (for filter string) and saved media files.  
  - Outputs: Final video file `output.mp4`.  
  - Edge Cases: FFmpeg command errors, missing fonts, file permission errors, long processing times.

- **Sticky Note1 (File Download & Video Processing)**  
  - Content: Describes the block’s role in downloading files, preparing overlay text, and generating the video clip.  
  - Position: Visual aid only.

---

#### 2.3 Video Upload & Post-Processing

**Overview:**  
Uploads the generated video to YouTube using the YouTube API with resumable upload, then updates the Google Sheet with the video’s YouTube URL and marks the background video as used.

**Nodes Involved:**  
- Initiate YouTube Resumable Upload  
- Read output file  
- Upload Video to YouTube  
- Update Quote Upload Status  
- Mark Background as Used  
- Sticky Note2 (Video Upload & Post-Processing)

**Node Details:**

- **Initiate YouTube Resumable Upload**  
  - Type: HTTP Request  
  - Role: Starts a resumable upload session with YouTube API, sending video metadata (title, description, language, privacy, etc.).  
  - Configuration:  
    - URL: `https://www.googleapis.com/upload/youtube/v3/videos?part=snippet,status&uploadType=resumable`  
    - Method: POST  
    - Body: JSON with snippet (title and description from quote), status (public, license, embeddable, etc.)  
    - Headers: Content-Type `application/json`, `X-Upload-Content-Type` set to `video/webm`  
    - Authentication: OAuth2 with YouTube credentials  
  - Inputs: From Generate Final Video Clip node (uses quote text for metadata)  
  - Outputs: Response with upload URL in headers for next step  
  - Edge Cases: Authentication failure, API quota limits, malformed metadata.

- **Read output file**  
  - Type: Read/Write File  
  - Role: Reads the generated video file `output.mp4` into binary data for upload.  
  - Configuration: File selector set to `output.mp4`.  
  - Inputs: From Initiate YouTube Resumable Upload node.  
  - Outputs: Binary video data.  
  - Edge Cases: File not found, read permission errors.

- **Upload Video to YouTube**  
  - Type: HTTP Request  
  - Role: Uploads the video binary data to the YouTube resumable upload URL obtained previously.  
  - Configuration:  
    - URL: Dynamic, from `Initiate YouTube Resumable Upload` response header `location`  
    - Method: PUT  
    - Content-Type: `video/webm`  
    - Authentication: OAuth2 with YouTube credentials  
    - Sends binary data field named `data`  
  - Inputs: From Read output file node.  
  - Outputs: YouTube API response including video ID.  
  - Edge Cases: Upload interruptions, network errors, API limits.

- **Update Quote Upload Status**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet row for the quote with `CreateStatus` set to `DONE` and inserts the YouTube URL.  
  - Configuration:  
    - Matches row by `Index` column from quote data  
    - Updates `YoutubeURL` with constructed YouTube watch URL using video ID from upload response  
    - Uses OAuth2 credentials  
  - Inputs: From Upload Video to YouTube node.  
  - Outputs: Confirmation of sheet update.  
  - Edge Cases: Sheet permission errors, mismatched index, update failures.

- **Mark Background as Used**  
  - Type: Google Sheets  
  - Role: Marks the used video background as `DONE` in the Google Sheet to avoid reuse.  
  - Configuration:  
    - Matches row by `BackgroundURL`  
    - Updates `BackgroudStatus` to `DONE`  
    - Uses OAuth2 credentials  
  - Inputs: From Update Quote Upload Status node.  
  - Outputs: Confirmation of sheet update.  
  - Edge Cases: Sheet permission errors, mismatched URL, update failures.

- **Sticky Note2 (Video Upload & Post-Processing)**  
  - Content: Explains this block uploads the video to YouTube and updates Google Sheets with status and URLs.  
  - Position: Visual aid only.

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                               | Input Node(s)                           | Output Node(s)                          | Sticky Note                                                                                     |
|--------------------------------|----------------------|----------------------------------------------|---------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Start AutoClip Workflow         | Manual Trigger       | Workflow entry point                          | -                                     | Retrieve Quote Data, List Video Background Files, Configure Music Background Folder ID |                                                                                                |
| Retrieve Quote Data             | Google Sheets        | Fetch quotes with pending status              | Start AutoClip Workflow                | Merge File Selection Data              |                                                                                                |
| List Video Background Files     | Google Drive         | List video background files                   | Start AutoClip Workflow                | Retrieve Video Background Data         |                                                                                                |
| Retrieve Video Background Data  | Google Sheets        | Append/update video background data           | List Video Background Files            | Merge File Selection Data              |                                                                                                |
| Configure Music Background Folder ID | Set              | Set folder ID for music files                  | Start AutoClip Workflow                | List Music Background Files            |                                                                                                |
| List Music Background Files     | Google Drive         | List music background files                    | Configure Music Background Folder ID  | Retrieve Music Background Data         |                                                                                                |
| Retrieve Music Background Data  | Google Sheets        | Append/update music background data            | List Music Background Files            | Merge File Selection Data              |                                                                                                |
| Merge File Selection Data       | Merge                | Combine quotes, video, and music data          | Retrieve Quote Data, Retrieve Video Background Data, Retrieve Music Background Data | Select Random Video, Music & Quote    |                                                                                                |
| Select Random Video, Music & Quote | Code (JS)          | Randomly select one video, music, and quote    | Merge File Selection Data              | Download Selected Video Background     |                                                                                                |
| Download Selected Video Background | Google Drive (Download) | Download selected video file                  | Select Random Video, Music & Quote    | Save Video Background Locally          |                                                                                                |
| Save Video Background Locally   | Read/Write File      | Save video file locally                         | Download Selected Video Background    | Download Selected Music Background     |                                                                                                |
| Download Selected Music Background | Google Drive (Download) | Download selected music file                  | Save Video Background Locally          | Save Music Background Locally          |                                                                                                |
| Save Music Background Locally   | Read/Write File      | Save music file locally                         | Download Selected Music Background    | Prepare Overlay Text (Quote & Author) |                                                                                                |
| Prepare Overlay Text (Quote & Author) | Code (JS)          | Generate FFmpeg drawtext filter for overlay   | Save Music Background Locally          | Generate Final Video Clip              |                                                                                                |
| Generate Final Video Clip       | Execute Command      | Create final video with overlay and audio      | Prepare Overlay Text                   | Initiate YouTube Resumable Upload      |                                                                                                |
| Initiate YouTube Resumable Upload | HTTP Request       | Start YouTube video upload session             | Generate Final Video Clip              | Read output file                      |                                                                                                |
| Read output file               | Read/Write File      | Read generated video file for upload           | Initiate YouTube Resumable Upload     | Upload Video to YouTube                |                                                                                                |
| Upload Video to YouTube         | HTTP Request         | Upload video binary to YouTube                  | Read output file                      | Update Quote Upload Status             |                                                                                                |
| Update Quote Upload Status      | Google Sheets        | Update quote row with YouTube URL and status   | Upload Video to YouTube                | Mark Background as Used                |                                                                                                |
| Mark Background as Used         | Google Sheets        | Mark video background as used                   | Update Quote Upload Status             | -                                     |                                                                                                |
| Sticky Note                    | Sticky Note          | Data Preparation & File Selection explanation  | -                                     | -                                     | ## Data Preparation & File Selection\nRetrieve and merge source data for quotes, video backgrounds, and music from Google Sheets and Google Drive; then randomly select one quote, one background video, and one music file. |
| Sticky Note1                   | Sticky Note          | File Download & Video Processing explanation    | -                                     | -                                     | ## File Download & Video Processing\nDownload the selected files, write them to disk, prepare overlay text (quote and author), and generate the final video clip using FFmpeg. |
| Sticky Note2                   | Sticky Note          | Video Upload & Post-Processing explanation      | -                                     | -                                     | ## Video Upload & Post-Processing\nUpload the final video to YouTube using the YouTube API and update your Google Sheets with upload statuses and YouTube links. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually.

2. **Add Google Sheets Node to Retrieve Quote Data**  
   - Operation: Read rows from Google Sheet (Quotes_status tab, gid=0)  
   - Filter: Only rows where `CreateStatus` is blank or pending  
   - Credentials: Google Sheets OAuth2  
   - Output: Quote data JSON.

3. **Add Google Drive Node to List Video Background Files**  
   - Operation: List files in folder with ID `1mLOqFJZvUm563mJ7LvTsIcKrAoakX-h2`  
   - Fields: `webViewLink`, `id`, `name`  
   - Credentials: Google Drive OAuth2  
   - Output: Video background file metadata.

4. **Add Google Sheets Node to Retrieve Video Background Data**  
   - Operation: Append or update rows in Google Sheet (video_backgroud_list tab, gid=90817124)  
   - Columns: `BackgroundURL`, `BackgroudStatus`  
   - Credentials: Google Sheets OAuth2.

5. **Add Set Node to Configure Music Background Folder ID**  
   - Set variable `MusicBackgroundFolderID` to `12T6ABEuR7WlZ2i88GqcB3U4DmuKVM4iR`.

6. **Add Google Drive Node to List Music Background Files**  
   - Operation: List files in folder with ID from `MusicBackgroundFolderID`  
   - Fields: `webViewLink`, `name`, `id`  
   - Credentials: Google Drive OAuth2.

7. **Add Google Sheets Node to Retrieve Music Background Data**  
   - Operation: Append or update rows in Google Sheet (music_backgroud_list tab, gid=1264732774)  
   - Columns: `MusicURL`, `MusicStatus`  
   - Credentials: Google Sheets OAuth2.

8. **Add Merge Node**  
   - Number of Inputs: 3  
   - Connect: Retrieve Quote Data, Retrieve Video Background Data, Retrieve Music Background Data.

9. **Add Code Node to Select Random Video, Music & Quote**  
   - JavaScript code to filter and randomly select one item from each input array.  
   - Throws error if any input array is empty.

10. **Add Google Drive Node to Download Selected Video Background**  
    - Operation: Download file by URL from selected video’s `BackgroundURL`.  
    - Output binary data property: `data`.

11. **Add Read/Write File Node to Save Video Background Locally**  
    - Operation: Write binary data to `video1.mp4`.

12. **Add Google Drive Node to Download Selected Music Background**  
    - Operation: Download file by URL from selected music’s `MusicURL`.  
    - Output binary data property: `data`.

13. **Add Read/Write File Node to Save Music Background Locally**  
    - Operation: Write binary data to `music1.mp3`.

14. **Add Code Node to Prepare Overlay Text (Quote & Author)**  
    - JavaScript code to:  
      - Split quote text into lines based on estimated max characters per line.  
      - Construct FFmpeg `drawtext` filter commands for each line and author.  
      - Use Thai font `Kanit-Italic.ttf` with font sizes 70 (quote) and 50 (author).  
      - Return combined drawtext filter string.

15. **Add Execute Command Node to Generate Final Video Clip**  
    - Command:  
      ```
      ffmpeg -i video1.mp4 -i music1.mp3 -filter_complex "[0:v]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920[vid]; color=black@0.3:size=1080x1920:d=10[bg]; [vid][bg]overlay=shortest=1[bgvid]; [bgvid]{{ $json.drawText }}[outv]; [1:a]volume=0.8[aout]" -map "[outv]" -map "[aout]" -aspect 9:16 -c:v libx264 -c:a aac -shortest output.mp4 -y
      ```  
    - Replace `{{ $json.drawText }}` with output from previous node.

16. **Add HTTP Request Node to Initiate YouTube Resumable Upload**  
    - Method: POST  
    - URL: `https://www.googleapis.com/upload/youtube/v3/videos?part=snippet,status&uploadType=resumable`  
    - Body (raw JSON):  
      ```json
      {
        "snippet": {
          "title": "{{quote text}}",
          "description": "{{quote text}}\n{{author}}",
          "defaultLanguage": "en",
          "defaultAudioLanguage": "en"
        },
        "status": {
          "privacyStatus": "public",
          "license": "youtube",
          "embeddable": true,
          "publicStatsViewable": true,
          "madeForKids": false
        }
      }
      ```  
    - Headers: Content-Type `application/json`, `X-Upload-Content-Type` `video/webm`  
    - Authentication: YouTube OAuth2 credentials.

17. **Add Read/Write File Node to Read Output File**  
    - Operation: Read `output.mp4` as binary data.

18. **Add HTTP Request Node to Upload Video to YouTube**  
    - Method: PUT  
    - URL: Use `location` header from previous HTTP response (upload URL)  
    - Content-Type: `video/webm`  
    - Send binary data from previous node.  
    - Authentication: YouTube OAuth2 credentials.

19. **Add Google Sheets Node to Update Quote Upload Status**  
    - Operation: Append or update row in Quotes_status sheet (gid=0)  
    - Match by `Index` from quote data  
    - Update `CreateStatus` to `DONE` and `YoutubeURL` to `https://www.youtube.com/watch?v={{videoId}}`.

20. **Add Google Sheets Node to Mark Background as Used**  
    - Operation: Append or update row in video_backgroud_list sheet (gid=90817124)  
    - Match by `BackgroundURL`  
    - Update `BackgroudStatus` to `DONE`.

21. **Connect all nodes according to the logical flow described in the overview and ensure credentials are configured for Google Sheets, Google Drive, and YouTube OAuth2.**

22. **Ensure FFmpeg is installed on the self-hosted n8n environment and that the font file `Kanit-Italic.ttf` is accessible to FFmpeg.**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow requires a self-hosted n8n instance because FFmpeg command execution is not supported on n8n Cloud. | Setup requirement                                                                                            |
| Google Sheet template for quotes and statuses is available here: https://docs.google.com/spreadsheets/d/184-zcrfWSzQpDa-t57Oo_8DLyAF-2B_6yvGrybrcd5I/edit?usp=sharing | Template for quote and media management                                                                      |
| Use fonts that fully support your target language (e.g., Kanit for Thai) and ensure they are installed in FFmpeg. | Font customization and language support                                                                     |
| Video output is optimized for 9:16 aspect ratio, suitable for YouTube Shorts and vertical video platforms.       | Video format specification                                                                                   |
| Adjust font sizes, text positioning, and audio volume in the FFmpeg command and overlay text preparation node.   | Customization tips                                                                                           |
| The workflow updates Google Sheets with statuses and YouTube URLs to maintain an up-to-date record of content.   | Data management best practice                                                                                 |

---

This structured documentation enables users and AI agents to fully understand, reproduce, and customize the workflow, anticipating potential errors and integration points.