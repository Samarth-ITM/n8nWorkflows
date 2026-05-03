Telegram to Google Drive: Auto Upload & Track Videos with Gemini AI Assistant

https://n8nworkflows.xyz/workflows/telegram-to-google-drive--auto-upload---track-videos-with-gemini-ai-assistant-10713


# Telegram to Google Drive: Auto Upload & Track Videos with Gemini AI Assistant

### 1. Workflow Overview

This workflow automates the process of receiving videos via Telegram, uploading them to Google Drive, tracking metadata in Google Sheets, and providing intelligent AI-driven interactions using Google Gemini AI. It is designed primarily for creators, educators, and teams who want a seamless Telegram-to-Google Drive video management system with smart renaming, real-time logging, and conversational AI support.

The workflow is logically divided into four blocks:

- **1.1 Trigger & Routing:** Receives incoming Telegram messages/videos and routes them based on content type or commands.
- **1.2 Video Upload Process:** Downloads Telegram videos, uploads them to Google Drive, logs metadata to Google Sheets, and notifies users.
- **1.3 File Rename Process:** Parses rename commands, updates Google Drive file names, updates Google Sheets records, and confirms changes via Telegram.
- **1.4 AI Auto-Reply System:** Handles all other text messages through an AI agent that integrates Google Gemini language model and Google Sheets data for intelligent replies and memory retention.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Routing

**Overview:**  
This block listens for all incoming Telegram messages and routes them into three paths: video uploads, rename commands, or general text messages for AI handling.

**Nodes Involved:**  
- Telegram Trigger  
- Switch1

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger (webhook)  
  - Configuration: Listens to "message" updates from Telegram bot.  
  - Input: Telegram message webhook  
  - Output: Passes incoming message JSON to Switch1  
  - Edge Cases: Telegram API downtime, webhook misconfiguration, invalid bot token.

- **Switch1**  
  - Type: Switch node for routing  
  - Configuration: Routes based on three conditions:
    1. Message contains a video (`message.video` exists) → video upload flow  
    2. Message text starts with `/edit` → rename file flow  
    3. Any message text exists → AI response flow  
  - Input: Telegram message JSON  
  - Output: Routes to "Get a file1" (video), "Code1" (edit command), or "AI Agent1" (AI text response)  
  - Edge Cases: Messages missing expected fields, unexpected message types, malformed commands.

---

#### 1.2 Video Upload Process

**Overview:**  
When a video is detected, this block downloads the video from Telegram, uploads it to a specific Google Drive folder, logs metadata into Google Sheets, and sends confirmation back to the user.

**Nodes Involved:**  
- Get a file1  
- Upload file1  
- Append row in sheet1  
- Send a text message3

**Node Details:**  

- **Get a file1**  
  - Type: Telegram node (file download)  
  - Configuration: Downloads video file using `file_id` from incoming message  
  - Credentials: Telegram API credentials  
  - Input: Routed from Switch1 on video message  
  - Output: Passes downloaded file data to "Upload file1"  
  - Edge Cases: File not found, Telegram API limits, network errors.

- **Upload file1**  
  - Type: Google Drive node (upload)  
  - Configuration: Uploads file to folder with ID `12QmhGydAPjYma1dLb40J6yD8V4azR8ok` in "My Drive"  
  - Credentials: Google Drive OAuth2  
  - Input: Receives file binary from "Get a file1"  
  - Output: Outputs uploaded file metadata to "Append row in sheet1"  
  - Edge Cases: Insufficient Drive permissions, folder not found, upload timeout.

- **Append row in sheet1**  
  - Type: Google Sheets node (append row)  
  - Configuration: Appends a new row to sheet ID `2144684696` in the spreadsheet `1WcP7wqEPZZlL9mor8enGIP2yJXZ7GfFwD0bzYFKk-Vg`  
  - Columns: File size (MB), File ID, Duration, File name, Drive link, Last update timestamp, Upload time (localized Asia/Jakarta)  
  - Credentials: Google Sheets OAuth2  
  - Input: Receives file metadata from "Upload file1"  
  - Output: Passes to "Send a text message3"  
  - Edge Cases: Permissions denied, sheet unavailable, data format issues.

- **Send a text message3**  
  - Type: Telegram node (send message)  
  - Configuration: Sends confirmation message with rename command template containing the file ID  
  - Credentials: Telegram API  
  - Input: Triggered after row append  
  - Output: Ends upload flow  
  - Edge Cases: Telegram API limits, chat ID issues.

---

#### 1.3 File Rename Process

**Overview:**  
This block processes `/edit` commands to rename files in Google Drive and update corresponding Google Sheets records, then notifies the user.

**Nodes Involved:**  
- Code1  
- Update file1  
- Update row in sheet1  
- Send a text message4

**Node Details:**  

- **Code1**  
  - Type: Code node (JavaScript)  
  - Configuration: Parses the `/edit ID new_name` command from the message text into `command`, `id`, and `name` variables  
  - Input: Routed from Switch1 when `/edit` command detected  
  - Output: Passes parsed data to "Update file1"  
  - Edge Cases: Malformed commands, missing parameters, empty new name.

- **Update file1**  
  - Type: Google Drive node (update file)  
  - Configuration: Updates file name on Google Drive using file ID and new name from Code1  
  - Credentials: Google Drive OAuth2  
  - Input: Receives parsed parameters from Code1  
  - Output: Passes updated file metadata to "Update row in sheet1"  
  - Edge Cases: File not found, permission denied, invalid file ID.

- **Update row in sheet1**  
  - Type: Google Sheets node (update row)  
  - Configuration: Updates row matching file ID in the same sheet as upload process, modifying file name and last update timestamp  
  - Credentials: Google Sheets OAuth2  
  - Input: Receives updated file info from "Update file1"  
  - Output: Passes to "Send a text message4"  
  - Edge Cases: Row not found, sheet permission errors.

- **Send a text message4**  
  - Type: Telegram node (send message)  
  - Configuration: Sends confirmation message with the new file name back to the user  
  - Credentials: Telegram API  
  - Input: Triggered after sheet update  
  - Output: Ends rename flow  
  - Edge Cases: Telegram API errors.

---

#### 1.4 AI Auto-Reply System

**Overview:**  
Handles all text messages that are not videos or rename commands using an AI agent integrated with Google Gemini, Google Sheets data, and conversation memory for intelligent, emoji-enhanced responses.

**Nodes Involved:**  
- AI Agent1  
- Google Gemini Chat Model1  
- Simple Memory1  
- Data Vidio1  
- Send a text message5

**Node Details:**  

- **AI Agent1**  
  - Type: Langchain AI Agent node  
  - Configuration: Uses message text as input, system prompt defines AI as personal assistant with access to "Data Vidio" tool (Google Sheets) and memory  
  - Input: Routed from Switch1 for general text messages  
  - Outputs:
    - Main: AI response text to "Send a text message5"  
    - ai_languageModel: connected to "Google Gemini Chat Model1"  
    - ai_memory: connected to "Simple Memory1"  
    - ai_tool: connected to "Data Vidio1"  
  - Edge Cases: API limits, malformed input, memory persistence issues.

- **Google Gemini Chat Model1**  
  - Type: Langchain Google Gemini LLM node  
  - Configuration: No special options, uses Google PaLM API credentials  
  - Input: Receives prompt from AI Agent1  
  - Output: AI model response back to AI Agent1  
  - Edge Cases: API quota, latency, credentials issues.

- **Simple Memory1**  
  - Type: Langchain memory buffer window  
  - Configuration: Uses Telegram chat ID as session key to maintain per-user conversation context  
  - Input: Memory used by AI Agent1  
  - Output: Memory context to AI Agent1  
  - Edge Cases: Memory overflow, session key errors.

- **Data Vidio1**  
  - Type: Google Sheets Tool node  
  - Configuration: Provides AI Agent access to Google Sheets data (`Up Video Jouney Project` sheet) for querying video metadata  
  - Credentials: Google Sheets OAuth2  
  - Input: Used as AI tool by AI Agent1  
  - Output: Data responses fed back into AI Agent1  
  - Edge Cases: Sheet access issues, data inconsistency.

- **Send a text message5**  
  - Type: Telegram node (send message)  
  - Configuration: Sends AI-generated reply back to user chat  
  - Credentials: Telegram API  
  - Input: Receives AI output from AI Agent1  
  - Output: Ends AI reply flow  
  - Edge Cases: Telegram API errors, chat ID mismatches.

---

### 3. Summary Table

| Node Name           | Node Type                   | Functional Role                             | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                          |
|---------------------|-----------------------------|---------------------------------------------|-------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger     | telegramTrigger             | Entry point; receives Telegram messages     | -                       | Switch1                  | ## 1. Trigger & Routing: Detects incoming message/video and routes accordingly                                                        |
| Switch1             | switch                     | Routes flow by message type/command          | Telegram Trigger         | Get a file1, Code1, AI Agent1 | ## 1. Trigger & Routing: Routes video upload, rename command, or AI text response                                                     |
| Get a file1          | telegram                   | Downloads video file from Telegram            | Switch1 (video path)     | Upload file1             | ## 2. Video Upload Process: Retrieves video file from Telegram                                                                        |
| Upload file1         | googleDrive                | Uploads video to Google Drive                  | Get a file1              | Append row in sheet1      | ## 2. Video Upload Process: Uploads video to Drive folder                                                                             |
| Append row in sheet1 | googleSheets               | Logs video metadata to Google Sheets          | Upload file1             | Send a text message3      | ## 2. Video Upload Process: Logs metadata and timestamps                                                                              |
| Send a text message3 | telegram                   | Sends upload confirmation and rename command | Append row in sheet1     | -                        | ## 2. Video Upload Process: Confirms upload and provides rename command                                                              |
| Code1                | code                       | Parses `/edit ID new_name` command            | Switch1 (edit path)      | Update file1             | ## 3. File Rename Process: Parses rename command                                                                                      |
| Update file1         | googleDrive                | Renames file on Google Drive                   | Code1                    | Update row in sheet1      | ## 3. File Rename Process: Renames file on Drive                                                                                      |
| Update row in sheet1 | googleSheets               | Updates Google Sheets row with new file name  | Update file1             | Send a text message4      | ## 3. File Rename Process: Updates metadata in sheet                                                                                  |
| Send a text message4 | telegram                   | Sends rename confirmation to Telegram user    | Update row in sheet1     | -                        | ## 3. File Rename Process: Confirms rename                                                                                           |
| AI Agent1            | langchain.agent            | AI assistant handling general text messages   | Switch1 (text path)      | Send a text message5      | ## 4. AI Auto-Reply System: AI agent uses Gemini LLM, memory, and sheet data to reply                                                |
| Google Gemini Chat Model1 | langchain.lmChatGoogleGemini | Language model for AI Agent                    | AI Agent1 (ai_languageModel) | AI Agent1 (response)       | ## 4. AI Auto-Reply System: Uses Google Gemini PaLM API                                                                               |
| Simple Memory1       | langchain.memoryBufferWindow | Maintains per-user conversation memory        | -                       | AI Agent1 (ai_memory)     | ## 4. AI Auto-Reply System: Session memory keyed by Telegram chat ID                                                                 |
| Data Vidio1          | googleSheetsTool           | Provides sheet data as AI tool                 | -                       | AI Agent1 (ai_tool)       | ## 4. AI Auto-Reply System: Supplies video metadata for AI queries                                                                    |
| Send a text message5 | telegram                   | Sends AI-generated reply back to Telegram     | AI Agent1                 | -                        | ## 4. AI Auto-Reply System: Sends AI response                                                                                         |
| Sticky Note          | stickyNote                 | Documentation: Trigger & Routing overview      | -                       | -                        | ## 1. Trigger & Routing: Explains routing rules and paths                                                                             |
| Sticky Note1         | stickyNote                 | Documentation: Workflow overview and setup     | -                       | -                        | Overview of workflow purpose, use cases, and setup tips                                                                              |
| Sticky Note2         | stickyNote                 | Documentation: AI Auto-Reply System explanation | -                       | -                        | Explains AI Agent interaction with Gemini, memory, and Google Sheets                                                                |
| Sticky Note4         | stickyNote                 | Documentation: Video Upload Process explanation | -                       | -                        | Describes video upload workflow steps                                                                                                |
| Sticky Note5         | stickyNote                 | Documentation: File Rename Process explanation  | -                       | -                        | Describes rename command flow and usage                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates on your Telegram bot using your bot token credentials.

2. **Add Switch node (Switch1)**  
   - Add three conditions:
     - Condition 1: `message.video` exists (object exists) → route to video upload path.  
     - Condition 2: `message.text` starts with `/edit` → route to rename command path.  
     - Condition 3: `message.text` exists → route to AI text response path.

3. **Video Upload Path:**  
   - **Get a file1 (Telegram node):** Configure to download file using `file_id` from incoming video message.  
   - **Upload file1 (Google Drive node):**  
     - Credentials: Google Drive OAuth2  
     - Upload file to folder ID `12QmhGydAPjYma1dLb40J6yD8V4azR8ok` in "My Drive"  
     - Use the binary data from "Get a file1" node.  
   - **Append row in sheet1 (Google Sheets node):**  
     - Credentials: Google Sheets OAuth2  
     - Spreadsheet ID: `1WcP7wqEPZZlL9mor8enGIP2yJXZ7GfFwD0bzYFKk-Vg`  
     - Sheet ID: `2144684696`  
     - Columns to append:  
       - Ukuran (size in MB, formatted)  
       - ID File (Google Drive file ID)  
       - Duration (video duration from Telegram message)  
       - Nama File (file name)  
       - Link Drive (webViewLink from Drive upload)  
       - Last Update (current datetime Asia/Jakarta)  
       - Time Upload (Drive file created time, localized Asia/Jakarta)  
   - **Send a text message3 (Telegram node):**  
     - Send confirmation text including rename command template:  
       ```
       Vidio is in!
       Rename the file:
       `/edit <File ID> name`
       ```  
     - Use chat ID from original Telegram message.

4. **Rename Command Path:**  
   - **Code1 (Code node):**  
     - Parse text command `/edit <ID> <new_name>` into JSON with keys: `command`, `id`, `name`.  
   - **Update file1 (Google Drive node):**  
     - Credentials: Google Drive OAuth2  
     - Operation: Update file  
     - Use parsed `id` as file ID and `name` as new file name.  
   - **Update row in sheet1 (Google Sheets node):**  
     - Credentials: Google Sheets OAuth2  
     - Update row matching `ID File` with new `Nama File` and current `Last Update` timestamp.  
   - **Send a text message4 (Telegram node):**  
     - Send confirmation message with the new file name to the user.

5. **AI Auto-Reply Path:**  
   - **AI Agent1 (Langchain agent node):**  
     - Input: Use incoming message text.  
     - System prompt: Define AI as personal assistant with emoji use and Data Vidio tool access.  
   - **Google Gemini Chat Model1 (Langchain LLM node):**  
     - Credentials: Google PaLM API (Google Gemini)  
     - Connect to AI Agent1's language model input.  
   - **Simple Memory1 (Langchain memory node):**  
     - Use Telegram chat ID as session key for per-user memory.  
     - Connect to AI Agent1 memory input.  
   - **Data Vidio1 (Google Sheets Tool node):**  
     - Credentials: Google Sheets OAuth2  
     - Configure to read from same spreadsheet and sheet as uploads.  
     - Connect as AI tool input to AI Agent1.  
   - **Send a text message5 (Telegram node):**  
     - Send the AI-generated output back to the user chat.

6. **Connect nodes:**
   - Telegram Trigger → Switch1  
   - Switch1 video output → Get a file1 → Upload file1 → Append row in sheet1 → Send a text message3  
   - Switch1 `/edit` output → Code1 → Update file1 → Update row in sheet1 → Send a text message4  
   - Switch1 text output → AI Agent1 → Send a text message5  
   - Connect AI Agent1 to Google Gemini Chat Model1 (language model), Simple Memory1 (memory), and Data Vidio1 (AI tool).

7. **Credentials Setup:**
   - Telegram API credentials for trigger and sending messages.  
   - Google Drive OAuth2 with permission to upload and update files.  
   - Google Sheets OAuth2 with read and write access to the specified spreadsheet.  
   - Google PaLM API credentials for Google Gemini model.

8. **Testing:**
   - Test sending a video via Telegram to confirm upload and logging.  
   - Test `/edit <ID> <new_name>` commands to rename files and update logs.  
   - Test sending general text messages to verify AI responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                              | Context or Link                                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow automates Telegram-to-Google Drive video uploads with smart renaming, Google Sheets logging, and Gemini AI assistance. Ideal for creators, educators, and automation-focused teams.                                                          | Overview sticky note at workflow start.                                                                       |
| The AI Agent uses Google Gemini PaLM API with per-user memory and can query video metadata from Google Sheets ("Data Vidio") to provide rich, emoji-enhanced responses.                                                                                   | Sticky note describing AI Auto-Reply System.                                                                  |
| Video upload process includes file download from Telegram, upload to Drive folder "My Projects Journey", metadata logging, and user notification with rename command template.                                                                             | Sticky note describing video upload process.                                                                   |
| Rename process triggered by `/edit` commands parsed by JavaScript, updating both Drive filenames and corresponding Sheets rows, followed by Telegram confirmation.                                                                                         | Sticky note describing file rename process with command format example `/edit <FILE_ID> <new file name>`.       |
| Setup tips: Use provided Google Sheets template, configure Telegram bot token, Google Drive and Sheets credentials, insert Gemini API key, and test before production.                                                                                      | Workflow overview sticky note.                                                                                  |

---

**Disclaimer:** The provided content is exclusively from an automated workflow created with n8n, strictly respecting all applicable content policies and containing no illegal, offensive, or protected elements. All data handled is legal and public.