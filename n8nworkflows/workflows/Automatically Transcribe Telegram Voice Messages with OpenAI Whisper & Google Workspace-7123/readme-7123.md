Automatically Transcribe Telegram Voice Messages with OpenAI Whisper & Google Workspace

https://n8nworkflows.xyz/workflows/automatically-transcribe-telegram-voice-messages-with-openai-whisper---google-workspace-7123


# Automatically Transcribe Telegram Voice Messages with OpenAI Whisper & Google Workspace

### 1. Workflow Overview

This workflow, titled **"🎙️ VoiceScribe AI: Telegram Audio Message Auto Transcription with OpenAI Whisper"**, automates the transcription of Telegram voice messages using OpenAI Whisper, archives the audio files in Google Drive, logs transcription data in Google Sheets, and communicates results back to users on Telegram. It is designed primarily for professionals such as journalists, content creators, or anyone needing quick, searchable voice-to-text conversion.

The workflow is logically structured into these main blocks:

- **1.1 Input Reception:** Triggered by incoming Telegram messages via a Telegram Bot.
- **1.2 Message Type Validation:** Checks if the incoming message is a Telegram voice message (audio/ogg format).
- **1.3 Audio Processing and Transcription:** Downloads the voice message, transcribes it using OpenAI Whisper, and uploads the audio to Google Drive.
- **1.4 Data Merging and Transformation:** Combines transcription output with Google Drive metadata and formats it for structured logging.
- **1.5 Logging and User Notification:** Appends the transcript and metadata to a Google Sheet and sends a confirmation message with transcript and download link back to the Telegram user.
- **1.6 Unsupported Message Handling:** Responds politely to users who send non-audio messages.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for any new messages sent to the Telegram Bot and initiates the workflow.

- **Nodes Involved:**  
  - Telegram Voice Message Trigger

- **Node Details:**

  - **Telegram Voice Message Trigger**  
    - Type: *Telegram Trigger*  
    - Role: Entry point that listens for new Telegram messages with the update type "message".  
    - Configuration: Listens for all messages; downloads message content by default.  
    - Inputs: None (trigger node).  
    - Outputs: Passes message JSON including chat and message data.  
    - Version: 1.2  
    - Edge Cases: Network interruptions, webhook misconfigurations, Telegram API rate limits.

#### 1.2 Message Type Validation

- **Overview:**  
  Verifies if the incoming Telegram message is a voice/audio message (specifically of MIME type audio/ogg). Routes valid audio messages for transcription or non-audio messages to an unsupported message handler.

- **Nodes Involved:**  
  - Is audio message? (If node)  
  - Un-supported message type (Telegram node)

- **Node Details:**

  - **Is audio message?**  
    - Type: *If*  
    - Role: Checks if the message JSON string contains `"audio/ogg"`.  
    - Configuration: Uses string containment condition on serialized message JSON.  
    - Inputs: Telegram message JSON from trigger.  
    - Outputs: True branch (audio message), False branch (non-audio).  
    - Version: 2.2  
    - Edge Cases: Message format changes, new audio MIME types, false negatives if Telegram changes payload structure.

  - **Un-supported message type**  
    - Type: *Telegram* (Send Message)  
    - Role: Sends a polite reply informing users that only voice messages are accepted.  
    - Configuration: Sends fixed text message to the user's chat ID extracted from the Telegram Voice Message Trigger node.  
    - Inputs: Receives from the False branch of the If node.  
    - Outputs: None.  
    - Version: 1.2  
    - Edge Cases: Telegram API errors, chat ID missing or invalid, message sending failures.

#### 1.3 Audio Processing and Transcription

- **Overview:**  
  Downloads the audio file from Telegram, sends it to OpenAI Whisper for transcription, and uploads the original audio file to Google Drive for backup.

- **Nodes Involved:**  
  - Download audio message (Telegram)  
  - Transcribe a recording (OpenAI Whisper)  
  - Upload file (Google Drive)  
  - Merge (Merge node to combine outputs)

- **Node Details:**

  - **Download audio message**  
    - Type: *Telegram* (Get File)  
    - Role: Downloads voice message `.oga` file using Telegram file_id.  
    - Configuration: Uses `message.voice.file_id` from Telegram message JSON; downloads file binary.  
    - Inputs: True branch from "Is audio message?" node.  
    - Outputs: Binary audio file.  
    - Version: 1.2  
    - Edge Cases: File download failures, file_id missing, Telegram API rate limits.

  - **Transcribe a recording**  
    - Type: *OpenAI* (LangChain node)  
    - Role: Sends the audio file binary to OpenAI Whisper API for transcription.  
    - Configuration: Operation set to `transcribe` under the `audio` resource; uses OpenAI API credentials configured.  
    - Inputs: Audio binary from "Download audio message".  
    - Outputs: JSON transcript with text and usage metadata (e.g., duration).  
    - Version: 1.8  
    - Edge Cases: API authentication failures, transcription timeouts, unsupported audio formats.

  - **Upload file**  
    - Type: *Google Drive*  
    - Role: Uploads original audio file to a predefined folder in Google Drive for safekeeping.  
    - Configuration: Filename prefixed with "audio-" and timestamp; uploads to folder with ID `1ObNNVJFR2vcKqP8p-ZnX_eaZy4gBHgha` on "My Drive".  
    - Inputs: Audio binary from "Download audio message".  
    - Outputs: Google Drive file metadata including download link, creation time.  
    - Version: 3  
    - Edge Cases: Google OAuth token expiration, quota limits, folder access errors.

  - **Merge**  
    - Type: *Merge*  
    - Role: Combines output from "Transcribe a recording" and "Upload file" nodes to aggregate transcript and file metadata.  
    - Configuration: Default merge mode (likely 'append' by position).  
    - Inputs:  
      - Input 1: Transcription result  
      - Input 2: Google Drive upload metadata  
    - Outputs: Combined JSON data for further processing.  
    - Version: 3.2  
    - Edge Cases: Mismatched input lengths, missing data on either input.

#### 1.4 Data Merging and Transformation

- **Overview:**  
  Transforms the merged transcription and file metadata into a structured JSON object suitable for Google Sheets logging and Telegram messaging.

- **Nodes Involved:**  
  - Transform the output of voice record (Code node)

- **Node Details:**

  - **Transform the output of voice record**  
    - Type: *Code* (JavaScript)  
    - Role: Extracts and combines relevant data fields from transcription and Google Drive metadata into a single JSON object with keys: DateTime, Duration, Transcript, AudioURL, ChatID.  
    - Configuration:  
      - Reads inputs[0] as transcription data (fields: text, usage.seconds)  
      - Reads inputs[1] as Google Drive metadata (fields: createdTime, webContentLink)  
      - Extracts chat ID from Telegram trigger node.  
    - Inputs: From "Merge" node.  
    - Outputs: Single JSON object with combined data, ready for logging and messaging.  
    - Version: 2  
    - Edge Cases: Missing fields in inputs, null or undefined values, failures in cross-node referencing.

#### 1.5 Logging and User Notification

- **Overview:**  
  Logs the transcription record into Google Sheets and sends a formatted confirmation message back to the Telegram user with transcript details and audio download link.

- **Nodes Involved:**  
  - Log voice record to google sheet (Google Sheets)  
  - Inform user via Telegram (Telegram Send Message)

- **Node Details:**

  - **Log voice record to google sheet**  
    - Type: *Google Sheets*  
    - Role: Appends a new row to a Google Sheet with transcript metadata (DateTime, Duration, Transcript, AudioURL).  
    - Configuration:  
      - Document ID: Spreadsheet ID `1a-u0XHQWjn4VKbq5WpvSJy5_JgHuFl5A2Q2TEBDC5bI`  
      - Sheet: GID=0 (Sheet1)  
      - Auto-maps input JSON fields to columns.  
    - Inputs: Transformed JSON from the Code node.  
    - Outputs: None (append operation).  
    - Version: 4.6  
    - Edge Cases: OAuth token expiry, quota limits, spreadsheet access permissions.

  - **Inform user via Telegram**  
    - Type: *Telegram* (Send Message)  
    - Role: Sends a user-friendly message confirming transcription completion, including recording duration, date/time, and audio download URL.  
    - Configuration:  
      - Uses template text with expressions referencing `Duration`, `DateTime`, `AudioURL`, and `ChatID` from incoming JSON.  
    - Inputs: Transformed JSON from the Code node.  
    - Outputs: None.  
    - Version: 1.2  
    - Edge Cases: Telegram API failures, invalid chat ID, message content formatting errors.

#### 1.6 Unsupported Message Handling

- **Overview:**  
  Handles cases where the incoming Telegram message is not a voice message by sending a polite notification to the user.

- **Nodes Involved:**  
  - Un-supported message type (Telegram Send Message)

- **Node Details:**  
  (Already detailed in Block 1.2)

---

### 3. Summary Table

| Node Name                    | Node Type             | Functional Role                                | Input Node(s)               | Output Node(s)                          | Sticky Note                                                                                       |
|------------------------------|-----------------------|-----------------------------------------------|-----------------------------|-----------------------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Voice Message Trigger | Telegram Trigger      | Entry point: listens for Telegram messages    | None                        | Is audio message?                       | ### 1. 📩 Telegram Trigger  \nListens for incoming messages from the user via the connected Telegram bot. This is the entry point of the workflow. |
| Is audio message?             | If                    | Checks whether message is audio/ogg            | Telegram Voice Message Trigger | Download audio message (true), Un-supported message type (false) | ### 2. Is Audio Message?  \nChecks whether the incoming Telegram message is an audio message. If not, routes to unsupported message handler. |
| Download audio message        | Telegram (Get File)    | Downloads voice message audio from Telegram    | Is audio message? (true)     | Transcribe a recording, Upload file     | ### 3.1. Download audio message and transcript with OpenAI\nSend the binary audio to OpenAI Whisper or your transcription service. |
| Transcribe a recording        | OpenAI (LangChain)     | Sends audio to OpenAI Whisper for transcription | Download audio message       | Merge                                    | ### 3.1. Download audio message and transcript with OpenAI\nSend the binary audio to OpenAI Whisper or your transcription service. |
| Upload file                  | Google Drive           | Uploads audio file to Google Drive             | Download audio message       | Merge                                    | ### 3.2. Upload the original audio to drive for later usage\nBackup the original audio file.          |
| Merge                        | Merge                  | Combines transcription and upload metadata    | Transcribe a recording, Upload file | Transform the output of voice record     |                                                                                                 |
| Transform the output of voice record | Code (JavaScript)  | Formats merged data for Google Sheets and Telegram | Merge                       | Log voice record to google sheet, Inform user via Telegram | ### 4.2 Transform & log expense\n- Transform the Output to Audio Record  \n- Log Audio Record to Google Sheet  |
| Log voice record to google sheet | Google Sheets         | Appends transcript and metadata to Google Sheet | Transform the output of voice record | None                                   |                                                                                                 |
| Inform user via Telegram     | Telegram (Send Message) | Sends confirmation message to Telegram user   | Transform the output of voice record | None                                   | ### 4.1. Inform user via telegram\nSend friendly message to user with audio download URL              |
| Un-supported message type    | Telegram (Send Message) | Responds to non-audio messages with polite info | Is audio message? (false)    | None                                   | Polite message sent when message is not a voice note.                                             |
| Sticky Note                  | Sticky Note            | Documentation and visual notes                  | None                        | None                                   | # 🎙️ VoiceScribe AI: Telegram Audio Message Auto Transcription with OpenAI Whisper (full description included) |
| Sticky Note1                 | Sticky Note            | Screenshot image                                | None                        | None                                   | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-07+at+2.19.15%E2%80%AFPM.png "Optional title") |
| Sticky Note2                 | Sticky Note            | Screenshot image                                | None                        | None                                   | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-07+at+1.54.03%E2%80%AFPM.png "Optional title") |
| Sticky Note5                 | Sticky Note            | Describes Telegram Trigger node                 | None                        | None                                   | ### 1. 📩 Telegram Trigger  \n**Description**: Listens for incoming messages from the user via the connected Telegram bot. This is the entry point of the workflow. |
| Sticky Note6                 | Sticky Note            | Describes audio message check                    | None                        | None                                   | ### 2. Is Audio Message?  \n**Description**: Checks whether the incoming Telegram message is a audio message. If not, the workflow routes to an "unsupported message type" handler. |
| Sticky Note7                 | Sticky Note            | Describes audio download and transcription      | None                        | None                                   | ### 3.1. Download audio message and transcript with OpenAI\n Send the binary audio to OpenAI Whisper or your transcription service. |
| Sticky Note8                 | Sticky Note            | Describes user notification                       | None                        | None                                   | ### 4.1. Inform user via telegram\nSend friendly message to user with audio download URL              |
| Sticky Note9                 | Sticky Note            | Describes transformation and logging             | None                        | None                                   | ### 4.2 Transform & log expense\n- Transform the Output to Audio Record  \n- Log Audio Record to Google Sheet  |
| Sticky Note10                | Sticky Note            | Describes audio upload to Google Drive            | None                        | None                                   | ### 3.2. Upload the original audio to drive for later usage\nBackup the original audio file.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Voice Message Trigger Node**  
   - Type: *Telegram Trigger*  
   - Configure to listen for update type: `message`  
   - Enable download of incoming files (audio)  
   - Connect Telegram API credentials  
   - Position this node as the workflow’s start

2. **Add an If Node "Is audio message?"**  
   - Type: *If*  
   - Condition: Check if the string representation of incoming message JSON (`{{$json.message.toJsonString()}}`) contains `"audio/ogg"`  
   - True branch: Proceed to audio processing  
   - False branch: Route to unsupported message handler

3. **Unsupported Message Type Node (Telegram Send Message)**  
   - Type: *Telegram* (Send Message)  
   - Message text:  
     ```
     Sorry, I can’t read your input right now.
     Please send me a voice message, and I’ll help you transcribe and track it! 🎙️💬
     ```  
   - Chat ID: Extract from `Telegram Voice Message Trigger` node: `{{$node["Telegram Voice Message Trigger"].json.message.chat.id}}`  
   - Connect Telegram API credentials  
   - Connect from False output of If node

4. **Download Audio Message Node (Telegram Get File)**  
   - Type: *Telegram* (Get File)  
   - File ID: `{{$json.message.voice.file_id}}`  
   - Connect Telegram API credentials  
   - Connect from True output of If node

5. **Transcribe Audio Node (OpenAI Whisper)**  
   - Type: *OpenAI (LangChain)*  
   - Resource: `audio`  
   - Operation: `transcribe`  
   - Connect OpenAI API credentials  
   - Connect from "Download audio message" node output (audio binary)

6. **Upload Audio to Google Drive**  
   - Type: *Google Drive*  
   - Operation: Upload file  
   - File Name: `audio-{{ $now.toFormat("yyyyLLdd-HHmmss") }}-{{$binary.data.fileName}}`  
   - Drive: "My Drive"  
   - Folder ID: Use your Google Drive folder ID for backups (e.g., `1ObNNVJFR2vcKqP8p-ZnX_eaZy4gBHgha`)  
   - Connect Google Drive OAuth2 credentials  
   - Connect from "Download audio message" node output (audio binary)

7. **Merge Transcription and Upload Metadata**  
   - Type: *Merge*  
   - Connect two inputs:  
     - Input 1: From "Transcribe a recording" node  
     - Input 2: From "Upload file" node  
   - Use default merge mode (append by position)

8. **Transform Output Node (Code)**  
   - Type: *Code* (JavaScript)  
   - Code snippet:  
     ```js
     const inputs = $input.all();
     const transcriptData = inputs[0].json;
     const driveData = inputs[1].json;

     const result = {
       DateTime: driveData.createdTime || '',
       Duration: transcriptData.usage?.seconds || '',
       Transcript: transcriptData.text || '',
       AudioURL: driveData.webContentLink || '',
       ChatID: $('Telegram Voice Message Trigger').first().json.message.chat.id
     };

     return [{ json: result }];
     ```  
   - Connect from "Merge" node output

9. **Log Voice Record to Google Sheets**  
   - Type: *Google Sheets*  
   - Operation: Append row  
   - Document ID: Your Google Sheet ID (e.g., `1a-u0XHQWjn4VKbq5WpvSJy5_JgHuFl5A2Q2TEBDC5bI`)  
   - Sheet Name: Use sheet GID or name (e.g., `gid=0` or `Sheet1`)  
   - Mapping mode: Auto map input data fields to columns  
   - Connect Google Sheets OAuth2 credentials  
   - Connect from "Transform the output of voice record" node

10. **Inform User via Telegram**  
    - Type: *Telegram* (Send Message)  
    - Message text:  
      ```
      ✅ Voice Transcription Complete

      Your voice recording (⏱️ {{ $json.Duration }} seconds, recorded at {{ $json.DateTime }}) has been successfully transcribed and securely stored.

      📎 Original audio stored here: {{ $json.AudioURL }}

      Thank you for using VoiceScribe AI! 🎙️
      ```  
    - Chat ID: `{{$json.ChatID}}`  
    - Connect Telegram API credentials  
    - Connect from "Transform the output of voice record" node

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow automatically transcribes Telegram voice messages and stores both transcription and audio in Google Sheets and Google Drive respectively, providing a streamlined way to archive voice notes. It supports only audio messages and politely rejects unsupported message types.                                                                                                   | Workflow purpose description (from Sticky Note)                 |
| Users can customize this workflow by swapping OpenAI Whisper with other transcription providers like Deepgram or AssemblyAI, adding features like speaker detection, or routing output to other platforms such as Notion or Airtable.                                                                                                                                                      | Customization suggestions (from Sticky Note)                    |
| Prerequisites include setting up Telegram Bot credentials, Google Drive & Sheets OAuth2 credentials, and OpenAI API credentials within n8n. The workflow requires an environment capable of running n8n workflows with internet access for API calls.                                                                                                                                         | Setup requirements listed in Sticky Note                        |
| For more details and live demos, visit [n8n official website](https://n8n.io) and review API documentation for Telegram, Google Drive, Google Sheets, and OpenAI Whisper.                                                                                                                                                                                                                     | Official documentation and API links                            |
| Visual screenshots included in workflow (Sticky Note1 and Sticky Note2) for UI reference and debugging.                                                                                                                                                                                                                                                                                     | Workflow screenshots links in Sticky Notes                       |

---

**Disclaimer:**  
The provided content is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.