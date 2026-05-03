Download TikTok Videos Without Watermarks via Telegram Bot

https://n8nworkflows.xyz/workflows/download-tiktok-videos-without-watermarks-via-telegram-bot-10001


# Download TikTok Videos Without Watermarks via Telegram Bot

### 1. Workflow Overview

This workflow implements a Telegram bot that downloads TikTok videos without watermarks when users send TikTok video links via Telegram messages. It targets users who want a seamless, automated method to obtain clean TikTok video files directly from Telegram, without dealing with ads or watermarked content.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives and triggers on incoming Telegram messages.
- **1.2 URL Validation:** Checks if the message contains a valid TikTok URL.
- **1.3 User Feedback:** Sends real-time feedback messages and chat actions to inform the user about validation status and processing progress.
- **1.4 Video Processing:** Fetches the TikTok video page HTML, extracts the direct video URL (no watermark), downloads the video file, and deletes the intermediate "processing" notification.
- **1.5 Video Delivery:** Sends the downloaded video file back to the user with descriptive captions including video statistics.
- **1.6 Error Handling:** Catches and formats errors throughout the process, delivering user-friendly error messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Listens for incoming Telegram messages from users and triggers the workflow with their content.

**Nodes Involved:**  
- Telegram Trigger

**Node Details:**  
- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Listens to Telegram updates, specifically messages sent to the bot.  
  - Configuration: Receives "message" updates only, no additional filters.  
  - Credentials: Uses Telegram API credentials for the bot "Video Download Bot".  
  - Input: Telegram webhook call on message received.  
  - Output: Passes message JSON for further processing.  
  - Failure: Network issues or invalid webhook setup may cause missed triggers.  
  - Version: 1.2

---

#### 2.2 URL Validation

**Overview:**  
Validates that the incoming Telegram message contains a TikTok URL to avoid processing invalid inputs.

**Nodes Involved:**  
- Validate TikTok URL  
- Send Invalid URL Message

**Node Details:**  
- **Validate TikTok URL**  
  - Type: If node  
  - Role: Checks if the text message contains the substring "tiktok.com".  
  - Configuration: Case sensitive string contains check on `{{ $json.message.text }}`.  
  - Input: Message JSON from Telegram Trigger node.  
  - Output:  
    - True branch (valid URL) proceeds to processing.  
    - False branch sends an invalid URL message.  
  - Failure: Expression evaluation failure if message text is missing or malformed.  
  - Version: 2.2

- **Send Invalid URL Message**  
  - Type: Telegram node (send message)  
  - Role: Sends a user-friendly error message if the URL is invalid.  
  - Configuration: Custom error message with examples of valid TikTok URLs.  
  - Inputs: Receives false branch from Validate TikTok URL.  
  - Outputs: Ends workflow for invalid inputs.  
  - Failure: Telegram API errors (e.g., invalid chat ID).  
  - Version: 1.2

---

#### 2.3 User Feedback

**Overview:**  
Provides user feedback via Telegram chat actions and messages to indicate the bot is processing the request.

**Nodes Involved:**  
- Send Chat Action  
- Send Processing Message

**Node Details:**  
- **Send Chat Action**  
  - Type: Telegram node (sendChatAction)  
  - Role: Sends "upload_video" chat action to show the bot is working.  
  - Configuration: Uses chat ID from Telegram message sender.  
  - Input: True branch from Validate TikTok URL.  
  - Output: Triggers "Send Processing Message".  
  - Failure: Telegram API errors if chat ID is invalid or blocked.  
  - Version: 1.2

- **Send Processing Message**  
  - Type: Telegram node (send message)  
  - Role: Sends a "Downloading video... Please wait!" message.  
  - Configuration: Chat ID from Telegram Trigger node, disables attribution.  
  - Input: Output from Send Chat Action.  
  - Output: Triggers variable configuration node.  
  - Failure: Telegram API errors.  
  - Version: 1.2

---

#### 2.4 Video Processing

**Overview:**  
Fetches TikTok HTML page, extracts video URL, downloads the video file, and cleans up the "processing" notification.

**Nodes Involved:**  
- Configure Variables  
- Get TikTok Page HTML  
- Extract Video URL  
- Download Video File  
- Delete Process Notif

**Node Details:**  
- **Configure Variables**  
  - Type: Set node  
  - Role: Stores trimmed TikTok URL and chat ID for reuse.  
  - Configuration: Assigns `share_video_url` from trimmed message text and `chat_id` from message sender ID.  
  - Input: From Send Processing Message.  
  - Output: Goes to Get TikTok Page HTML.  
  - Failure: None expected; basic variable assignment.  
  - Version: 3.4

- **Get TikTok Page HTML**  
  - Type: HTTP Request node  
  - Role: Fetches the full HTML of the TikTok video page.  
  - Configuration:  
    - URL from the original message text.  
    - Timeout 30 seconds.  
    - Redirects enabled.  
    - Response as full text with headers.  
    - Custom headers mimicking a browser (User-Agent, Accept, Accept-Language).  
  - Input: From Configure Variables.  
  - Output: On success, passes HTML to Extract Video URL. On error, continues error output.  
  - Failure: Network errors, 404 or TikTok page unavailable, timeouts.  
  - Version: 4.2

- **Extract Video URL**  
  - Type: Code node (JavaScript)  
  - Role: Parses the TikTok HTML to extract the direct video URL and metadata.  
  - Configuration:  
    - Parses `<script id="__UNIVERSAL_DATA_FOR_REHYDRATION__" type="application/json">` JSON.  
    - Extracts video play address or download address.  
    - Collects video description, author, and statistics (plays, likes, comments).  
    - Throws errors if video data is missing or cannot parse JSON.  
  - Input: HTML from Get TikTok Page HTML.  
  - Output: On success, passes video info JSON to Download Video File. On error, routes to error handler.  
  - Failure: JSON parse errors, missing or changed TikTok page structure, private or deleted videos.  
  - Version: 2

- **Download Video File**  
  - Type: HTTP Request node  
  - Role: Downloads the actual video file using extracted URL and cookies.  
  - Configuration:  
    - URL from extracted video URL in JSON.  
    - Timeout 60 seconds.  
    - Response as file binary.  
    - Headers include User-Agent, Referer (TikTok homepage), Accept for video types, Cookies from extraction, Range header for partial content.  
    - Allows unauthorized certs.  
  - Input: Video info JSON from Extract Video URL.  
  - Output: On success, triggers Delete Process Notif and Send Video to User. On error, routes to error handler.  
  - Failure: Network errors, expired or restricted URL, invalid cookies.  
  - Version: 4.2

- **Delete Process Notif**  
  - Type: Telegram node (deleteMessage)  
  - Role: Deletes the "Downloading video..." message to keep chat clean.  
  - Configuration: Chat ID and message ID from Telegram Trigger and Send Processing Message nodes.  
  - Input: Success path from Download Video File.  
  - Output: Ends after deletion.  
  - Failure: If message already deleted or Telegram API errors.  
  - Version: 1.2

---

#### 2.5 Video Delivery

**Overview:**  
Sends the downloaded video file back to the user with a caption including author and video stats.

**Nodes Involved:**  
- Send Video to User

**Node Details:**  
- **Send Video to User**  
  - Type: Telegram node (sendVideo)  
  - Role: Sends the binary video file to the user's chat with formatted caption.  
  - Configuration:  
    - Chat ID from Telegram Trigger node.  
    - Uses binary data from Download Video File node.  
    - Caption includes a success emoji, author username, views, and likes formatted with thousands separators.  
    - Attribution disabled.  
  - Input: Success path from Download Video File.  
  - Output: Ends workflow.  
  - Failure: Telegram API errors (file size limits, chat blocked).  
  - Version: 1.2

---

#### 2.6 Error Handling

**Overview:**  
Centralizes error formatting and user notification for any processing failures.

**Nodes Involved:**  
- Format Error  
- Send Error Message

**Node Details:**  
- **Format Error**  
  - Type: Code node (JavaScript)  
  - Role: Extracts error message from input or defaults to a generic message, associates chat ID for reply.  
  - Configuration: Reads `$input.first().json.error` or `.message`.  
  - Input: Error outputs from Extract Video URL, Download Video File, Get TikTok Page HTML nodes.  
  - Output: Passes formatted error JSON to Send Error Message.  
  - Failure: Minimal risk; error formatting logic is simple.  
  - Version: 2

- **Send Error Message**  
  - Type: Telegram node (send message)  
  - Role: Sends a detailed error message describing possible reasons and encouraging retry.  
  - Configuration:  
    - Message includes error text and suggestions like deleted video, invalid link, maintenance, or restrictions.  
    - Chat ID dynamically assigned from error formatting node.  
  - Input: From Format Error node.  
  - Output: Ends workflow.  
  - Failure: Telegram API errors if chat ID invalid.  
  - Version: 1.2

---

### 3. Summary Table

| Node Name                | Node Type            | Functional Role                          | Input Node(s)            | Output Node(s)                        | Sticky Note                                                                                       |
|--------------------------|----------------------|----------------------------------------|--------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Trigger         | Telegram Trigger     | Receives Telegram messages              | -                        | Validate TikTok URL                  | ## 📥 TRIGGER Receives messages from Telegram users                                              |
| Validate TikTok URL      | If                   | Validates TikTok URL in message         | Telegram Trigger         | Send Chat Action (true), Send Invalid URL Message (false) | ## ✅ VALIDATION Checks if the message contains a valid TikTok URL                               |
| Send Invalid URL Message | Telegram             | Sends error for invalid TikTok URLs     | Validate TikTok URL (false) | -                                   | ## ✅ VALIDATION Checks if the message contains a valid TikTok URL                               |
| Send Chat Action         | Telegram             | Sends "upload_video" chat action        | Validate TikTok URL (true) | Send Processing Message             | ## 💬 USER FEEDBACK Sends status messages to keep user informed                                 |
| Send Processing Message  | Telegram             | Sends "Downloading video... Please wait" | Send Chat Action         | Configure Variables                 | ## 💬 USER FEEDBACK Sends status messages to keep user informed                                 |
| Configure Variables      | Set                  | Stores chat ID and TikTok URL           | Send Processing Message  | Get TikTok Page HTML                | ## 🎬 VIDEO PROCESSING Fetches HTML → Extracts video URL → Downloads file                       |
| Get TikTok Page HTML     | HTTP Request         | Fetches TikTok video page HTML          | Configure Variables      | Extract Video URL (success), Format Error (failure) | ## 🎬 VIDEO PROCESSING Fetches HTML → Extracts video URL → Downloads file                       |
| Extract Video URL        | Code                 | Parses HTML to extract video info       | Get TikTok Page HTML     | Download Video File (success), Format Error (failure) | ## 🎬 VIDEO PROCESSING Fetches HTML → Extracts video URL → Downloads file                       |
| Download Video File      | HTTP Request         | Downloads the TikTok video file          | Extract Video URL        | Delete Process Notif, Send Video to User (success), Format Error (failure) | ## 🎬 VIDEO PROCESSING Fetches HTML → Extracts video URL → Downloads file                       |
| Delete Process Notif     | Telegram             | Deletes "processing" message            | Download Video File      | -                                   | ## 🎬 VIDEO PROCESSING Fetches HTML → Extracts video URL → Downloads file                       |
| Send Video to User       | Telegram             | Sends the downloaded video to user     | Download Video File      | -                                   | ## 🎬 VIDEO PROCESSING Fetches HTML → Extracts video URL → Downloads file                       |
| Format Error             | Code                 | Formats error messages for user display | Get TikTok Page HTML (fail), Extract Video URL (fail), Download Video File (fail) | Send Error Message                   | ## ⚠️ ERROR HANDLING Catches errors and sends user-friendly messages                            |
| Send Error Message       | Telegram             | Sends user-friendly error message       | Format Error             | -                                   | ## ⚠️ ERROR HANDLING Catches errors and sends user-friendly messages                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Enable updates for "message" only  
   - Credentials: Connect Telegram bot credentials (named "Video Download Bot")  
   - Position: Start of workflow

2. **Add Validate TikTok URL node (If node)**  
   - Condition: Check if `{{$json.message.text}}` contains "tiktok.com" (case sensitive)  
   - Input: Connect from Telegram Trigger output  
   - Output: True → continue processing; False → send invalid message

3. **Add Send Invalid URL Message node (Telegram)**  
   - Text: "❌ Please send a valid TikTok link! ..." with examples  
   - Chat ID: `={{ $json.message.from.id }}`  
   - Credentials: Use same Telegram bot credentials  
   - Connect from False output of Validate TikTok URL

4. **Add Send Chat Action node (Telegram)**  
   - Action: "upload_video"  
   - Chat ID: `={{ $json.message.from.id }}`  
   - Credentials: Telegram bot  
   - Connect from True output of Validate TikTok URL

5. **Add Send Processing Message node (Telegram)**  
   - Text: "⏳ Downloading video... Please wait!"  
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.from.id }}`  
   - Credentials: Telegram bot  
   - Connect from Send Chat Action output

6. **Add Configure Variables node (Set)**  
   - Assign:  
     - `share_video_url` = trimmed `{{$json.message.text}}`  
     - `chat_id` = `{{$json.message.from.id}}`  
   - Connect from Send Processing Message output

7. **Add Get TikTok Page HTML node (HTTP Request)**  
   - URL: `={{ $('Telegram Trigger').item.json.message.text }}`  
   - Timeout: 30000 ms  
   - Redirect: enabled  
   - Response format: Full response, text  
   - Headers:  
     - User-Agent: browser-like string  
     - Accept: "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8"  
     - Accept-Language: "en-US,en;q=0.9"  
   - Connect from Configure Variables output  
   - On error: continue to error handling

8. **Add Extract Video URL node (Code)**  
   - JavaScript code to parse HTML and extract JSON data from `<script id="__UNIVERSAL_DATA_FOR_REHYDRATION__" ...>`  
   - Extract video URL, description, author, stats (plays, likes, comments)  
   - Throw errors if parsing or extraction fails  
   - Input: Connect from success output of Get TikTok Page HTML  
   - On error: continue to error handling

9. **Add Download Video File node (HTTP Request)**  
   - URL: `={{ $json.videoUrl }}` (from extracted data)  
   - Timeout: 60000 ms  
   - Response format: file (binary)  
   - Headers:  
     - User-Agent: browser-like string  
     - Referer: "https://www.tiktok.com/"  
     - Accept: "video/mp4,video/webm,video/*;q=0.9,application/octet-stream;q=0.8"  
     - Accept-Language: "en-US,en;q=0.9"  
     - Connection: "keep-alive"  
     - Cookie: `={{ $json.cookies }}`  
     - Range: "bytes=0-"  
   - Allow unauthorized certificates: true  
   - Connect from Extract Video URL success output  
   - On error: continue to error handling

10. **Add Delete Process Notif node (Telegram)**  
    - Operation: deleteMessage  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.from.id }}`  
    - Message ID: `={{ $('Send Processing Message').item.json.result.message_id }}`  
    - Connect from Download Video File success output

11. **Add Send Video to User node (Telegram)**  
    - Operation: sendVideo  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.from.id }}`  
    - Binary data: enabled (use video file binary)  
    - Caption: Includes author, views, likes using expressions from Extract Video URL node  
    - Connect from Download Video File success output

12. **Add Format Error node (Code)**  
    - JavaScript code to extract error message from input and add chat ID from Configure Variables node  
    - Connect error outputs from Get TikTok Page HTML, Extract Video URL, Download Video File nodes

13. **Add Send Error Message node (Telegram)**  
    - Text: Formatted error message with possible reasons and retry suggestion  
    - Chat ID: from Format Error node JSON  
    - Connect from Format Error output

14. **Connect all error outputs properly**  
    - Each critical node with potential failure connects to Format Error node

15. **Add sticky notes (optional)** for documentation clarity to match logical blocks.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow author: Nguyễn Thiệu Toàn (n8n Creator)                                                          | https://nguyenthieutoan.com                         |
| Workflow turns Telegram bot into TikTok downloader without watermark, with video stats display             | Workflow description in sticky note                  |
| Workflow flow diagram and technical process detailed in sticky notes within the workflow                   | See nodes "Sticky Note1" and "Sticky Note2"          |
| Error handling provides clear, actionable messages to users to improve experience                          | Part of error handling block                          |
| Telegram bot credentials must be properly configured with bot token (Open Source bots via BotFather)       | Telegram API setup instructions                      |
| HTTP requests mimic browser headers to avoid TikTok blocking or bot detection                              | Important for stable video extraction                 |
| Video download uses cookies from TikTok HTML response to prevent access issues                             | Ensures video file can be fetched                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.