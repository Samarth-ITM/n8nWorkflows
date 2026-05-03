Convert YouTube Videos to Shorts with Klap & Auto-Post to Multiple Social Platforms

https://n8nworkflows.xyz/workflows/convert-youtube-videos-to-shorts-with-klap---auto-post-to-multiple-social-platforms-5608


# Convert YouTube Videos to Shorts with Klap & Auto-Post to Multiple Social Platforms

### 1. Workflow Overview

This workflow automates the process of converting full-length YouTube videos into viral Shorts clips using the Klap API, scheduling their publication via Google Sheets data, and auto-posting the resulting Shorts across multiple social media platforms through Blotato API. It is designed for content creators and social media managers who want to maximize video reach by repurposing YouTube content into short-form videos and scheduling multi-platform distribution efficiently.

The workflow consists of three main logical blocks:

- **1.1 Input Reception and Validation:** Receives YouTube video URLs and the number of Shorts to generate via Telegram, validates input format, and triggers the Shorts generation process with Klap.

- **1.2 Shorts Generation and Status Monitoring:** Submits the video to Klap for Shorts creation, polls for processing completion, retrieves generated Shorts metadata and export URLs, and verifies readiness before scheduling.

- **1.3 Scheduling and Multi-Platform Publishing:** Loads publishing schedules from Google Sheets, calculates publication times for each Short, logs scheduling data back to Google Sheets, uploads clips to Blotato, and publishes them to Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Bluesky, and Pinterest. Finally, it sends a Telegram summary notification.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block listens for user input via Telegram containing a YouTube URL and the desired number of Shorts (1-10). It validates this input format and initiates the Shorts generation request.

**Nodes Involved:**  
- Trigger: Receive YouTube URL via Telegram  
- Extract YouTube URL & Number of Shorts  
- Send Video to Klap for Shorts Generation

**Node Details:**

- **Trigger: Receive YouTube URL via Telegram**  
  - Type: Telegram Trigger  
  - Role: Webhook to receive Telegram messages  
  - Configuration: Listens to "message" updates, uses configured Telegram API credentials  
  - Inputs: Incoming Telegram messages  
  - Outputs: Raw JSON message for processing  
  - Failures: Telegram API connectivity or credential issues  

- **Extract YouTube URL & Number of Shorts**  
  - Type: Code (JavaScript)  
  - Role: Parses the message text, extracts YouTube URL and number of Shorts, validates the number (1-10)  
  - Key Expressions: Regex to detect URLs and numeric value, conditional logic to validate input  
  - Inputs: Telegram message JSON  
  - Outputs: JSON with action flags (`process`, `invalid_number`, `invalid_format`) and extracted data  
  - Failures: Malformed messages, missing or invalid inputs handled gracefully with error messages  
  - Edge Cases: User sends invalid number or incorrect format; returns user-friendly feedback  
 
- **Send Video to Klap for Shorts Generation**  
  - Type: HTTP Request  
  - Role: Sends POST request to Klap API to initiate Shorts generation with editing options enabled (captions, emojis, reframe, silence removal, intro title)  
  - Configuration:  
    - URL: `https://api.klap.app/v2/tasks/video-to-shorts`  
    - Headers: Authorization key required  
    - Body: JSON includes source video URL, target clip count, duration settings, and editing options  
  - Inputs: Validated YouTube URL and number of Shorts  
  - Outputs: Klap task ID and metadata for processing status checks  
  - Failures: API key invalid, rate limits, network errors, malformed requests  

---

#### 2.2 Shorts Generation and Status Monitoring

**Overview:**  
This block polls Klap API to check if Shorts generation is complete, lists generated clip ideas, exports HD Shorts, fetches final URLs, and verifies that all Shorts are ready before proceeding.

**Nodes Involved:**  
- Check Shorts Generation Status  
- Is Video Processing Complete? (If)  
- Wait Before Checking Status Again (Wait)  
- List Generated Clip Ideas  
- Export HD Short from Klap  
- Fetch Final Shorts URLs  
- Are All Shorts Ready? (Code)  
- Ready to Schedule Shorts? (If)  
- Wait Before Rechecking Final Shorts (Wait)  

**Node Details:**

- **Check Shorts Generation Status**  
  - Type: HTTP Request  
  - Role: GET Klap task status using task ID  
  - Configuration: URL built dynamically with task ID, Authorization header  
  - Inputs: Klap task ID from previous node  
  - Outputs: Task status JSON (`ready`, `processing`, etc.)  
  - Failures: API errors, task not found, auth failures  

- **Is Video Processing Complete?**  
  - Type: If  
  - Role: Checks if Klap status equals "ready"  
  - Inputs: Status JSON  
  - Outputs: Branches workflow to either fetch clips or wait and retry  
  - Failures: Logic errors rare  

- **Wait Before Checking Status Again**  
  - Type: Wait  
  - Role: Delays next status check by 60 seconds to avoid spamming API  
  - Inputs: Not conditional; triggered if processing not complete  
  - Outputs: Triggers status check again  
  - Failures: Workflow timeout if stuck  

- **List Generated Clip Ideas**  
  - Type: HTTP Request  
  - Role: Retrieves generated clip metadata from Klap project ID  
  - Inputs: Klap output project ID  
  - Outputs: List of clip ideas with metadata (name, virality score, captions)  
  - Failures: API failures, invalid IDs  

- **Export HD Short from Klap**  
  - Type: HTTP Request  
  - Role: Requests export for each clip ID in HD  
  - Inputs: Project ID and clip ID from previous node  
  - Outputs: Export task IDs for fetching URLs  
  - Failures: API errors, export limitations  

- **Fetch Final Shorts URLs**  
  - Type: HTTP Request  
  - Role: Retrieves download URLs for exported clips using export IDs  
  - Inputs: Export task IDs  
  - Outputs: Direct URLs to video files  
  - Failures: Timing issues if export not ready, retries expected  

- **Are All Shorts Ready?**  
  - Type: Code (JavaScript)  
  - Role: Aggregates all fetched exports and checks if all statuses are "ready"  
  - Inputs: All fetched clip statuses  
  - Outputs: Boolean flag `is_everything_ready` to gate scheduling  
  - Failures: Partial results, missing items  

- **Ready to Schedule Shorts?**  
  - Type: If  
  - Role: Branches workflow if all clips are ready or waits further  
  - Inputs: Boolean from previous node  
  - Outputs: Schedule block trigger or wait node  
  - Failures: Logic errors unlikely  

- **Wait Before Rechecking Final Shorts**  
  - Type: Wait  
  - Role: Delays 60 seconds before retrying to fetch final URLs  
  - Inputs: Triggered if not all clips ready  
  - Outputs: Loops back to fetch final URLs  
  - Failures: Potential infinite loop if clips never ready  

---

#### 2.3 Scheduling and Multi-Platform Publishing

**Overview:**  
This block manages scheduling parameters from Google Sheets, calculates publication times per Short, logs scheduling to Sheets, uploads videos to Blotato, and posts to multiple social media accounts.

**Nodes Involved:**  
- Fetch YouTube Video Title  
- Load Publishing Schedule from Google Sheets  
- Extract Scheduling Parameters (Posts/Day, Hours)  
- Fetch Already Scheduled Shorts  
- Determine Last Scheduled Time (Code)  
- Calculate Publication Times for New Shorts (Code)  
- Convert Times to Local Timezone (Paris) (Code)  
- Log Shorts & Schedule Info to Google Sheets  
- Assign Social Media IDs  
- Upload Video to Blotato  
- INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWITTER, LINKEDIN, BLUESKY, PINTEREST (HTTP Requests)  
- Send Publication Summary to Telegram  

**Node Details:**

- **Fetch YouTube Video Title**  
  - Type: HTTP Request  
  - Role: Uses YouTube oEmbed API to get video title from URL  
  - Inputs: YouTube URL from earlier extraction  
  - Outputs: Video title JSON  
  - Failures: Invalid URL, YouTube API limits  

- **Load Publishing Schedule from Google Sheets**  
  - Type: Google Sheets  
  - Role: Reads scheduling settings (posts per day, start/end time) from sheet  
  - Inputs: None (triggered by video title fetch)  
  - Outputs: Scheduling data rows  
  - Failures: Google API auth errors, missing sheet data  

- **Extract Scheduling Parameters (Posts/Day, Hours)**  
  - Type: Code (JavaScript)  
  - Role: Parses sheet rows for "Post per Day", "Start Time", "End Time"  
  - Inputs: Sheet rows  
  - Outputs: JSON object with scheduling parameters  
  - Failures: Missing or malformed sheet values  

- **Fetch Already Scheduled Shorts**  
  - Type: Google Sheets  
  - Role: Reads previously scheduled Shorts for time calculations  
  - Inputs: None  
  - Outputs: List of scheduled Shorts with timestamps  
  - Failures: Sheet read errors  

- **Determine Last Scheduled Time**  
  - Type: Code (JavaScript)  
  - Role: Finds latest scheduled post time from existing Shorts to avoid overlaps  
  - Inputs: Scheduled Shorts data  
  - Outputs: Last scheduled time ISO string and scheduling parameters  
  - Failures: Empty data fallback handled  

- **Calculate Publication Times for New Shorts**  
  - Type: Code (JavaScript)  
  - Role: Calculates scheduled publication times for each new Short based on posts per day, start/end hour constraints, and randomness ±30 min  
  - Inputs: Scheduling parameters, last scheduled time, Shorts metadata, final Shorts URLs  
  - Outputs: Array of Shorts with scheduled_at timestamps and metadata  
  - Failures: Date calculation errors, empty inputs  

- **Convert Times to Local Timezone (Paris)**  
  - Type: Code (JavaScript)  
  - Role: Converts UTC scheduled times to Europe/Paris timezone strings for user-friendly display  
  - Inputs: Scheduled Shorts with UTC times  
  - Outputs: Same Shorts with added `local_time` field  
  - Failures: Timezone parsing errors unlikely  

- **Log Shorts & Schedule Info to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends Shorts metadata and schedule info (including local time) to a log sheet for record keeping  
  - Inputs: Scheduled Shorts with metadata and local time  
  - Outputs: Confirmation of append operation  
  - Failures: Write permission or API errors  

- **Assign Social Media IDs**  
  - Type: Set  
  - Role: Defines social media platform account IDs and page/board IDs as fixed JSON object for downstream posting  
  - Inputs: None  
  - Outputs: JSON with platform IDs  
  - Failures: Misconfigured IDs will cause posting failures  

- **Upload Video to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads each Short video URL to Blotato media endpoint to prepare for posting  
  - Inputs: Clips URLs from Google Sheets log  
  - Outputs: Blotato media upload responses containing media URLs  
  - Failures: API key errors, upload failures  

- **Social Media Posting Nodes (INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWITTER, LINKEDIN, BLUESKY, PINTEREST)**  
  - Type: HTTP Request (one per platform)  
  - Role: Posts the uploaded media with captions and scheduled times to respective social media accounts via Blotato API  
  - Inputs: Media URLs, captions, scheduled time, account IDs from Assign Social Media IDs node and Google Sheets data  
  - Outputs: Posting confirmation JSONs  
  - Failures: Authentication errors, API rate limits, platform-specific restrictions, invalid account IDs  
  - Notes: Each node uses Blotato API key in header; JSON body customized per platform requirements  

- **Send Publication Summary to Telegram**  
  - Type: Telegram  
  - Role: Sends summary message to user with video title, number of Shorts, platforms, and scheduled times in local timezone  
  - Inputs: Video title, number of shorts, local scheduled times, chat ID from Telegram trigger  
  - Outputs: Telegram message sent confirmation  
  - Failures: Telegram API errors, invalid chat ID  

---

### 3. Summary Table

| Node Name                                | Node Type              | Functional Role                                        | Input Node(s)                               | Output Node(s)                                                                                                     | Sticky Note                                                                                              |
|-----------------------------------------|------------------------|-------------------------------------------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Trigger: Receive YouTube URL via Telegram | Telegram Trigger       | Receive YouTube URL and number of Shorts via Telegram | -                                           | Extract YouTube URL & Number of Shorts                                                                            |                                                                                                         |
| Extract YouTube URL & Number of Shorts  | Code                   | Parse and validate input message                       | Trigger: Receive YouTube URL via Telegram   | Send Video to Klap for Shorts Generation                                                                           |                                                                                                         |
| Send Video to Klap for Shorts Generation | HTTP Request           | Initiate Shorts generation with Klap API               | Extract YouTube URL & Number of Shorts      | Check Shorts Generation Status                                                                                      |                                                                                                         |
| Check Shorts Generation Status           | HTTP Request           | Poll Klap API for task status                           | Send Video to Klap for Shorts Generation    | Is Video Processing Complete?                                                                                       |                                                                                                         |
| Is Video Processing Complete?            | If                     | Branch if Shorts generation complete                    | Check Shorts Generation Status               | List Generated Clip Ideas, Wait Before Checking Status Again                                                       |                                                                                                         |
| Wait Before Checking Status Again        | Wait                   | Delay before polling Klap again                         | Is Video Processing Complete? (no branch)  | Check Shorts Generation Status                                                                                      |                                                                                                         |
| List Generated Clip Ideas                | HTTP Request           | Retrieve generated clip metadata                        | Is Video Processing Complete? (yes branch) | Export HD Short from Klap                                                                                           |                                                                                                         |
| Export HD Short from Klap                | HTTP Request           | Request HD export for each clip                         | List Generated Clip Ideas                    | Fetch Final Shorts URLs                                                                                            |                                                                                                         |
| Fetch Final Shorts URLs                  | HTTP Request           | Retrieve final video URLs                               | Export HD Short from Klap                    | Are All Shorts Ready?                                                                                               |                                                                                                         |
| Are All Shorts Ready?                    | Code                   | Verify if all Shorts exports are ready                 | Fetch Final Shorts URLs                       | Ready to Schedule Shorts?                                                                                           |                                                                                                         |
| Ready to Schedule Shorts?                | If                     | Branch to scheduling or wait                            | Are All Shorts Ready?                        | Fetch YouTube Video Title, Wait Before Rechecking Final Shorts                                                    |                                                                                                         |
| Wait Before Rechecking Final Shorts      | Wait                   | Delay before retrying fetch of final Shorts URLs       | Ready to Schedule Shorts? (no branch)       | Fetch Final Shorts URLs                                                                                            |                                                                                                         |
| Fetch YouTube Video Title                | HTTP Request           | Get video title using YouTube oEmbed                    | Ready to Schedule Shorts? (yes branch)      | Load Publishing Schedule from Google Sheets                                                                        |                                                                                                         |
| Load Publishing Schedule from Google Sheets | Google Sheets        | Load post scheduling settings                           | Fetch YouTube Video Title                    | Extract Scheduling Parameters (Posts/Day, Hours)                                                                    |                                                                                                         |
| Extract Scheduling Parameters (Posts/Day, Hours) | Code                | Parse scheduling parameters                             | Load Publishing Schedule from Google Sheets | Fetch Already Scheduled Shorts                                                                                      |                                                                                                         |
| Fetch Already Scheduled Shorts           | Google Sheets          | Load already scheduled Shorts data                      | Extract Scheduling Parameters                | Determine Last Scheduled Time                                                                                       |                                                                                                         |
| Determine Last Scheduled Time            | Code                   | Find last scheduled post time                           | Fetch Already Scheduled Shorts               | Calculate Publication Times for New Shorts                                                                          |                                                                                                         |
| Calculate Publication Times for New Shorts | Code                 | Compute scheduled times for new Shorts                  | Determine Last Scheduled Time                | Convert Times to Local Timezone (Paris)                                                                             |                                                                                                         |
| Convert Times to Local Timezone (Paris) | Code                   | Convert UTC times to local Paris time                   | Calculate Publication Times for New Shorts  | Log Shorts & Schedule Info to Google Sheets                                                                         |                                                                                                         |
| Log Shorts & Schedule Info to Google Sheets | Google Sheets        | Append Shorts and schedule info to log                  | Convert Times to Local Timezone (Paris)     | Assign Social Media IDs                                                                                              |                                                                                                         |
| Assign Social Media IDs                  | Set                    | Define social media account IDs                         | Log Shorts & Schedule Info to Google Sheets | Upload Video to Blotato                                                                                              |                                                                                                         |
| Upload Video to Blotato                  | HTTP Request           | Upload Shorts video to Blotato media                    | Assign Social Media IDs                       | INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWITTER, LINKEDIN, BLUESKY, PINTEREST, Send Publication Summary to Telegram |                                                                                                         |
| INSTAGRAM                              | HTTP Request           | Post video to Instagram via Blotato                     | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| YOUTUBE                                | HTTP Request           | Post video to YouTube Shorts via Blotato                | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| TIKTOK                                | HTTP Request           | Post video to TikTok via Blotato                         | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| FACEBOOK                              | HTTP Request           | Post video to Facebook via Blotato                       | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| THREADS                               | HTTP Request           | Post video to Threads via Blotato                        | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| TWITTER                               | HTTP Request           | Post video to Twitter via Blotato                        | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| LINKEDIN                              | HTTP Request           | Post video to LinkedIn via Blotato                       | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| BLUESKY                               | HTTP Request           | Post video to Bluesky via Blotato                        | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| PINTEREST                             | HTTP Request           | Post video to Pinterest via Blotato                      | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| Send Publication Summary to Telegram   | Telegram               | Notify user with publishing summary                      | Upload Video to Blotato                       | -                                                                                                                   |                                                                                                         |
| Sticky Note                            | Sticky Note            | Step 1 — Convert YouTube Video to Shorts                | -                                           | -                                                                                                                   | # ✅ Step 1 — Convert YouTube Video to Shorts                                                           |
| Sticky Note1                           | Sticky Note            | Step 2 — Schedule Shorts for Publication                 | -                                           | -                                                                                                                   | # ✅ Step 2 — Schedule Shorts for Publication                                                            |
| Sticky Note2                           | Sticky Note            | Step 3 — Publish Shorts to Social Media with Blotato    | -                                           | -                                                                                                                   | # ✅ Step 3 — Publish Shorts to Social Media with Blotato                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Name: "Trigger: Receive YouTube URL via Telegram"  
   - Configure webhook for Telegram message updates  
   - Set Telegram API credentials  

2. **Add a Code node**  
   - Name: "Extract YouTube URL & Number of Shorts"  
   - JavaScript to parse Telegram message text, extract URL and number (1-10), validate input, return structured JSON with action flags  

3. **Add HTTP Request node to send video to Klap**  
   - Name: "Send Video to Klap for Shorts Generation"  
   - Method: POST  
   - URL: `https://api.klap.app/v2/tasks/video-to-shorts`  
   - Headers: Authorization with Klap API key  
   - Body JSON: Include source_video_url, language "en", target_clip_count and max_clip_count equal to extracted number, editing options enabled, min/max/target durations set (15/60/30)  

4. **Add HTTP Request node to check Klap task status**  
   - Name: "Check Shorts Generation Status"  
   - Method: GET  
   - URL: `https://api.klap.app/v2/tasks/{{ task_id }}` (dynamic)  
   - Headers: Authorization with Klap API key  

5. **Add If node to check if status equals "ready"**  
   - Name: "Is Video Processing Complete?"  
   - Condition: `$json.status === 'ready'`  

6. **Add Wait node**  
   - Name: "Wait Before Checking Status Again"  
   - Duration: 60 seconds  

7. **Link "No" branch of If node to Wait node, and from Wait node back to status check node** (loop until ready)  

8. **Add HTTP Request node to list generated clip ideas**  
   - Name: "List Generated Clip Ideas"  
   - Method: GET  
   - URL: `https://api.klap.app/v2/projects/{{ output_id }}`  
   - Headers: Authorization with Klap API key  

9. **Add HTTP Request node to export HD Short**  
   - Name: "Export HD Short from Klap"  
   - Method: POST  
   - URL: `https://api.klap.app/v2/projects/{{ output_id }}/{{ clip_id }}/exports`  
   - Headers: Authorization with Klap API key  

10. **Add HTTP Request node to fetch final Shorts URLs**  
    - Name: "Fetch Final Shorts URLs"  
    - Method: GET  
    - URL: `https://api.klap.app/v2/projects/{{ output_id }}/{{ clip_id }}/exports/{{ export_id }}`  
    - Headers: Authorization with Klap API key  

11. **Add Code node to check if all Shorts are ready**  
    - Name: "Are All Shorts Ready?"  
    - JavaScript: Verify all exports have status "ready" and set flag `is_everything_ready`  

12. **Add If node "Ready to Schedule Shorts?"**  
    - Condition: `$json.is_everything_ready === true`  

13. **Add Wait node "Wait Before Rechecking Final Shorts"**  
    - Duration: 60 seconds, loop if not ready  

14. **Add HTTP Request node to fetch YouTube video title via oEmbed**  
    - Name: "Fetch YouTube Video Title"  
    - URL: `https://www.youtube.com/oembed?url={{ YouTube URL }}&format=json`  

15. **Add Google Sheets node to load publishing schedule**  
    - Name: "Load Publishing Schedule from Google Sheets"  
    - Configure with Google Sheets OAuth2 credentials, specify document and sheet ID  

16. **Add Code node to extract scheduling parameters (posts/day, start/end hours)**  
    - Name: "Extract Scheduling Parameters (Posts/Day, Hours)"  
    - JavaScript: Parse rows to find relevant settings  

17. **Add Google Sheets node to fetch already scheduled Shorts**  

18. **Add Code node "Determine Last Scheduled Time"**  
    - Find latest scheduled time from fetched Shorts  

19. **Add Code node "Calculate Publication Times for New Shorts"**  
    - Calculate scheduled_at timestamps for each new Short with randomness and daily limits  

20. **Add Code node "Convert Times to Local Timezone (Paris)"**  

21. **Add Google Sheets node "Log Shorts & Schedule Info to Google Sheets"**  
    - Append new scheduled Shorts with metadata  

22. **Add Set node "Assign Social Media IDs"**  
    - Hardcode social media platform account IDs and page/board IDs as JSON  

23. **Add HTTP Request node "Upload Video to Blotato"**  
    - POST to `https://backend.blotato.com/v2/media` with video URL, header containing Blotato API key  

24. **Add multiple HTTP Request nodes for each social platform (Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Bluesky, Pinterest)**  
    - POST to `https://backend.blotato.com/v2/posts`  
    - Include appropriate accountId, targetType, content text and mediaUrls, and scheduledTime  
    - Use Blotato API key in headers  
    - Use dynamic expressions to pull captions, media URLs, and schedule times  

25. **Add Telegram node "Send Publication Summary to Telegram"**  
    - Send message with video title, number of shorts, platforms, and scheduled times in local timezone  
    - Use Telegram credentials, chat ID from initial trigger  

26. **Connect all nodes according to dependencies and logic described.**  
    - Looping logic for polling Klap status and final URLs  
    - Branching logic for input validation and scheduling readiness  

27. **Credentials Setup:**  
    - Telegram API credentials for triggers and messages  
    - Klap API key for Shorts generation and management  
    - Google Sheets OAuth2 credentials for reading/writing scheduling data  
    - Blotato API key for uploading videos and posting to social media  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                      |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Step 1 — Convert YouTube Video to Shorts                                                          | Sticky Note in workflow for initial conversion step |
| Step 2 — Schedule Shorts for Publication                                                          | Sticky Note in workflow for scheduling block        |
| Step 3 — Publish Shorts to Social Media with Blotato                                              | Sticky Note in workflow for publishing block        |
| Blotato API documentation for multi-platform posting: https://docs.blotato.com                     | Referenced for HTTP requests to Blotato endpoints   |
| Klap API documentation: https://docs.klap.app                                                       | Used for Shorts generation and export management     |
| YouTube oEmbed API for video metadata retrieval: https://oembed.com                              | Used to fetch video title                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.