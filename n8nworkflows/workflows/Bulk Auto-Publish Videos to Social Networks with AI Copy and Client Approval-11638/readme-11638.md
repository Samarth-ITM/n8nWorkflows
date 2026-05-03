Bulk Auto-Publish Videos to Social Networks with AI Copy and Client Approval

https://n8nworkflows.xyz/workflows/bulk-auto-publish-videos-to-social-networks-with-ai-copy-and-client-approval-11638


# Bulk Auto-Publish Videos to Social Networks with AI Copy and Client Approval

### 1. Workflow Overview

This n8n workflow automates the bulk processing, AI-based content generation, client approval, and scheduling of videos to multiple social media platforms (TikTok, Instagram Reels, and YouTube Shorts). It is designed for marketers or content managers who want to streamline video publishing with AI-generated social copy and a manual approval step before auto-publishing.

The workflow logically divides into two main flows:

- **Flow 1: Campaign Intake, AI Processing, and Draft Saving**
  - Input reception via a form trigger that collects campaign settings.
  - Fetching videos from a specified Google Drive folder.
  - Filtering for video files and building a posting schedule calendar based on user-defined cadence.
  - Downloading each video and performing AI video content analysis using Google Gemini.
  - Generating platform-specific titles, descriptions, and hashtags with AI.
  - Formatting and saving the results as draft rows in a Google Sheet for client approval.

- **Flow 2: Approval Monitoring and Auto-Publish**
  - Runs every 15 minutes to check the Google Sheet for videos marked as approved.
  - Downloads approved videos and schedules them for publishing on enabled platforms via Upload-Post.
  - Updates the Google Sheet status to "scheduled" after successful scheduling.

---

### 2. Block-by-Block Analysis

#### 2.1 Flow 1: Campaign Intake & Drive Fetch

**Overview:**  
This block collects campaign parameters via a form, reads all video files from a Google Drive folder, filters out non-video files, and builds a posting calendar based on the user-selected cadence and start date.

**Nodes Involved:**  
- Start Campaign (Form Trigger)  
- Campaign Settings (Set)  
- Fetch Videos from Drive (Google Drive)  
- Video Files Only (Filter)  
- Build Schedule Calendar (Code)  

**Node Details:**

- **Start Campaign**  
  - Type: Form Trigger  
  - Role: Collects user inputs for campaign settings (Drive Folder ID, Upload-Post profile username, platforms, timezone, start date, cadence, publish hour, Google Sheet ID).  
  - Key parameters: Form fields with validation for required inputs.  
  - Outputs data to Campaign Settings node.  
  - Edge cases: Invalid folder IDs or missing required fields will prevent workflow continuation.

- **Campaign Settings**  
  - Type: Set  
  - Role: Assigns form input values to workflow variables for downstream use.  
  - Configuration: Maps form response fields to variables such as drive_folder_id, profile_username, platforms_raw, etc.  
  - Inputs: From Start Campaign node.  
  - Outputs: To Fetch Videos from Drive node.

- **Fetch Videos from Drive**  
  - Type: Google Drive (List Files)  
  - Role: Queries Google Drive for files inside the specified folder ID, excluding trashed items.  
  - Configuration: Uses a query string `'{{ $json.drive_folder_id }}' in parents and trashed = false`.  
  - Credentials: Requires OAuth2 Google Drive connection.  
  - Edge cases: API errors, folder access permissions, empty folder.

- **Video Files Only**  
  - Type: Filter  
  - Role: Filters the fetched files to keep only those whose names contain ".mp4" (video files).  
  - Configuration: Condition checks if filename contains ".mp4".  
  - Edge cases: Misnamed or unsupported video formats are excluded.

- **Build Schedule Calendar**  
  - Type: Code (JavaScript)  
  - Role: Calculates scheduled publishing dates for each video based on start date, cadence (daily, 5 per week, 3 per week), and publish hour.  
  - Logic:  
    - Maps cadence options to day offsets, skipping weekends if necessary.  
    - Builds ISO datetime strings for scheduling.  
  - Outputs enriched items with schedule info and campaign config.  
  - Edge cases: Invalid date formats, empty input list, time zone considerations.

---

#### 2.2 Flow 1: AI Analysis & Copy Generation

**Overview:**  
Processes each scheduled video by downloading it, performing a detailed AI analysis (content description, transcription, keywords, etc.), then generating platform-specific social media copy using AI. The output is structured JSON content for TikTok, Instagram, and YouTube.

**Nodes Involved:**  
- Process Each Video (Split in Batches)  
- Download from Drive (Google Drive)  
- AI Video Analysis (Google Gemini Video Analysis)  
- Generate Social Copy (AI Agent)  
- Gemini Pro (Google Gemini Chat LM)  
- Structured Output Parser (AI Output Parser)  

**Node Details:**

- **Process Each Video**  
  - Type: SplitInBatches  
  - Role: Processes videos one by one to avoid API rate limits and memory issues.  
  - Edge cases: Batch processing failures, large batch sizes.

- **Download from Drive**  
  - Type: Google Drive (Download)  
  - Role: Downloads each video file as binary data for AI analysis.  
  - Credentials: Google Drive OAuth2.  
  - Edge cases: Download failures, permission errors, large file sizes.

- **AI Video Analysis**  
  - Type: Google Gemini Video Analysis (Langchain node)  
  - Role: Uses Gemini Flash model to analyze video content and generate a detailed textual analysis (description, transcription, keywords, etc.).  
  - Configuration: Text prompt instructs output format starting with detected language line.  
  - Credentials: Google Palm API key.  
  - Edge cases: API timeout, malformed responses, quota limits.

- **Generate Social Copy**  
  - Type: AI Agent (Langchain)  
  - Role: Generates unique platform-specific titles, descriptions, and hashtags for TikTok, Instagram, and YouTube based on the video analysis.  
  - Configuration: Prompt enforces language consistency based on detected language and requests strict JSON output.  
  - Edge cases: Invalid JSON output, API failures.

- **Gemini Pro**  
  - Type: LM Chat Google Gemini  
  - Role: Supports the Generate Social Copy node as an alternative or reinforcement LM.  
  - Credentials: Google Palm API key.

- **Structured Output Parser**  
  - Type: AI Output Parser Structured  
  - Role: Validates and cleans up the AI-generated JSON content, auto-fixing minor errors.  
  - Configuration: Uses a JSON schema example to enforce structure.  
  - Edge cases: Parsing errors, incomplete data.

---

#### 2.3 Flow 1: Save Draft to Google Sheet

**Overview:**  
Formats the AI-generated social copy along with schedule and campaign info into rows, then appends them as draft entries in a Google Sheet. These drafts await manual approval for publishing.

**Nodes Involved:**  
- Format Content Data (Code)  
- Save Draft to Sheet (Google Sheets Append)  

**Node Details:**

- **Format Content Data**  
  - Type: Code (JavaScript)  
  - Role:  
    - Parses the AI output JSON safely, handling parse errors.  
    - Constructs a row object with campaign metadata and social copy fields (titles, descriptions, hashtags/tags) per platform.  
    - Sets initial status to "draft".  
  - Edge cases: JSON parse failures result in rows marked "error".

- **Save Draft to Sheet**  
  - Type: Google Sheets (Append)  
  - Role: Appends formatted draft rows into the "VideosTable" sheet of the specified Google Sheet.  
  - Configuration: Uses auto-mapping for columns based on the formatted row structure.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: API errors, sheet permissions, column mismatches.

---

#### 2.4 Flow 2: Approval & Auto-Publish

**Overview:**  
Periodically (every 15 minutes) checks the Google Sheet for videos marked as "approved", downloads each approved video, and schedules posts on enabled platforms via Upload-Post. After scheduling, updates the status to "scheduled".

**Nodes Involved:**  
- 15 mins check (Schedule Trigger)  
- Publisher Config (Set)  
- Load Content Queue (Google Sheets Read)  
- Only Approved (Filter)  
- Process Queue (Split in Batches)  
- Get Video File (Google Drive Download)  
- Build Upload Payload (Code)  
- TikTok Enabled? (If)  
- Instagram Enabled? (If)  
- YouTube Enabled? (If)  
- Schedule TikTok (Upload-Post)  
- Schedule Instagram (Upload-Post)  
- Schedule YouTube (Upload-Post)  
- Mark as Scheduled (Google Sheets Update)  

**Node Details:**

- **15 mins check**  
  - Type: Schedule Trigger  
  - Role: Triggers this flow every 15 minutes to scan for approved videos.  
  - Edge cases: Timing conflicts, missed runs.

- **Publisher Config**  
  - Type: Set  
  - Role: Sets fixed configuration values for sheet ID, Drive folder ID, and profile username for publishing.  
  - Hardcoded values may need manual update for each use case.

- **Load Content Queue**  
  - Type: Google Sheets (Read)  
  - Role: Reads all rows from "VideosTable" sheet.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Large data sets, API rate limits.

- **Only Approved**  
  - Type: Filter  
  - Role: Filters rows to only those with Status = "approved".  
  - Edge cases: Empty results if no approved videos.

- **Process Queue**  
  - Type: SplitInBatches  
  - Role: Processes approved videos in batches to control load.

- **Get Video File**  
  - Type: Google Drive (Download)  
  - Role: Downloads the approved video file binary.  
  - Credentials: Google Drive OAuth2.

- **Build Upload Payload**  
  - Type: Code (JavaScript)  
  - Role: Constructs the payload for Upload-Post API, including platform flags, titles, descriptions, hashtags, schedule datetime, and profile username.  
  - Inputs: Row data and downloaded binary video.

- **TikTok Enabled?**  
  - Type: If  
  - Role: Checks if TikTok is included in the platforms for this video.  
  - Routes to Schedule TikTok if true.

- **Instagram Enabled?**  
  - Type: If  
  - Role: Checks if Instagram is enabled.  
  - Routes accordingly.

- **YouTube Enabled?**  
  - Type: If  
  - Role: Checks if YouTube is enabled.  
  - Routes accordingly.

- **Schedule TikTok / Instagram / YouTube**  
  - Type: Upload-Post (uploadVideo operation)  
  - Role: Uploads and schedules the video on the respective platform, using configured profile and scheduled datetime.  
  - Credentials: Upload-Post API key.  
  - Parameters: Titles, descriptions, hashtags, media type (Instagram Reels), etc.  
  - Edge cases: API errors, scheduling failures, invalid credentials.

- **Mark as Scheduled**  
  - Type: Google Sheets (Update)  
  - Role: Updates the status of the row to "scheduled" once the upload is successful.  
  - Uses Video ID as key for matching.  
  - Edge cases: Update conflicts, API errors.

---

### 3. Summary Table

| Node Name              | Node Type                            | Functional Role                          | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                   |
|------------------------|------------------------------------|----------------------------------------|----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Start Campaign         | Form Trigger                       | Collect campaign input from user       |                            | Campaign Settings            | Batch process and schedule your videos across TikTok, Instagram & YouTube with AI-powered optimization |
| Campaign Settings      | Set                               | Assign form inputs to workflow variables | Start Campaign             | Fetch Videos from Drive      |                                                                                               |
| Fetch Videos from Drive | Google Drive                      | List files in specified Drive folder   | Campaign Settings           | Video Files Only             | Flow 1: Campaign intake & Drive fetch                                                        |
| Video Files Only       | Filter                           | Filter to only .mp4 video files         | Fetch Videos from Drive      | Build Schedule Calendar      | Flow 1: Campaign intake & Drive fetch                                                        |
| Build Schedule Calendar | Code                             | Compute posting schedule per cadence    | Video Files Only            | Process Each Video           | Flow 1: Campaign intake & Drive fetch                                                        |
| Process Each Video     | SplitInBatches                   | Process videos one at a time             | Build Schedule Calendar      | Download from Drive          | Flow 1: AI analysis & copy                                                                   |
| Download from Drive    | Google Drive                    | Download video file binary               | Process Each Video          | AI Video Analysis            | Flow 1: AI analysis & copy                                                                   |
| AI Video Analysis      | Google Gemini Video Analysis     | Analyze video content with AI            | Download from Drive         | Generate Social Copy         | Flow 1: AI analysis & copy                                                                   |
| Generate Social Copy   | AI Agent (Langchain)             | Generate platform-specific social copy  | AI Video Analysis           | Structured Output Parser     | Flow 1: AI analysis & copy                                                                   |
| Gemini Pro             | LM Chat Google Gemini            | Language model assistance for copy gen  | Generate Social Copy        | Structured Output Parser     | Flow 1: AI analysis & copy                                                                   |
| Structured Output Parser| AI Output Parser                | Validate and clean AI JSON output        | Generate Social Copy        | Format Content Data          | Flow 1: AI analysis & copy                                                                   |
| Format Content Data    | Code                            | Format AI output and metadata to rows   | Structured Output Parser    | Save Draft to Sheet          | Flow 1: Save drafts                                                                          |
| Save Draft to Sheet    | Google Sheets (Append)           | Append draft rows to Google Sheet        | Format Content Data         | Process Each Video (loop)    | Flow 1: Save drafts                                                                          |
| 15 mins check          | Schedule Trigger                | Trigger Flow 2 every 15 minutes          |                            | Publisher Config             | Flow 2: Approval & auto-publish                                                             |
| Publisher Config       | Set                             | Set hardcoded config for publishing      | 15 mins check              | Load Content Queue           | Flow 2: Approval & auto-publish                                                             |
| Load Content Queue     | Google Sheets (Read)             | Read VideosTable sheet rows               | Publisher Config           | Only Approved                | Flow 2: Approval & auto-publish                                                             |
| Only Approved          | Filter                         | Filter rows where Status=approved         | Load Content Queue         | Process Queue                | Flow 2: Approval & auto-publish                                                             |
| Process Queue          | SplitInBatches                 | Process approved videos in batches        | Only Approved              | Get Video File / End         | Flow 2: Approval & auto-publish                                                             |
| Get Video File         | Google Drive (Download)          | Download approved video file              | Process Queue              | Build Upload Payload         | Flow 2: Approval & auto-publish                                                             |
| Build Upload Payload   | Code                            | Prepare upload payload with video & copy | Get Video File             | TikTok Enabled?, Instagram Enabled?, YouTube Enabled? | Flow 2: Approval & auto-publish                                                             |
| TikTok Enabled?        | If                              | Check if TikTok platform is enabled       | Build Upload Payload       | Schedule TikTok / End        | Flow 2: Approval & auto-publish                                                             |
| Instagram Enabled?     | If                              | Check if Instagram platform is enabled    | Build Upload Payload       | Schedule Instagram / End     | Flow 2: Approval & auto-publish                                                             |
| YouTube Enabled?       | If                              | Check if YouTube platform is enabled      | Build Upload Payload       | Schedule YouTube / End       | Flow 2: Approval & auto-publish                                                             |
| Schedule TikTok        | Upload-Post (uploadVideo)        | Schedule video upload on TikTok           | TikTok Enabled?            | Mark as Scheduled           | Flow 2: Approval & auto-publish                                                             |
| Schedule Instagram     | Upload-Post (uploadVideo)        | Schedule video upload on Instagram Reels | Instagram Enabled?         | Mark as Scheduled           | Flow 2: Approval & auto-publish                                                             |
| Schedule YouTube       | Upload-Post (uploadVideo)        | Schedule video upload on YouTube Shorts   | YouTube Enabled?           | Mark as Scheduled           | Flow 2: Approval & auto-publish                                                             |
| Mark as Scheduled      | Google Sheets (Update)           | Update status to "scheduled" after upload | Schedule TikTok / Instagram / YouTube | Process Queue (loop)        | Flow 2: Approval & auto-publish                                                             |
| PUBLISHER NOTE         | Sticky Note                     | Explains Flow 2 logic                      |                            |                             | Every 15 minutes the workflow reads VideosTable, filters Status=approved, downloads file and schedules it |
| SAVE NOTE              | Sticky Note                     | Explains Flow 1 draft saving               |                            |                             | Formats AI output and appends new row in VideosTable with Status=draft                        |
| AI NOTE                | Sticky Note                     | Explains Flow 1 AI analysis and copy       |                            |                             | Each video is analyzed with Gemini and generates platform-specific social copy                |
| FETCH NOTE             | Sticky Note                     | Explains Flow 1 intake and Drive fetch     |                            |                             | Form collects inputs, Drive lists files, filters videos, builds posting calendar             |
| WORKFLOW INFO          | Sticky Note                     | Overview and setup instructions            |                            |                             | How it works and setup steps with important links                                           |
| FETCH NOTE1            | Sticky Note                     | Explains social network uploads             |                            |                             | Configured for TikTok, Instagram, YouTube; can add more platforms                           |
| SAVE NOTE1             | Sticky Note                     | Explains Flow 1 schedule saving             |                            |                             | Formats AI output plus schedule info and appends new row in VideosTable with Status=approved |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("Start Campaign")**  
   - Form Title: "Social Video Autopilot"  
   - Add required fields: Google Drive Folder ID (string), Profile Username (string), Platforms (dropdown with combinations of TikTok, Instagram, YouTube), Start Date (date string YYYY-MM-DD), Cadence (dropdown: 1_daily, 5_per_week, 3_per_week), Publish Hour (HH:MM), Google Sheet ID (string)  
   - Save and activate webhook.

2. **Add a Set node ("Campaign Settings")**  
   - Map form responses to variables: drive_folder_id, profile_username, platforms_raw, timezone, start_date, cadence, publish_hour, sheet_id

3. **Add a Google Drive node ("Fetch Videos from Drive")**  
   - Operation: List Files  
   - Query: `'{{ $json.drive_folder_id }}' in parents and trashed = false`  
   - Credentials: Set up Google Drive OAuth2

4. **Add a Filter node ("Video Files Only")**  
   - Condition: File name contains ".mp4"

5. **Add a Code node ("Build Schedule Calendar")**  
   - Paste JavaScript code that calculates scheduled dates based on start date and cadence (see original code for logic)  
   - Output enriched items with schedule and config info.

6. **Add a SplitInBatches node ("Process Each Video")**  
   - Default batch size (e.g., 1)

7. **Add a Google Drive node ("Download from Drive")**  
   - Operation: Download file by ID (from current item)  
   - Credentials: Google Drive OAuth2

8. **Add a Google Gemini Video Analysis node ("AI Video Analysis")**  
   - Model: Gemini 2.5 Flash  
   - Input: binary video data  
   - Prompt: Detailed instructions for video analysis including language detection line

9. **Add an AI Agent node ("Generate Social Copy")**  
   - Use Gemini Pro or similar LM with prompt to generate platform-specific social copy (titles, descriptions, hashtags) in JSON only format  
   - Set system message and task instructions as per original prompt

10. **Add a Gemini Pro node ("Gemini Pro")**  
   - Connect as LM for the Generate Social Copy node  
   - Credentials: Google Palm API key

11. **Add an Output Parser node ("Structured Output Parser")**  
   - Set JSON schema example matching expected social copy structure  
   - Enable auto-fix for minor JSON errors

12. **Add a Code node ("Format Content Data")**  
   - JavaScript code to parse AI output, build a row object with draft status and social copy fields, handle parse errors marking status as "error"

13. **Add a Google Sheets node ("Save Draft to Sheet")**  
   - Operation: Append row  
   - Configure with columns: Video ID, Video Name, Index, Status, Schedule Date, Schedule DateTime, TikTok Title, TikTok Description, TikTok Hashtags, Instagram Title, Instagram Description, Instagram Hashtags, YouTube Title, YouTube Description, YouTube Tags, Summary, Profile, Platforms, Created At  
   - Sheet: VideosTable  
   - Credentials: Google Sheets OAuth2  
   - Document ID from Campaign Settings sheet_id variable

14. **Connect "Save Draft to Sheet" back to "Process Each Video" to loop processing all videos**

15. **Create a Schedule Trigger node ("15 mins check")**  
   - Interval: every 15 minutes

16. **Add a Set node ("Publisher Config")**  
   - Hardcode sheet_id, drive_folder_id, profile_username for publishing  
   - (Alternatively, these could be dynamically passed)

17. **Add a Google Sheets node ("Load Content Queue")**  
   - Operation: Read rows from VideosTable sheet  
   - Document ID from Publisher Config sheet_id

18. **Add a Filter node ("Only Approved")**  
   - Filter rows where Status equals "approved"

19. **Add a SplitInBatches node ("Process Queue")**  
   - Batch size 1 or suitable value for processing

20. **Add a Google Drive node ("Get Video File")**  
   - Download file by Video ID from the row

21. **Add a Code node ("Build Upload Payload")**  
   - Build JSON payload with video info, platform flags, social copy, schedule datetime, profile username, and binary video data

22. **Add three If nodes ("TikTok Enabled?", "Instagram Enabled?", "YouTube Enabled?")**  
   - Check boolean flags has_tiktok, has_instagram, has_youtube in payload

23. **Add three Upload-Post nodes ("Schedule TikTok", "Schedule Instagram", "Schedule YouTube")**  
   - Operation: uploadVideo  
   - Configure platform, video, title, description, hashtags/tags, scheduledDate, user profile  
   - Instagram node: set media type to REELS  
   - Credentials: Upload-Post API key

24. **Add a Google Sheets node ("Mark as Scheduled")**  
   - Operation: Update row  
   - Match by Video ID  
   - Set Status to "scheduled"

25. **Connect scheduling nodes to "Mark as Scheduled" and loop back to "Process Queue"**

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                                             |
|------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This template requires Google Drive folder with videos, a copy of the provided Google Sheet template, and API credentials | Setup instructions are included in the Workflow Info sticky note                                                                            |
| Google Sheet Template: https://docs.google.com/spreadsheets/d/1cegJHxj7Kx4Tg8gMr3uixpzToNc62VEvuuz37iFvnRw/edit         | Essential for tracking video drafts, approvals, and scheduling                                                                              |
| Platforms supported out of the box: TikTok, Instagram Reels, YouTube Shorts                                            | Upload-Post supports more platforms; workflow can be extended accordingly                                                                    |
| AI video analysis and copy generation use Google Gemini models (Gemini Flash and Gemini Pro)                            | Requires Google Palm API key and configured n8n Langchain nodes                                                                              |
| API limits, video file sizes, and scheduling conflicts should be monitored and handled                                  | Potential edge cases include API rate limits, large file downloads, invalid scheduling times, and permission errors                         |

---

**Disclaimer:**  
The provided content results exclusively from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.