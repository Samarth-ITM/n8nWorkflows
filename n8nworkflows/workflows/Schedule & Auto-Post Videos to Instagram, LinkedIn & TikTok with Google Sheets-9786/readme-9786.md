Schedule & Auto-Post Videos to Instagram, LinkedIn & TikTok with Google Sheets

https://n8nworkflows.xyz/workflows/schedule---auto-post-videos-to-instagram--linkedin---tiktok-with-google-sheets-9786


# Schedule & Auto-Post Videos to Instagram, LinkedIn & TikTok with Google Sheets

### 1. Workflow Overview

This workflow automates the scheduling and posting of videos to multiple social media platforms—Instagram, LinkedIn, and TikTok—based on a Google Sheets schedule. It runs twice daily (9 AM and 9 PM Santiago, Chile time), reads posts marked as ready from Google Sheets, matches the scheduled posting time with the current trigger time, uploads the video content to the specified platforms, updates the post status in Google Sheets, and sends a Telegram notification with the posting results.

The workflow is logically grouped into the following blocks:

- **1.1 Automatic Trigger:** Scheduled execution twice daily with timezone adjustment.
- **1.2 Fetch Scheduled Posts:** Reads Google Sheets for posts marked ready to post at the current time.
- **1.3 Time Formatting & Validation:** Converts trigger time to a readable format and verifies if it's time to post.
- **1.4 Data Preparation:** Extracts and formats post content and metadata for upload.
- **1.5 Account Configuration:** Prepares social media account IDs for posting.
- **1.6 Multi-Platform Video Upload:** Uploads video content simultaneously to Instagram, LinkedIn, and TikTok.
- **1.7 Status Update:** Marks posts as published in Google Sheets to prevent duplicates.
- **1.8 Telegram Notification:** Sends a detailed report of the posting results via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Automatic Trigger

**Overview:**  
Triggers the workflow twice daily at 9:00 AM and 9:00 PM in Santiago, Chile timezone.

**Nodes Involved:**  
- Schedule Trigger  
- Code  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow at two fixed times daily (09:00 and 21:00 hours).  
  - Configuration: Two triggerAtHour values set to 9 and 21, no specific timezone parameter (handled later in code).  
  - Input: None (trigger node)  
  - Output: Passes a trigger event object with timestamp or scheduled time.  
  - Edge cases: Workflow depends on correct system time and node execution; no timezone adjustment here.  

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Converts the trigger timestamp to a formatted Spanish date/time string matching Google Sheets’ schedule format.  
  - Configuration: Runs JS to convert current trigger time to "day de month a las hour am/pm" format in America/Santiago timezone, including day of week.  
  - Key expressions: Uses Date object, locale string with timezone, arrays of Spanish month/day names, 12-hour time format conversion.  
  - Input: Trigger timestamp from Schedule Trigger node.  
  - Output: JSON with `Readable time` (e.g., "16 de Octubre a las 9 am") and `Day of week`.  
  - Edge cases: If timezone conversion fails or trigger data is missing, could misalign scheduling.  

---

#### 1.2 Fetch Scheduled Posts

**Overview:**  
Reads Google Sheets to find a post with Status "Listo para postear" that matches the formatted trigger time.

**Nodes Involved:**  
- Google Sheets  
- If  

**Node Details:**

- **Google Sheets**  
  - Type: Google Sheets  
  - Role: Retrieves the first post with Status = "Listo para postear" from the configured sheet.  
  - Configuration: Filter on "Status" column with value "Listo para postear", returns first match only. Document and sheet IDs are set to specific Google Sheet.  
  - Input: Trigger from Code node.  
  - Output: Post data JSON with fields: Title, Copy, Video Link, Status, Fecha.Hora, row_number, etc.  
  - Edge cases: No matching rows causes no output; network or auth errors possible.  

- **If**  
  - Type: If  
  - Role: Compares the formatted trigger time (`Readable time` from Code node) with the scheduled post time (`Fecha.Hora` in Google Sheets).  
  - Configuration: Condition set to equality between the two time strings.  
  - Input: Data from Google Sheets and Code nodes.  
  - Output: Proceed if times match; otherwise stop workflow.  
  - Edge cases: Time format mismatch, case sensitivity issues, missing fields result in false condition.  

---

#### 1.3 Data Preparation

**Overview:**  
Extracts and formats post data from Google Sheets for the upload API, including setting flags and renaming fields.

**Nodes Involved:**  
- Format Drive Content  
- Social Media Account IDs  

**Node Details:**

- **Format Drive Content**  
  - Type: Set  
  - Role: Maps Google Sheets fields into variables needed for upload, e.g., Title, Copy, Status, video_url, is_drive_file (boolean true), and row_number.  
  - Configuration: Uses expressions to assign values directly from Google Sheets item JSON.  
  - Input: Output from If node (only if time matched).  
  - Output: Structured JSON with necessary fields for upload.  
  - Edge cases: Missing or malformed fields in Sheets can cause runtime errors or incorrect data submission.  

- **Social Media Account IDs**  
  - Type: Set  
  - Role: Prepares social media account identifiers (Facebook Page ID, Pinterest Board ID) for use in posting. Currently, those fields are empty but prepared for future extension.  
  - Configuration: Sets empty strings for these IDs, includes other fields from previous node unchanged.  
  - Input: Output from Format Drive Content.  
  - Output: JSON enriched with account IDs.  
  - Edge cases: Empty ID fields mean some platforms may not be properly targeted if those IDs are required.  

---

#### 1.4 Multi-Platform Video Upload

**Overview:**  
Uploads the video post to Instagram, LinkedIn, and TikTok simultaneously using the UploadPost API.

**Nodes Involved:**  
- Upload a video  

**Node Details:**

- **Upload a video**  
  - Type: UploadPost (custom integration node)  
  - Role: Uploads video content to multiple platforms using unified API.  
  - Configuration:  
    - User: "JoseAI" (hardcoded user/credential name)  
    - Title: concatenated Title and Copy fields from previous node as caption  
    - Video: video_url from previous node (Google Drive link)  
    - Platforms: Instagram, LinkedIn, TikTok  
    - Operation: uploadVideo  
  - Credentials: UploadPost API account connected with social media profiles.  
  - Input: Social media account data and formatted post data.  
  - Output: JSON results array with platform-wise upload success info, post URLs, and success flags.  
  - Edge cases: API failure, invalid credentials, video not accessible, unsupported formats, network timeouts.  

---

#### 1.5 Status Update

**Overview:**  
Updates the Google Sheets row for the post, marking it as "Posteado" to avoid reposting.

**Nodes Involved:**  
- Google Sheets1  

**Node Details:**

- **Google Sheets1**  
  - Type: Google Sheets  
  - Role: Updates the Status column of the posted row to "Posteado" using row_number as key.  
  - Configuration: Update operation with matching column "row_number". Changes Status field only.  
  - Input: Output from Upload a video node (to proceed only after upload).  
  - Output: Confirmation of sheet update.  
  - Edge cases: Sheet write conflicts, incorrect row_number, auth errors, network failures.  

---

#### 1.6 Telegram Notification

**Overview:**  
Sends a detailed Telegram message summarizing the upload results for each platform.

**Nodes Involved:**  
- Send a text message  

**Node Details:**

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends a message with post results including platform name, post URL (clickable), and success status for Instagram, LinkedIn, and TikTok.  
  - Configuration:  
    - Chat ID: "Your Chat Id" placeholder (must be configured)  
    - Text: Template with embedded expressions extracting results from Upload a video node output.  
    - Append Attribution: Disabled.  
  - Credentials: Telegram API with appropriate bot token.  
  - Input: Output from Google Sheets1 node (after status update).  
  - Output: Telegram message sent confirmation.  
  - Edge cases: Invalid chat ID, bot permissions, network failure, malformed template variables.  

---

### 3. Summary Table

| Node Name              | Node Type                 | Functional Role                              | Input Node(s)        | Output Node(s)          | Sticky Note                                                                                                     |
|------------------------|---------------------------|----------------------------------------------|----------------------|-------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger          | Triggers workflow twice daily at 9 AM & 9 PM| -                    | Google Sheets           | Runs twice daily at 9:00 AM and 9:00 PM (America/Santiago timezone). Modify times in configuration.            |
| Google Sheets          | Google Sheets             | Fetches first post with Status "Listo para postear" | Schedule Trigger     | Code                    | Reads scheduled posts; filters on Status = "Listo para postear". Requires columns: Title, Copy, Video Link...   |
| Code                   | Code                      | Converts trigger timestamp to Spanish readable time | Google Sheets         | If                      | Converts trigger time to format matching Google Sheets "Fecha.Hora" column, e.g., "16 de Octubre a las 9 am".  |
| If                     | If                        | Checks if current time matches scheduled post time | Code                  | Format Drive Content      | Compares formatted current time with Google Sheets schedule time to decide if posting proceeds.                |
| Format Drive Content    | Set                       | Formats data for upload API including video URL | If                   | Social Media Account IDs | Extracts Title, Copy, Status, video_url (Drive link), is_drive_file flag, row_number for upload.                |
| Social Media Account IDs| Set                       | Prepares social media account identifiers     | Format Drive Content  | Upload a video           | Prepares account IDs like Facebook Page ID and Pinterest Board ID (currently empty).                           |
| Upload a video         | UploadPost (custom)       | Uploads video simultaneously to Instagram, LinkedIn, TikTok | Social Media Account IDs | Google Sheets1          | Posts video with combined Title + Copy. Uses UploadPost API credentials.                                       |
| Google Sheets1          | Google Sheets             | Updates post Status to "Posteado" in sheet    | Upload a video        | Send a text message       | Marks post as published to prevent duplicates, matched by row_number.                                         |
| Send a text message     | Telegram                  | Sends Telegram notification with posting results | Google Sheets1        | -                       | Sends detailed success report per platform with clickable URLs.                                               |
| Sticky Note Main        | Sticky Note               | Overall workflow description                   | -                    | -                       | # Automated Social Media Video Posting. Key features listed.                                                  |
| Sticky Note 1           | Sticky Note               | Explains schedule trigger configuration       | -                    | -                       | Runs twice daily at 9:00 and 21:00 hrs Santiago timezone.                                                     |
| Sticky Note 2           | Sticky Note               | Explains Google Sheets fetching logic         | -                    | -                       | Reads posts with Status = "Listo para postear". Required columns listed.                                      |
| Sticky Note 3           | Sticky Note               | Explains time formatter code node              | -                    | -                       | Converts trigger timestamp to Spanish readable format matching Sheets.                                        |
| Sticky Note 4           | Sticky Note               | Explains If node time validation                | -                    | -                       | Checks if formatted times match; stops workflow if no match.                                                  |
| Sticky Note 5           | Sticky Note               | Explains data preparation node                  | -                    | -                       | Extracts and formats post data for upload API.                                                                |
| Sticky Note 6           | Sticky Note               | Explains social media account IDs node          | -                    | -                       | Prepares account IDs for posting.                                                                             |
| Sticky Note 7           | Sticky Note               | Explains multi-platform video upload node       | -                    | -                       | Uploads video to Instagram, LinkedIn, TikTok using UploadPost API.                                            |
| Sticky Note 8           | Sticky Note               | Explains Google Sheets update node               | -                    | -                       | Updates Status to "Posteado" to prevent duplicate posts.                                                      |
| Sticky Note 9           | Sticky Note               | Explains Telegram notification node              | -                    | -                       | Sends detailed Telegram report with platform, URL, and success status.                                       |
| Sticky Note             | Sticky Note               | Provides Google Sheets document link             | -                    | -                       | https://docs.google.com/spreadsheets/d/1ZKbnD7GM6eAHKzyDY98cvz9jMUib1HLU7d9gohYkR3s/edit?usp=sharing          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set two trigger times: 9:00 and 21:00 hours.  
   - No timezone setting here; handled in code.  

2. **Add a Google Sheets node ("Google Sheets"):**  
   - Operation: Read  
   - Document ID: Use your Google Sheet document ID.  
   - Sheet Name: Use the sheet with scheduled posts (e.g., "Video" or gid=0).  
   - Filter rows: Status column equals "Listo para postear".  
   - Return only the first match.  
   - Connect Schedule Trigger output to this node.  
   - Configure Google Sheets OAuth2 credentials.  

3. **Add a Code node ("Code"):**  
   - Use JavaScript to convert the trigger timestamp to the formatted Spanish date/time string matching Google Sheets "Fecha.Hora".  
   - Use timezone "America/Santiago".  
   - Output JSON with keys: `Readable time` and `Day of week`.  
   - Connect Google Sheets output to this node.  

4. **Add an If node ("If"):**  
   - Condition: Check if Code node's `Readable time` equals Google Sheets' `Fecha.Hora` field of the fetched post.  
   - Use string equality, case sensitive.  
   - Connect Code output to If, and Google Sheets data also accessible via expressions.  

5. **Add a Set node ("Format Drive Content"):**  
   - Map fields from Google Sheets post data:  
     - Title, Copy, Status, Video Link (to video_url), set is_drive_file to true, row_number.  
   - Use expressions to pull values from Google Sheets node.  
   - Connect If's true branch output here.  

6. **Add a Set node ("Social Media Account IDs"):**  
   - Prepare social media account identifiers fields: facebook id (Page Id), board id (Pinterest), etc. Set empty strings or actual IDs if available.  
   - Include other fields unchanged.  
   - Connect Format Drive Content output here.  

7. **Add UploadPost node ("Upload a video"):**  
   - Operation: uploadVideo  
   - Platforms: Instagram, LinkedIn, TikTok  
   - User: Set to your UploadPost account user (e.g., "JoseAI")  
   - Title: Combine Title + Copy as caption using expressions.  
   - Video: Use video_url from previous node.  
   - Configure UploadPost API credentials.  
   - Connect Social Media Account IDs output here.  

8. **Add a Google Sheets node ("Google Sheets1"):**  
   - Operation: Update  
   - Document and Sheet: same as before.  
   - Match rows by "row_number".  
   - Update Status column to "Posteado".  
   - Connect Upload a video output here.  
   - Configure credentials as before.  

9. **Add a Telegram node ("Send a text message"):**  
   - Chat ID: Your Telegram chat ID where you want notifications sent.  
   - Message text: Use expressions to include platform, post URL, and success flag from Upload a video node results.  
   - Disable "append attribution".  
   - Configure Telegram API credentials.  
   - Connect Google Sheets1 output here.  

10. **Add Sticky Notes:**  
    - Add explanatory sticky notes as per descriptions to document each block for maintainability.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed for automated video posting with multi-platform support and status tracking. | Main workflow purpose.                                                                           |
| Google Sheets document link: https://docs.google.com/spreadsheets/d/1ZKbnD7GM6eAHKzyDY98cvz9jMUib1HLU7d9gohYkR3s/edit?usp=sharing | Source for scheduling and tracking posts.                                                       |
| Timezone handling is done in a custom Code node for compatibility with Google Sheets time format.   | Ensures scheduling correctness in America/Santiago timezone.                                    |
| UploadPost API credentials must have connected profiles for Instagram, LinkedIn, and TikTok.         | Required for actual posting.                                                                     |
| Telegram bot credentials and chat ID must be correctly configured to receive notifications.          | For real-time post status updates.                                                              |
| The workflow prevents duplicate posting by updating the post status in Google Sheets after posting. | Ensures idempotency and workflow safety.                                                        |
| Possible failure points include Google Sheets auth issues, UploadPost API errors, time mismatches, and Telegram message failures. | Monitoring and error handling recommended.                                                      |

---

**Disclaimer:** The content analyzed and documented here originates solely from an automated n8n workflow. It strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.