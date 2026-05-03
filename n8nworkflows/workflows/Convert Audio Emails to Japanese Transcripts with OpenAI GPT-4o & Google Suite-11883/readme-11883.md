Convert Audio Emails to Japanese Transcripts with OpenAI GPT-4o & Google Suite

https://n8nworkflows.xyz/workflows/convert-audio-emails-to-japanese-transcripts-with-openai-gpt-4o---google-suite-11883


# Convert Audio Emails to Japanese Transcripts with OpenAI GPT-4o & Google Suite

### 1. Workflow Overview

This workflow automates the process of converting audio emails into Japanese transcripts, generating AI-based summaries, storing files in Google Drive, logging results in Google Sheets, and notifying relevant parties via Gmail and Slack. It is designed for users who receive voice notes or meeting recordings by email and want quick, structured Japanese transcripts and summaries with archival and notification capabilities.

The workflow is logically divided into five main blocks:

- **1.1 Input Reception and Attachment Preparation:** Detect and extract audio attachments from incoming Gmail messages.
- **1.2 Drive Folder Management and Audio Upload:** Organize storage by creating date-based folders in Google Drive and uploading audio files.
- **1.3 AI Transcription and Summary Generation:** Transcribe audio into Japanese text and generate structured AI summaries.
- **1.4 File Conversion, Upload, and Sharing:** Convert transcripts and summaries into text/Markdown files, upload them to Drive, and share with specified users.
- **1.5 Logging and Notifications:** Log the summarized data into Google Sheets and send notifications via Gmail and Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Attachment Preparation

**Overview:**  
This block monitors Gmail for new emails with audio attachments, extracts each audio file as an individual item, normalizes binary data, and filters to keep only supported audio formats.

**Nodes Involved:**  
- New Gmail with Audio  
- Code in JavaScript  
- Filter

**Node Details:**

- **New Gmail with Audio**  
  - Type: Gmail Trigger  
  - Role: Watches Gmail inbox and downloads attachments from new emails every minute.  
  - Configuration: Polls every minute, downloads attachments (not limited by label), no sender restrictions by default.  
  - Expressions: None beyond standard trigger data.  
  - Connections: Output to Code in JavaScript node.  
  - Edge cases: Possible Gmail API rate limits, attachment download errors, or empty attachments.  
  - Version: 1.3

- **Code in JavaScript**  
  - Type: Code Node (JavaScript)  
  - Role: Splits email attachments into separate items, normalizes binary key naming to "data".  
  - Configuration: Loops over all binary keys (attachments), extracts metadata (filename, extension, mimetype), forwards binary under a unified key.  
  - Key Expressions: Accesses `item.binary`, standardizes to `binary.data`.  
  - Inputs: Items with attachments from Gmail trigger.  
  - Outputs: One item per attachment with normalized binary.  
  - Edge cases: Emails without attachments result in no items; keys for binary data must be present or skip.  
  - Version: 2

- **Filter**  
  - Type: Filter Node  
  - Role: Filters items to keep only audio files based on file extension regex matching `mp3|wav|m4a|ogg`.  
  - Configuration: Condition on `fileExtension` JSON property using regex.  
  - Input: Items from Code node.  
  - Output: Items matching audio formats only.  
  - Edge cases: Non-audio attachments filtered out; unsupported audio formats excluded unless regex updated.  
  - Version: 2.2

---

#### 2.2 Drive Folder Management and Audio Upload

**Overview:**  
Generates a year/month folder path based on the current date, creates the folder in Google Drive if it does not exist, merges folder metadata with file info, and uploads the audio file to the created folder.

**Nodes Involved:**  
- Generate Year-Month Folder Path  
- Create Folder If Not Exists  
- Merge  
- Upload Audio to Drive

**Node Details:**

- **Generate Year-Month Folder Path**  
  - Type: Code Node (JavaScript)  
  - Role: Constructs folder path strings in `YYYY/MM` format from current date; passes original binary audio forward.  
  - Configuration: Returns JSON with folderPath, year, month, and original item data; binary data preserved.  
  - Edge cases: Date retrieval should always work; binary data must be passed correctly to avoid data loss.  
  - Version: 2

- **Create Folder If Not Exists**  
  - Type: Google Drive Node  
  - Role: Creates a folder named as generated path under a specified parent Drive folder if missing.  
  - Configuration: Uses folderPath as folder name; parent folder ID is hardcoded; searches and creates folder if absent.  
  - Edge cases: Permissions errors on Drive, invalid parent folder ID, API limits.  
  - Version: 3

- **Merge**  
  - Type: Merge Node  
  - Role: Combines two streams — one with folder metadata and one with file metadata — by position to consolidate info for upload.  
  - Configuration: Mode "combine by position".  
  - Inputs: From Create Folder If Not Exists (folder metadata) and Generate Year-Month Folder Path (file metadata).  
  - Edge cases: Unequal item counts cause merge mismatches.  
  - Version: 3.2

- **Upload Audio to Drive**  
  - Type: Google Drive Node  
  - Role: Uploads the audio binary file to the created folder in Google Drive.  
  - Configuration: File name from merged data; folder ID from folder creation node; uploads binary under `data` key.  
  - Edge cases: Upload failures, binary data missing or corrupted, Drive quota exceeded.  
  - Version: 3

---

#### 2.3 AI Transcription and Summary Generation

**Overview:**  
This block sends the audio file to OpenAI's transcription API for Japanese verbatim transcription and then generates a structured AI summary (JSON) from the transcript.

**Nodes Involved:**  
- HTTP Request  
- Generate AI Summary  
- Edit Fields

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request Node  
  - Role: Calls OpenAI Audio Transcriptions API (`gpt-4o-transcribe`) to transcribe audio.  
  - Configuration:  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Sends binary audio as form data under `file`.  
    - Model: `gpt-4o-transcribe`  
    - Prompt: Japanese verbatim transcription instructions (no summarization, no guessing).  
    - Language: Japanese (`ja`)  
    - Temperature: 0 (deterministic)  
    - Response format: text  
    - Authentication: HTTP Bearer Auth with OpenAI API key.  
  - Inputs: Audio binary from Merge node.  
  - Outputs: Plain text transcript in `data` or `text` JSON fields.  
  - Edge cases: API rate limits, transcription errors, unsupported audio formats, network errors.  
  - Version: 4.2

- **Generate AI Summary**  
  - Type: Langchain OpenAI Node  
  - Role: Uses GPT-4o to analyze Japanese transcript and generate a JSON summary with fields: title, points, decisions, actionItems.  
  - Configuration:  
    - Model: GPT-4o  
    - Prompts instruct JSON structure output in Japanese.  
    - JSON output enabled for direct object parsing.  
  - Inputs: Transcript text from HTTP Request node.  
  - Outputs: Structured JSON summary.  
  - Edge cases: Incomplete or malformed transcripts cause summary issues; API quota; prompt failures.  
  - Version: 1.8

- **Edit Fields**  
  - Type: Set Node  
  - Role: Formats the JSON summary into a readable Markdown string (`summaryContent`) for user-friendly display.  
  - Configuration: Uses expressions to construct Markdown with title, points, decisions, action items as bullet lists.  
  - Inputs: AI summary JSON from Generate AI Summary.  
  - Outputs: Adds `summaryContent` string to JSON.  
  - Edge cases: Missing keys in JSON cause empty lists replaced by 'なし' (none).  
  - Version: 3.4

---

#### 2.4 File Conversion, Upload, and Sharing

**Overview:**  
Converts transcript and summary data into text and Markdown files respectively, uploads them to Google Drive, and shares them with the original email sender and last modifying user.

**Nodes Involved:**  
- Convert Transcript to TXT  
- Upload Transcript to Drive  
- Share Transcript File  
- Convert Summary to MD  
- Upload Summary to Drive  
- Share Summary File

**Node Details:**

- **Convert Transcript to TXT**  
  - Type: Convert To File Node  
  - Role: Converts transcript text into a `.txt` file binary.  
  - Configuration: Filename derived from original file name + `.txt`; source property is `data` field (transcript text).  
  - Inputs: Transcript text from HTTP Request node.  
  - Outputs: Binary file for upload.  
  - Edge cases: Empty transcripts produce empty files.  
  - Version: 1.1

- **Upload Transcript to Drive**  
  - Type: Google Drive Node  
  - Role: Uploads the `.txt` transcript file to the created Drive folder.  
  - Configuration: Filename from binary file; folder is the created month folder.  
  - Edge cases: Upload failures, permission issues.  
  - Version: 3

- **Share Transcript File**  
  - Type: Google Drive Node  
  - Role: Shares the transcript file with the original email sender (reader role).  
  - Configuration: Uses file ID from upload node; email address from `owners[0].emailAddress` in JSON.  
  - Edge cases: Sharing permission errors, missing email addresses.  
  - Version: 3

- **Convert Summary to MD**  
  - Type: Convert To File Node  
  - Role: Converts Markdown summary string to `.md` file binary.  
  - Configuration: Filename based on original file name + `_summary.md`; uses `summaryContent` property.  
  - Outputs: Binary file for upload.  
  - Edge cases: Empty summaries produce empty files.  
  - Version: 1.1

- **Upload Summary to Drive**  
  - Type: Google Drive Node  
  - Role: Uploads the `.md` summary file to the same Drive folder.  
  - Configuration: Filename from binary; folder same as transcript.  
  - Edge cases: Upload failures.  
  - Version: 3

- **Share Summary File**  
  - Type: Google Drive Node  
  - Role: Shares the summary file with the last modifying user (reader role).  
  - Configuration: File ID from upload; email from `lastModifyingUser.emailAddress`.  
  - Edge cases: Sharing permission errors, missing email data.  
  - Version: 3

---

#### 2.5 Logging and Notifications

**Overview:**  
Collects relevant information including subject, file name, transcripts, summaries, and file share links, appends a row in Google Sheets, and sends notifications via Gmail and Slack.

**Nodes Involved:**  
- Prepare Sheets Data  
- Log to Google Sheets  
- Send Gmail Notification  
- Send Slack Notification

**Node Details:**

- **Prepare Sheets Data**  
  - Type: Code Node (JavaScript)  
  - Role: Gathers subject, file name, transcript, summary, and Drive share links into a single JSON object for logging.  
  - Configuration:  
    - Reads data from multiple nodes (Merge, HTTP Request, Edit Fields, Upload nodes).  
    - Handles fallback logic for missing data gracefully.  
  - Edge cases: Missing nodes or fields handled with defaults; malformed data may cause partial info.  
  - Version: 2

- **Log to Google Sheets**  
  - Type: Google Sheets Node  
  - Role: Appends a row to a Google Sheets document with collected data.  
  - Configuration:  
    - Sheet name: "シート1"  
    - Document ID: hardcoded ID (should be updated per user).  
    - Columns mapped to JSON fields: Timestamp, Subject, File Name, Summary, Links, Transcript.  
  - Edge cases: API rate limits, permission issues, invalid sheet name or document ID.  
  - Version: 4.7

- **Send Gmail Notification**  
  - Type: Gmail Node  
  - Role: Sends an email to the original sender with the meeting summary and links.  
  - Configuration:  
    - Recipient: from original email's `from` address.  
    - Subject: fixed "n8n".  
    - Message body: HTML formatted with subject, summary, and links.  
  - Edge cases: SMTP errors, invalid email addresses.  
  - Version: 2.1

- **Send Slack Notification**  
  - Type: Slack Node  
  - Role: Sends a formatted message to a Slack channel with summary and links.  
  - Configuration:  
    - Authenticated with OAuth2.  
    - Channel ID hardcoded (e.g., C0A3GEB36SJ).  
    - Message text includes subject, file name, summary, and links.  
    - Includes link to workflow in Slack message.  
  - Edge cases: Slack API rate limits, invalid channel, OAuth token expiry.  
  - Version: 2.3

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                           | Input Node(s)                    | Output Node(s)                         | Sticky Note                                                                                                                        |
|----------------------------|--------------------------|-----------------------------------------|---------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| New Gmail with Audio        | Gmail Trigger            | Gmail polling and attachment download   | —                               | Code in JavaScript                    | ## 1) Gmail intake + attachment prep<br>- Polls Gmail and downloads attachments<br>- Extend filter regex for more formats      |
| Code in JavaScript          | Code                     | Split attachments, normalize binary     | New Gmail with Audio            | Filter                               | ## 1) Gmail intake + attachment prep<br>- Splits attachments into items<br>- Normalizes binary keys                              |
| Filter                     | Filter                   | Keep only supported audio files          | Code in JavaScript              | Generate Year-Month Folder Path       | ## 1) Gmail intake + attachment prep<br>- Filters audio by extension regex                                                    |
| Generate Year-Month Folder Path | Code                 | Create folder path based on date         | Filter                        | Create Folder If Not Exists, Merge    | ## 2) Drive folder + upload original audio<br>- Builds YYYY/MM folder path                                                     |
| Create Folder If Not Exists | Google Drive             | Create folder at Drive if missing        | Generate Year-Month Folder Path | Merge                               | ## 2) Drive folder + upload original audio<br>- Creates month folder if not present                                            |
| Merge                      | Merge                    | Combine folder info and file info        | Create Folder If Not Exists, Generate Year-Month Folder Path | HTTP Request, Upload Audio to Drive | ## 2) Drive folder + upload original audio<br>- Combines folder and file metadata                                               |
| Upload Audio to Drive       | Google Drive             | Upload audio binary to Drive folder      | Merge                         | HTTP Request                        | ## 2) Drive folder + upload original audio<br>- Uploads original audio                                                           |
| HTTP Request               | HTTP Request             | Call OpenAI transcription API            | Merge                         | Generate AI Summary, Convert Transcript to TXT | ## 3) Transcribe + summarize with AI<br>- Calls OpenAI API for Japanese verbatim transcription                                  |
| Generate AI Summary         | Langchain OpenAI         | Generate structured summary from transcript | HTTP Request                  | Edit Fields                        | ## 3) Transcribe + summarize with AI<br>- Creates JSON summary of transcript                                                   |
| Edit Fields                | Set                      | Format summary JSON into Markdown         | Generate AI Summary            | Convert Summary to MD               | ## 3) Transcribe + summarize with AI<br>- Formats summary as Markdown                                                         |
| Convert Transcript to TXT   | Convert To File          | Convert transcript text to .txt file      | HTTP Request                  | Upload Transcript to Drive          | ## 4) Save transcript + summary files to Drive<br>- Converts transcript to text file                                           |
| Upload Transcript to Drive  | Google Drive             | Upload transcript .txt file                | Convert Transcript to TXT      | Share Transcript File               | ## 4) Save transcript + summary files to Drive<br>- Uploads transcript file                                                    |
| Share Transcript File       | Google Drive             | Share transcript file with original sender | Upload Transcript to Drive    | Prepare Sheets Data                | ## 4) Save transcript + summary files to Drive<br>- Shares transcript file                                                     |
| Convert Summary to MD       | Convert To File          | Convert summary Markdown to .md file       | Edit Fields                   | Upload Summary to Drive            | ## 4) Save transcript + summary files to Drive<br>- Converts summary to Markdown file                                           |
| Upload Summary to Drive     | Google Drive             | Upload summary .md file                     | Convert Summary to MD         | Share Summary File                | ## 4) Save transcript + summary files to Drive<br>- Uploads summary file                                                       |
| Share Summary File          | Google Drive             | Share summary file with last modifying user | Upload Summary to Drive      | Prepare Sheets Data                | ## 4) Save transcript + summary files to Drive<br>- Shares summary file                                                        |
| Prepare Sheets Data         | Code                     | Collect data for logging and notification | Share Transcript File, Share Summary File | Log to Google Sheets           | ## 5) Log + notify<br>- Prepares data from multiple sources for logging and notifications                                      |
| Log to Google Sheets        | Google Sheets            | Append row with summary data                | Prepare Sheets Data           | Send Gmail Notification, Send Slack Notification | ## 5) Log + notify<br>- Logs summary and transcript info to Google Sheets                                                     |
| Send Gmail Notification     | Gmail                    | Email summary to original sender            | Log to Google Sheets          | —                                 | ## 5) Log + notify<br>- Sends email notification with summary and links                                                       |
| Send Slack Notification     | Slack                    | Post summary message to Slack channel       | Log to Google Sheets          | —                                 | ## 5) Log + notify<br>- Sends Slack notification with summary and links                                                       |
| Template Overview (read me) | Sticky Note              | Workflow overview text                       | —                            | —                                 | ## Overview<br>This template watches Gmail for audio emails, transcribes, summarizes, saves, logs, and notifies.              |
| Step 1 - Intake             | Sticky Note              | Describes intake block                        | —                            | —                                 | ## 1) Gmail intake + attachment prep<br>Details nodes and customization info                                                  |
| Step 2 - Drive storage      | Sticky Note              | Describes Drive storage block                  | —                            | —                                 | ## 2) Drive folder + upload original audio<br>Details nodes and customization info                                           |
| Step 3 - AI transcription & summary | Sticky Note       | Describes AI transcription and summary block  | —                            | —                                 | ## 3) Transcribe + summarize with AI<br>Details nodes and customization info                                                 |
| Step 4 - Save & share files | Sticky Note              | Describes file saving and sharing block         | —                            | —                                 | ## 4) Save transcript + summary files to Drive<br>Details nodes and customization info                                       |
| Step 5 - Log & notifications | Sticky Note             | Describes logging and notification block        | —                            | —                                 | ## 5) Log + notify<br>Details nodes and customization info                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node (New Gmail with Audio):**  
   - Type: Gmail Trigger  
   - Poll interval: every minute  
   - Download attachments enabled  
   - Leave label filters empty (or add as needed)  
   - Credentials: Configure Gmail OAuth2 credentials

2. **Add Code Node (Code in JavaScript):**  
   - Purpose: Split email attachments into individual items and normalize binary keys  
   - Paste the provided JavaScript code that iterates over binary attachment keys, creating one item per attachment with binary under `data` key.  
   - Connect output from Gmail Trigger to this node.

3. **Add Filter Node:**  
   - Filter: Keep only items where `fileExtension` matches regex `mp3|wav|m4a|ogg`  
   - Connect output from Code Node to this node.

4. **Add Code Node (Generate Year-Month Folder Path):**  
   - JavaScript to get current date and create folder path string `YYYY/MM`  
   - Pass original binary data forward unchanged  
   - Connect output from Filter Node to this node.

5. **Add Google Drive Node (Create Folder If Not Exists):**  
   - Operation: Create folder if missing  
   - Folder name: Use expression `{{$json.folderPath}}`  
   - Parent folder ID: Set your Drive parent folder ID  
   - Credentials: Google Drive OAuth2  
   - Connect output from Year-Month Folder Path Node to this node.

6. **Add Merge Node:**  
   - Mode: Combine by Position  
   - Connect outputs:  
     - Create Folder If Not Exists → second input  
     - Generate Year-Month Folder Path → first input

7. **Add Google Drive Node (Upload Audio to Drive):**  
   - Operation: Upload file  
   - File name: Use expression from merged data, e.g., `{{$json.fileExtension}}` or filename property  
   - Folder ID: Use ID from Create Folder If Not Exists node  
   - Upload binary from `binary.data`  
   - Connect output from Merge Node (main 0) to this node.

8. **Add HTTP Request Node:**  
   - Method: POST  
   - URL: `https://api.openai.com/v1/audio/transcriptions`  
   - Content-Type: multipart/form-data  
   - Body parameters:  
     - file: binary data (`data`)  
     - model: `gpt-4o-transcribe`  
     - prompt: Japanese verbatim transcription instructions  
     - temperature: 0  
     - language: `ja`  
     - response_format: `text`  
   - Credentials: OpenAI API key (HTTP Bearer Auth)  
   - Connect output from Merge Node (main 1) to this node.

9. **Add Langchain OpenAI Node (Generate AI Summary):**  
   - Model: GPT-4o  
   - Prompt: Analyze Japanese transcript and generate JSON summary with fields: title, points, decisions, actionItems  
   - JSON output enabled  
   - Connect output from HTTP Request node to this node.

10. **Add Set Node (Edit Fields):**  
    - Create `summaryContent` string formatted as Markdown using expressions to populate title, points, decisions, actionItems from AI summary JSON.  
    - Connect output from Generate AI Summary node to this node.

11. **Add Convert To File Node (Convert Summary to MD):**  
    - Operation: toText  
    - Source property: `summaryContent`  
    - File name: original filename + `_summary.md`  
    - Connect output from Edit Fields node to this node.

12. **Add Convert To File Node (Convert Transcript to TXT):**  
    - Operation: toText  
    - Source property: transcription text from HTTP Request node (`data`)  
    - File name: original filename + `.txt`  
    - Connect output from HTTP Request node to this node.

13. **Add Google Drive Node (Upload Summary to Drive):**  
    - Upload summary `.md` binary file  
    - Folder ID from Create Folder If Not Exists node  
    - Connect output from Convert Summary to MD node to this node.

14. **Add Google Drive Node (Upload Transcript to Drive):**  
    - Upload transcript `.txt` binary file  
    - Folder ID from Create Folder If Not Exists node  
    - Connect output from Convert Transcript to TXT node to this node.

15. **Add Google Drive Node (Share Summary File):**  
    - Operation: Share  
    - File ID: from Upload Summary to Drive node  
    - Permission: reader, user, email address from `lastModifyingUser.emailAddress`  
    - Connect output from Upload Summary to Drive node to this node.

16. **Add Google Drive Node (Share Transcript File):**  
    - Operation: Share  
    - File ID: from Upload Transcript to Drive node  
    - Permission: reader, user, email address from `owners[0].emailAddress`  
    - Connect output from Upload Transcript to Drive node to this node.

17. **Add Code Node (Prepare Sheets Data):**  
    - Collect subject, file name, transcript, summary, and share links into one JSON object  
    - Use JavaScript code that accesses various nodes’ outputs safely  
    - Connect outputs from Share Summary File and Share Transcript File nodes to this node.

18. **Add Google Sheets Node (Log to Google Sheets):**  
    - Operation: Append row  
    - Document ID and sheet name configured as per your Google Sheets  
    - Map columns to JSON fields: Timestamp, Subject, File Name, Summary, Links, Transcript  
    - Credentials: Google Sheets OAuth2  
    - Connect output from Prepare Sheets Data node to this node.

19. **Add Gmail Node (Send Gmail Notification):**  
    - Send email to original sender’s email address from Gmail trigger  
    - Subject: e.g., "n8n"  
    - Body: HTML formatted with subject, summary, and links from logged data  
    - Credentials: Gmail OAuth2  
    - Connect output from Log to Google Sheets node to this node.

20. **Add Slack Node (Send Slack Notification):**  
    - Send message to specified Slack channel (channel ID hardcoded or parameterized)  
    - Message includes subject, file name, summary, links  
    - Authentication: Slack OAuth2  
    - Connect output from Log to Google Sheets node to this node.

21. **Add Sticky Notes (Optional):**  
    - Add sticky notes for overview and block descriptions to help understanding.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| This template watches Gmail for emails with audio attachments, transcribes to Japanese, summarizes meetings in JSON & Markdown, saves to Drive and logs results.| Overview sticky note inside workflow ("Template Overview (read me)")                                                    |
| Supported audio formats can be extended by modifying the regex in the Filter node (`mp3|wav|m4a|ogg`).                                                           | Sticky note "Step 1 - Intake"                                                                                            |
| Folder structure for Drive can be customized in the "Generate Year-Month Folder Path" node (e.g., add day).                                                     | Sticky note "Step 2 - Drive storage"                                                                                     |
| Transcription and summary prompts can be adjusted for different languages or formats.                                                                            | Sticky note "Step 3 - AI transcription & summary"                                                                        |
| Notifications target Gmail original sender email and Slack channel; update credentials and channel IDs as needed before deployment.                              | Sticky note "Step 5 - Log & notifications"                                                                                |
| Google Sheets document and sheet name are hardcoded; update document ID and sheet name to match your environment.                                              | Google Sheets node configuration                                                                                         |
| OpenAI API usage requires API key with transcription and chat completions enabled; monitor quota and handle errors accordingly.                                  | HTTP Request and OpenAI nodes                                                                                             |
| Slack integration requires OAuth2 app with chat:write permission and correct channel IDs.                                                                        | Slack node configuration                                                                                                |

---

**Disclaimer:**  
The above documentation is based exclusively on an n8n workflow JSON export. It complies strictly with content policies and contains no illegal or offensive material. All processed data is public and legal.