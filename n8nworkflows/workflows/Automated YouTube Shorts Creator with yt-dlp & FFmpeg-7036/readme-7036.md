Automated YouTube Shorts Creator with yt-dlp & FFmpeg

https://n8nworkflows.xyz/workflows/automated-youtube-shorts-creator-with-yt-dlp---ffmpeg-7036


# Automated YouTube Shorts Creator with yt-dlp & FFmpeg

### 1. Workflow Overview

This workflow automates the creation and uploading of YouTube Shorts by downloading video and music content from YouTube, processing them into short clips with text overlays, and uploading the final videos back to YouTube. It leverages Google Sheets as a data source and tracking system, using yt-dlp and FFmpeg for media processing, and integrates with YouTube API for uploads. The workflow consists of two main pipelines — one for videos and one for music — and a final composition and upload sequence.

Logical blocks:

- **1.1 Video Download Pipeline:** Fetch YouTube video URLs from Google Sheets, download videos with unique filenames, and update the sheets with local paths and status.
- **1.2 Music Download Pipeline:** Fetch YouTube music URLs from another Google Sheets tab, download audio as MP3 with embedded thumbnails, and update the sheets similarly.
- **1.3 Video & Music Aggregation and Selection:** Aggregate all successfully downloaded video and music files, randomly select one video and one music track for composition.
- **1.4 Text Overlay Generation:** Fetch video transcript quotes, generate multi-line text overlay parameters for FFmpeg.
- **1.5 Video Composition:** Use FFmpeg to combine the selected video and music, apply text overlays, scale and crop video to 9:16 aspect ratio, and generate a 10-second short clip.
- **1.6 YouTube Upload and Post-Processing:** Upload the generated video via YouTube API, update logs in Google Sheets with upload info, and clean up local files.
- **1.7 Triggers:** Manual and scheduled triggers to initiate download or upload workflows.

---

### 2. Block-by-Block Analysis

#### 2.1 Video Download Pipeline

- **Overview:** This block retrieves video URLs from a Google Sheet ("Video Pool"), limits to one record, generates a unique filename, downloads the video using yt-dlp, and updates the sheet with download status and local file path.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - GetYTvideo  
  - Limit_1  
  - GenrateFileName  
  - DownloadVideoFootage  
  - UpdateVideoFootageDetails

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution  
    - Configuration: Default manual trigger, no parameters  
    - Inputs: None  
    - Outputs: Triggers GetYTvideo  
    - Edge cases: None (user-initiated)

  - **GetYTvideo**  
    - Type: Google Sheets (Read)  
    - Role: Fetch video records with filter on "Status" column (likely to fetch only those needing download)  
    - Configuration: Reads from "Video Pool" sheet, uses OAuth2 Google Sheets credentials  
    - Inputs: Trigger from manual node  
    - Outputs: Passes filtered rows to Limit_1  
    - Edge cases: Google Sheets API failures, OAuth token expiration, empty results

  - **Limit_1**  
    - Type: Limit  
    - Role: Restrict processing to one record at a time  
    - Configuration: Default limit to 1 item  
    - Inputs: Output of GetYTvideo  
    - Outputs: Forward to GenrateFileName  
    - Edge cases: No records available

  - **GenrateFileName**  
    - Type: Set  
    - Role: Generate unique random filename and folder name ("Video Footage") for saving video  
    - Configuration: Creates "File Name" as random 10-character string; sets "Folder Name" to "Video Footage"  
    - Inputs: From Limit_1  
    - Outputs: Passes filename info to DownloadVideoFootage  
    - Expressions: `={{ Array.from({ length: 10 }, () => Math.random().toString(36)[2]).join('') }}`  
    - Edge cases: Random collision unlikely but possible

  - **DownloadVideoFootage**  
    - Type: Execute Command  
    - Role: Run yt-dlp to download YouTube video at best available MP4 quality, save to dynamically generated path  
    - Configuration: Command uses output path `./<Folder Name>/<File Name>.mp4` and input URL from GetYTvideo's "Video Path"  
    - Inputs: From GenrateFileName  
    - Outputs: Passes to UpdateVideoFootageDetails  
    - Edge cases: yt-dlp failures (network issues, invalid URLs), file write permission errors

  - **UpdateVideoFootageDetails**  
    - Type: Google Sheets (Update)  
    - Role: Update the row in "Video Pool" sheet with Status="Success" and local file path  
    - Configuration: Matches row by "row_number" from Limit_1, updates "Status" and "Internal Path" columns  
    - Inputs: From DownloadVideoFootage  
    - Outputs: End of pipeline branch  
    - Edge cases: Google Sheets update failures, row mismatch

#### 2.2 Music Download Pipeline

- **Overview:** Similar to video pipeline but for music. Fetches music links from "Music Pool" sheet, downloads as high-quality MP3 with embedded thumbnails using yt-dlp, updates sheet with status and path.

- **Nodes Involved:**  
  - Schedule Trigger (periodic)  
  - GetYTmusic  
  - Limit_01  
  - GenrateFileName_01  
  - DownloadMusic  
  - UpdateMusicDetails

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Periodically trigger music download workflow  
    - Configuration: Default interval (unspecified)  
    - Outputs: Starts GetYTmusic

  - **GetYTmusic**  
    - Type: Google Sheets (Read)  
    - Role: Fetch music entries filtered by "Status" (pending download) from "Music Pool" sheet  
    - Inputs: From Schedule Trigger  
    - Outputs: To Limit_01  
    - Edge cases: Same as GetYTvideo

  - **Limit_01**  
    - Type: Limit  
    - Role: Process one music record at a time  
    - Inputs: From GetYTmusic  
    - Outputs: To GenrateFileName_01

  - **GenrateFileName_01**  
    - Type: Set  
    - Role: Generate unique filename and folder name ("MusicFile") for music file  
    - Configuration: Similar random 10-character string for "File Name", folder "MusicFile"  
    - Outputs: To DownloadMusic

  - **DownloadMusic**  
    - Type: Execute Command  
    - Role: Use yt-dlp to extract audio as MP3 with best quality and embedded thumbnail  
    - Command example: `yt-dlp -x --audio-format mp3 --audio-quality 0 --embed-thumbnail -o "./MusicFile/<File Name>.mp3" <Music URL>`  
    - Inputs: From GenrateFileName_01  
    - Outputs: To UpdateMusicDetails

  - **UpdateMusicDetails**  
    - Type: Google Sheets (Update)  
    - Role: Update music pool sheet row with Status="Success" and internal path  
    - Inputs: From DownloadMusic  
    - Outputs: End of music download branch

#### 2.3 Video & Music Aggregation and Selection

- **Overview:** Collect all successfully downloaded videos and music tracks, aggregate their internal paths, and randomly select one of each for composition.

- **Nodes Involved:**  
  - Schedule Trigger1 (scheduled)  
  - GetAllVideos  
  - AggregateVideos  
  - GetAllMusic  
  - AggregateMusic  
  - GetQuotes  
  - Limit  
  - PickMusicVideoRandom

- **Node Details:**

  - **Schedule Trigger1**  
    - Type: Schedule Trigger  
    - Role: Trigger aggregation and processing at a fixed time (hour:11, minute:51)  
    - Outputs: GetAllVideos

  - **GetAllVideos**  
    - Type: Google Sheets (Read)  
    - Role: Fetch all video records with Status="Success"  
    - Inputs: From Schedule Trigger1  
    - Outputs: AggregateVideos

  - **AggregateVideos**  
    - Type: Aggregate  
    - Role: Aggregate "Internal Path" field values into an array for videos  
    - Inputs: From GetAllVideos  
    - Outputs: GetAllMusic

  - **GetAllMusic**  
    - Type: Google Sheets (Read)  
    - Role: Fetch all music records with Status="Success"  
    - Inputs: From AggregateVideos  
    - Outputs: AggregateMusic

  - **AggregateMusic**  
    - Type: Aggregate  
    - Role: Aggregate "Internal Path" field values for music files  
    - Inputs: From GetAllMusic  
    - Outputs: GetQuotes

  - **GetQuotes**  
    - Type: Google Sheets (Read)  
    - Role: Fetch video transcript quotes from "Video Logs" sheet filtered by "Status" (likely pending processing)  
    - Inputs: From AggregateMusic  
    - Outputs: Limit

  - **Limit**  
    - Type: Limit  
    - Role: Process one quote record at a time  
    - Inputs: From GetQuotes  
    - Outputs: PickMusicVideoRandom

  - **PickMusicVideoRandom**  
    - Type: Set  
    - Role: Randomly select one video path and one music path from aggregated arrays  
    - Configuration: Uses expressions accessing aggregated arrays and selects random index e.g.,  
      - `Music_Path = {{ aggregatedMusicPaths[Math.floor(Math.random() * aggregatedMusicPaths.length)] }}`  
      - `Video_Path = {{ aggregatedVideoPaths[Math.floor(Math.random() * aggregatedVideoPaths.length)] }}`  
    - Inputs: From Limit  
    - Outputs: GenrateFinalFileName

#### 2.4 Text Overlay Generation

- **Overview:** Generate FFmpeg drawtext filter parameters to overlay multi-line text (video transcript) centered on the video.

- **Nodes Involved:**  
  - GenrateFinalFileName  
  - GenrateTextOverlayForVideo

- **Node Details:**

  - **GenrateFinalFileName**  
    - Type: Set  
    - Role: Generate unique random filename for final composed video  
    - Configuration: Random 10-character string as "File Name"  
    - Inputs: From PickMusicVideoRandom  
    - Outputs: GenrateTextOverlayForVideo

  - **GenrateTextOverlayForVideo**  
    - Type: Code (JavaScript)  
    - Role: Process video transcript text into multiple lines that fit video width, generate FFmpeg drawtext commands with font, size, color, and vertical positioning  
    - Inputs: From GenrateFinalFileName; uses "Video Transcript" from GetQuotes node  
    - Outputs: JSON object with `drawText` string containing FFmpeg filter commands  
    - Edge cases: Empty or very long transcripts, special characters in text (escaping), missing font file

#### 2.5 Video Composition

- **Overview:** Use FFmpeg command to merge selected video and audio, scale and crop video to 1080x1920 (9:16), apply text overlays, adjust audio volume, generate 10-second short clip.

- **Nodes Involved:**  
  - GenerateVideo

- **Node Details:**

  - **GenerateVideo**  
    - Type: Execute Command  
    - Role: Run FFmpeg with complex filter: scale video to 1080x1920, overlay background color, apply drawtext overlays, mix audio with volume 0.8, output MP4 short  
    - Inputs: From GenrateTextOverlayForVideo  
    - Outputs: CreateYoutubeLink  
    - Command uses expressions for video and music paths, drawText filter string, output file path  
    - Edge cases: FFmpeg errors, missing input files, incorrect filter syntax

#### 2.6 YouTube Upload and Post-Processing

- **Overview:** Upload the generated video file to YouTube via resumable upload API, update Google Sheet logs with video info and YouTube link, then delete local video file.

- **Nodes Involved:**  
  - CreateYoutubeLink  
  - Read/Write Files from Disk  
  - UploadVideo  
  - UpdateVideoPost  
  - RemoveFile

- **Node Details:**

  - **CreateYoutubeLink**  
    - Type: HTTP Request  
    - Role: Initiate YouTube resumable upload session, sending video metadata (title, description, language, privacy, license)  
    - Inputs: From GenerateVideo  
    - Outputs: Read/Write Files from Disk  
    - Requires: OAuth2 credentials with YouTube scope  
    - Edge cases: API quota exceeded, invalid credentials

  - **Read/Write Files from Disk**  
    - Type: Read/Write File  
    - Role: Read the generated MP4 video file from disk to prepare for upload  
    - Inputs: From CreateYoutubeLink  
    - Outputs: UploadVideo

  - **UploadVideo**  
    - Type: HTTP Request  
    - Role: PUT upload of binary video data to YouTube resumable upload URL (from CreateYoutubeLink response header)  
    - Inputs: From Read/Write Files from Disk  
    - Outputs: UpdateVideoPost  
    - Headers: Content-Type set to "video/webm" (possible mismatch with MP4 input; may cause upload issues)  
    - Edge cases: Upload failures, network interruptions, content-type mismatch

  - **UpdateVideoPost**  
    - Type: Google Sheets (Update)  
    - Role: Update "Video Logs" sheet with upload status, output path, YouTube URL, and post time  
    - Inputs: From UploadVideo  
    - Outputs: RemoveFile

  - **RemoveFile**  
    - Type: Execute Command  
    - Role: Delete the local generated MP4 file after successful upload to save disk space  
    - Inputs: From UpdateVideoPost  
    - Outputs: End of workflow branch

#### 2.7 Triggers and Notes

- **Schedule Trigger:** Periodic start of Music Download pipeline  
- **Schedule Trigger1:** Scheduled start of aggregation and upload pipeline at fixed time  
- **Manual Trigger:** To manually start video download pipeline  
- **Sticky Notes:** Document high-level purpose and block summaries

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                            | Input Node(s)                   | Output Node(s)                       | Sticky Note                                                                                           |
|-----------------------------|-----------------------|------------------------------------------|--------------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Manual entry point                        | None                           | GetYTvideo                         |                                                                                                     |
| GetYTvideo                  | Google Sheets         | Fetch video URLs pending download        | When clicking ‘Execute workflow’| Limit_1                           |                                                                                                     |
| Limit_1                    | Limit                 | Limit to 1 video record                   | GetYTvideo                     | GenrateFileName                    |                                                                                                     |
| GenrateFileName             | Set                   | Generate unique filename for video       | Limit_1                        | DownloadVideoFootage               |                                                                                                     |
| DownloadVideoFootage        | Execute Command       | Download video using yt-dlp               | GenrateFileName                | UpdateVideoFootageDetails          |                                                                                                     |
| UpdateVideoFootageDetails   | Google Sheets         | Update video sheet with status and path | DownloadVideoFootage           | None                             |                                                                                                     |
| Schedule Trigger            | Schedule Trigger      | Periodic trigger for music download      | None                          | GetYTmusic                       |                                                                                                     |
| GetYTmusic                 | Google Sheets         | Fetch music URLs pending download         | Schedule Trigger               | Limit_01                         |                                                                                                     |
| Limit_01                   | Limit                 | Limit to 1 music record                   | GetYTmusic                    | GenrateFileName_01               |                                                                                                     |
| GenrateFileName_01          | Set                   | Generate unique filename for music file  | Limit_01                      | DownloadMusic                   |                                                                                                     |
| DownloadMusic              | Execute Command       | Download music as MP3 with thumbnail      | GenrateFileName_01            | UpdateMusicDetails               |                                                                                                     |
| UpdateMusicDetails          | Google Sheets         | Update music sheet with status and path  | DownloadMusic                 | None                           |                                                                                                     |
| Schedule Trigger1           | Schedule Trigger      | Scheduled trigger for aggregation/upload | None                          | GetAllVideos                   |                                                                                                     |
| GetAllVideos               | Google Sheets         | Fetch all successful videos               | Schedule Trigger1             | AggregateVideos                |                                                                                                     |
| AggregateVideos            | Aggregate             | Aggregate video internal paths            | GetAllVideos                 | GetAllMusic                   |                                                                                                     |
| GetAllMusic                | Google Sheets         | Fetch all successful music files          | AggregateVideos              | AggregateMusic                |                                                                                                     |
| AggregateMusic             | Aggregate             | Aggregate music internal paths             | GetAllMusic                  | GetQuotes                    |                                                                                                     |
| GetQuotes                  | Google Sheets         | Fetch video transcript quotes              | AggregateMusic               | Limit                      |                                                                                                     |
| Limit                      | Limit                 | Limit to 1 quote record                    | GetQuotes                   | PickMusicVideoRandom         |                                                                                                     |
| PickMusicVideoRandom       | Set                   | Randomly select one video and music path | Limit                       | GenrateFinalFileName         |                                                                                                     |
| GenrateFinalFileName        | Set                   | Generate unique filename for final video | PickMusicVideoRandom         | GenrateTextOverlayForVideo   |                                                                                                     |
| GenrateTextOverlayForVideo | Code                  | Generate FFmpeg text overlay parameters   | GenrateFinalFileName          | GenerateVideo                |                                                                                                     |
| GenerateVideo              | Execute Command       | Compose video with audio and overlays     | GenrateTextOverlayForVideo    | CreateYoutubeLink            |                                                                                                     |
| CreateYoutubeLink          | HTTP Request          | Initiate YouTube resumable upload session | GenerateVideo                | Read/Write Files from Disk   |                                                                                                     |
| Read/Write Files from Disk | Read/Write File       | Read video file from disk for upload      | CreateYoutubeLink            | UploadVideo                 |                                                                                                     |
| UploadVideo                | HTTP Request          | Upload video binary data to YouTube       | Read/Write Files from Disk   | UpdateVideoPost             |                                                                                                     |
| UpdateVideoPost            | Google Sheets         | Update logs with upload info                | UploadVideo                 | RemoveFile                  |                                                                                                     |
| RemoveFile                 | Execute Command       | Delete local video file after upload       | UpdateVideoPost             | None                       |                                                                                                     |
| Sticky Note                | Sticky Note           | Workflow overview and notes                | None                        | None                       | # YouTube Video & Music Downloader: video & music download pipelines summarized                     |
| Sticky Note1               | Sticky Note           | Pipeline overview and tracking notes       | None                        | None                       | # YouTube Video & Music Processor: describes video pipeline, music pipeline, and tracking          |
| Sticky Note2               | Sticky Note           | High-level workflow description            | None                        | None                       | # Automatic Youtube shorts creation and uploading: mentions two workflows (download & merge/upload) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Name: `When clicking ‘Execute workflow’`
   - Type: Manual Trigger
   - No parameters

2. **Create Google Sheets Node to Get Videos:**
   - Name: `GetYTvideo`
   - Operation: Read rows
   - Document ID: Google Sheet with "Video Pool"
   - Sheet name: "Video Pool"
   - Filter: Column "Status" is empty or specific to pending downloads
   - Credentials: Google Sheets OAuth2

3. **Create Limit Node:**
   - Name: `Limit_1`
   - Limit: 1 item

4. **Create Set Node to Generate Video Filename:**
   - Name: `GenrateFileName`
   - Add fields:  
     - `File Name`: expression to generate random 10-character string  
       `={{ Array.from({ length: 10 }, () => Math.random().toString(36)[2]).join('') }}`  
     - `Folder Name`: `"Video Footage"`

5. **Create Execute Command Node to Download Video:**
   - Name: `DownloadVideoFootage`
   - Command:  
     ```
     yt-dlp -o "./{{$json["Folder Name"]}}/{{$json["File Name"]}}.mp4" "{{ $json["Video Path"] }}" -f "bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]/best"
     ```
   - Note: Use inputs from previous node's JSON fields

6. **Create Google Sheets Node to Update Video Status:**
   - Name: `UpdateVideoFootageDetails`
   - Operation: Update row
   - Document ID & Sheet: same as GetYTvideo
   - Match by: `row_number`
   - Update columns:  
     - `Status`: "Success"  
     - `Internal Path`: `=./{{$json["Folder Name"]}}/{{$json["File Name"]}}.mp4`

7. **Connect nodes as:**  
   `When clicking ‘Execute workflow’` → `GetYTvideo` → `Limit_1` → `GenrateFileName` → `DownloadVideoFootage` → `UpdateVideoFootageDetails`

8. **Create Schedule Trigger Node for Music Downloads:**
   - Name: `Schedule Trigger`
   - Set interval as needed

9. **Create Google Sheets Node to Get Music:**
   - Name: `GetYTmusic`
   - Document ID: same Google Sheet, sheet name: "Music Pool"
   - Filter: "Status" pending
   - Credentials: Google Sheets OAuth2

10. **Create Limit Node:**
    - Name: `Limit_01`
    - Limit: 1

11. **Create Set Node for Music Filename:**
    - Name: `GenrateFileName_01`
    - Fields:  
      - `File Name`: random 10-char string as above  
      - `Folder Name`: `"MusicFile"`

12. **Create Execute Command Node to Download Music:**
    - Name: `DownloadMusic`
    - Command:  
      ```
      yt-dlp -x --audio-format mp3 --audio-quality 0 --embed-thumbnail -o "./{{$json["Folder Name"]}}/{{$json["File Name"]}}.mp3" "{{ $json["Music Path"] }}"
      ```

13. **Create Google Sheets Node to Update Music Status:**
    - Name: `UpdateMusicDetails`
    - Update "Music Pool" sheet, match row_number, set Status and Internal Path

14. **Connect nodes as:**  
    `Schedule Trigger` → `GetYTmusic` → `Limit_01` → `GenrateFileName_01` → `DownloadMusic` → `UpdateMusicDetails`

15. **Create Schedule Trigger1 for Aggregation and Upload:**
    - Name: `Schedule Trigger1`
    - Set to daily at 11:51 (or desired time)

16. **Create Google Sheets Node to Get All Videos (Status=Success):**
    - Name: `GetAllVideos`
    - Filter: Status = "Success"
    - Sheet: "Video Pool"

17. **Create Aggregate Node to collect Video Paths:**
    - Name: `AggregateVideos`
    - Aggregate field: Internal Path

18. **Create Google Sheets Node to Get All Music (Status=Success):**
    - Name: `GetAllMusic`
    - Filter: Status = "Success"
    - Sheet: "Music Pool"

19. **Create Aggregate Node for Music Paths:**
    - Name: `AggregateMusic`
    - Aggregate field: Internal Path

20. **Create Google Sheets Node to Get Quotes:**
    - Name: `GetQuotes`
    - Sheet: "Video Logs"
    - Filter: Status pending

21. **Create Limit Node:**
    - Name: `Limit`
    - Limit: 1

22. **Create Set Node to Pick Random Video and Music:**
    - Name: `PickMusicVideoRandom`
    - Fields:  
      - `Music_Path` = randomly select element from `AggregateMusic.Internal Path` array  
      - `Video_Path` = randomly select element from `AggregateVideos.Internal Path` array  
    - Use JavaScript expressions to pick random index

23. **Create Set Node to Generate Final Filename:**
    - Name: `GenrateFinalFileName`
    - Generate random 10-char string as before

24. **Create Code Node to Generate Text Overlay:**
    - Name: `GenrateTextOverlayForVideo`
    - JavaScript code to split transcript into lines fitting 1080px width, generate FFmpeg drawtext filter string

25. **Create Execute Command Node to Generate Final Video:**
    - Name: `GenerateVideo`
    - Command uses FFmpeg to:  
      - Input selected video and music paths  
      - Scale video to 1080x1920 with crop  
      - Overlay semi-transparent black background  
      - Apply drawtext overlay from previous node  
      - Adjust audio volume to 0.8  
      - Output 10s MP4 short clip with generated filename

26. **Create HTTP Request Node to Initiate YouTube Upload:**
    - Name: `CreateYoutubeLink`
    - POST to `https://www.googleapis.com/upload/youtube/v3/videos?part=snippet,status&uploadType=resumable`  
    - Body JSON includes video metadata: title, description, language, privacy, license  
    - Use YouTube OAuth2 credentials

27. **Create Read/Write File Node to Read Video:**
    - Name: `Read/Write Files from Disk`
    - File path: generated final video path

28. **Create HTTP Request Node to Upload Video Data:**
    - Name: `UploadVideo`
    - PUT request to URL from `CreateYoutubeLink` response header `location`  
    - Content-Type: "video/webm" (note: input file is MP4; consider matching content-type)  
    - Send binary data from previous node  
    - Credentials: YouTube OAuth2

29. **Create Google Sheets Node to Update Upload Log:**
    - Name: `UpdateVideoPost`
    - Update "Video Logs" sheet row with:  
      - Status: "Success"  
      - Post Time: current timestamp  
      - Output Path: local file path  
      - YouTube Link: constructed from returned video ID

30. **Create Execute Command Node to Remove File:**
    - Name: `RemoveFile`
    - Command: `rm ./<finalFileName>.mp4`

31. **Connect final nodes as:**  
    `Schedule Trigger1` → `GetAllVideos` → `AggregateVideos` → `GetAllMusic` → `AggregateMusic` → `GetQuotes` → `Limit` → `PickMusicVideoRandom` → `GenrateFinalFileName` → `GenrateTextOverlayForVideo` → `GenerateVideo` → `CreateYoutubeLink` → `Read/Write Files from Disk` → `UploadVideo` → `UpdateVideoPost` → `RemoveFile`

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates YouTube Shorts creation: downloads videos & music, merges with text overlays, uploads video | High-level summary of the entire workflow                                                      |
| yt-dlp used for downloading videos and extracting audio                                                        | https://github.com/yt-dlp/yt-dlp                                                              |
| FFmpeg used for video scaling, cropping, overlay, and audio mixing                                             | https://ffmpeg.org/                                                                            |
| YouTube API resumable upload requires OAuth2 credentials with proper scopes                                    | https://developers.google.com/youtube/v3/guides/using_resumable_uploads                         |
| Google Sheets used for tracking video/music URLs, status, and logs                                            | Requires Google Sheets OAuth2 credentials configured in n8n                                     |
| Caution: UploadVideo node uses Content-Type "video/webm" but input file is MP4; may cause upload failures      | Consider verifying and matching content-type for YouTube upload                                |
| Font file "NotoSerif_Condensed-BlackItalic.ttf" must be available in FFmpeg working directory                  | Required for correct text overlay rendering                                                    |
| Random file name generation is simple; consider collision handling for large scale usage                       | Currently uses random 10-character alphanumeric strings                                        |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.