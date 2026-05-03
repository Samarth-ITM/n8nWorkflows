Transcribe audio files with Google Gemini and Telegram

https://n8nworkflows.xyz/workflows/transcribe-audio-files-with-google-gemini-and-telegram-3388


# Transcribe audio files with Google Gemini and Telegram

### 1. Workflow Overview

This workflow automates the transcription of audio messages sent to a Telegram bot by leveraging Google Gemini’s free transcription model. It is designed for users who frequently receive audio messages on Telegram and want quick, accurate text transcriptions returned directly in the chat.

**Target Use Cases:**  
- Content creators and podcasters needing transcripts of audio clips  
- Coaches or educators who receive voice notes for documentation  
- Anyone receiving numerous audio messages on Telegram wanting automated transcription  

**Logical Blocks:**  
- **1.1 Telegram Input Reception:** Listens for incoming Telegram messages and routes audio files for processing.  
- **1.2 Audio File Download and Preparation:** Downloads the audio file from Telegram and prepares it for upload to Google Gemini.  
- **1.3 Google Gemini Transcription Request:** Uploads the audio to Google Gemini, requests transcription, and receives the text.  
- **1.4 Telegram Reply:** Sends the transcription text back to the user on Telegram.  
- **1.5 Alternative Message Handling:** Routes non-audio messages separately (no transcription).  
- **1.6 Additional Input Sources (Optional):** Nodes for processing audio files from Google Drive or direct uploads (not connected to main flow).  

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

- **Overview:**  
  This block listens for incoming Telegram messages via webhook and determines if the message contains audio for transcription or other content.

- **Nodes Involved:**  
  - Telegram Trigger1  
  - Switch  
  - Telegram3  
  - Text messages go this way (NoOp)  

- **Node Details:**  

  - **Telegram Trigger1**  
    - Type: Telegram Trigger  
    - Role: Entry point for Telegram messages via webhook.  
    - Config: Default webhook listening for all message types.  
    - Inputs: External Telegram webhook calls.  
    - Outputs: Routes message data to Switch node.  
    - Edge Cases: Webhook failures, invalid Telegram token, message format variations.  

  - **Switch**  
    - Type: Switch  
    - Role: Routes messages based on content type (audio vs text).  
    - Config: Conditions to detect if message contains audio file.  
    - Inputs: From Telegram Trigger1.  
    - Outputs:  
      - Audio messages → Telegram3 node  
      - Text messages → Text messages go this way node  
    - Edge Cases: Messages without audio or unsupported media types may be misrouted.  

  - **Telegram3**  
    - Type: Telegram node (Send/Receive)  
    - Role: Handles audio message processing start.  
    - Config: Uses Telegram credentials with bot token.  
    - Inputs: Audio messages from Switch.  
    - Outputs: Forwards to initialize upload session and Merge nodes.  
    - Edge Cases: Telegram API rate limits, invalid file references.  

  - **Text messages go this way**  
    - Type: NoOp (No Operation)  
    - Role: Placeholder for non-audio messages, effectively ignoring or handling them separately.  
    - Inputs: Text messages from Switch.  
    - Outputs: None.  
    - Edge Cases: None (safe fallback).  

---

#### 1.2 Audio File Download and Preparation

- **Overview:**  
  Downloads the audio file from Telegram, initializes an upload session with Google Gemini, and merges data for upload.

- **Nodes Involved:**  
  - initialize upload session (HTTP Request)  
  - Merge  
  - Upload file (HTTP Request)  

- **Node Details:**  

  - **initialize upload session**  
    - Type: HTTP Request  
    - Role: Starts a new upload session with Google Gemini API.  
    - Config: HTTP POST to Gemini’s upload session endpoint with authentication headers (API key).  
    - Inputs: From Telegram3 node (audio file metadata).  
    - Outputs: Upload session details to Merge node.  
    - Edge Cases: API authentication errors, network timeouts, invalid session parameters.  

  - **Merge**  
    - Type: Merge  
    - Role: Combines upload session data and audio file data into one dataset for upload.  
    - Config: Merges inputs from initialize upload session and Telegram3 nodes.  
    - Inputs: Two inputs — upload session response and audio file info.  
    - Outputs: Combined data to Upload file node.  
    - Edge Cases: Mismatched data, missing inputs causing merge failure.  

  - **Upload file**  
    - Type: HTTP Request  
    - Role: Uploads the audio file to Google Gemini using the session info.  
    - Config: HTTP PUT or POST with file binary data and session token.  
    - Inputs: From Merge node.  
    - Outputs: Upload confirmation and transcription request trigger to Ask Gemini to transcribe node.  
    - Edge Cases: Upload failures, file size limits, network interruptions.  

---

#### 1.3 Google Gemini Transcription Request

- **Overview:**  
  Sends the uploaded audio file to Google Gemini’s transcription endpoint and receives the transcription text.

- **Nodes Involved:**  
  - Ask Gemini to transcribe (HTTP Request)  

- **Node Details:**  

  - **Ask Gemini to transcribe**  
    - Type: HTTP Request  
    - Role: Calls Google Gemini’s transcription API with uploaded audio reference.  
    - Config: HTTP POST with session ID or file reference, requesting transcription.  
    - Inputs: From Upload file node.  
    - Outputs: Transcription text to Reply in Telegram node.  
    - Edge Cases: API rate limits, transcription errors, malformed requests.  

---

#### 1.4 Telegram Reply

- **Overview:**  
  Sends the transcription result back to the user on Telegram.

- **Nodes Involved:**  
  - Reply in Telegram (Telegram node)  

- **Node Details:**  

  - **Reply in Telegram**  
    - Type: Telegram node  
    - Role: Sends a message containing the transcription text to the original sender.  
    - Config: Uses Telegram bot credentials, sends message to chat ID from original message.  
    - Inputs: From Ask Gemini to transcribe node.  
    - Outputs: None (end of flow).  
    - Edge Cases: Telegram API errors, message formatting issues, chat ID missing.  

---

#### 1.5 Alternative Message Handling

- **Overview:**  
  Handles messages that are not audio files by routing them to a NoOp node, effectively ignoring or allowing future extension.

- **Nodes Involved:**  
  - Text messages go this way (NoOp)  

- **Node Details:**  
  - See above in 1.1.  

---

#### 1.6 Additional Input Sources (Optional / Not Connected)

- **Overview:**  
  Nodes present for processing audio files from Google Drive, direct uploads, or webhooks, but not connected to the main flow. These can be adapted for other input sources.

- **Nodes Involved:**  
  - Process every uploaded file in a folder (Google Drive Trigger)  
  - Download file from Google Drive (Google Drive node)  
  - Receive using webhook (Webhook node)  
  - Read Files from Disk (Read/Write File node)  
  - Download file from URL (HTTP Request)  

- **Node Details:**  
  - These nodes are configured for alternative audio input methods but require manual integration.  
  - Edge Cases: Authentication with Google Drive, webhook security, file format compatibility.  

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                      | Input Node(s)                 | Output Node(s)               | Sticky Note                          |
|-------------------------------|---------------------|-----------------------------------|------------------------------|-----------------------------|------------------------------------|
| Telegram Trigger1              | Telegram Trigger    | Entry point for Telegram messages | -                            | Switch                      |                                    |
| Switch                        | Switch              | Routes audio vs text messages      | Telegram Trigger1             | Telegram3, Text messages go this way |                                    |
| Telegram3                     | Telegram            | Handles audio message processing   | Switch                       | initialize upload session, Merge |                                    |
| initialize upload session     | HTTP Request        | Starts Google Gemini upload session| Telegram3                    | Merge                       |                                    |
| Merge                        | Merge               | Combines upload session and file data | initialize upload session, Telegram3 | Upload file                 |                                    |
| Upload file                  | HTTP Request        | Uploads audio file to Gemini       | Merge                        | Ask Gemini to transcribe    |                                    |
| Ask Gemini to transcribe     | HTTP Request        | Requests transcription from Gemini| Upload file                  | Reply in Telegram           |                                    |
| Reply in Telegram            | Telegram            | Sends transcription back to user  | Ask Gemini to transcribe     | -                           |                                    |
| Text messages go this way    | NoOp                | Handles non-audio messages         | Switch                       | -                           |                                    |
| Process every uploaded file in a folder | Google Drive Trigger | Optional: triggers on new Drive files | -                            | Download file from Google Drive |                                    |
| Download file from Google Drive | Google Drive       | Optional: downloads files from Drive | Process every uploaded file in a folder | -                           |                                    |
| Receive using webhook        | Webhook             | Optional: receives files via webhook | -                            | Read Files from Disk        |                                    |
| Read Files from Disk         | Read/Write File     | Optional: reads files from disk    | Receive using webhook         | Download file from URL      |                                    |
| Download file from URL       | HTTP Request        | Optional: downloads files from URL | Read Files from Disk          | -                           |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials (bot token).  
   - Set to listen for all message types.  
   - This node will receive incoming Telegram messages.  

2. **Add Switch Node**  
   - Type: Switch  
   - Configure to check if incoming message contains an audio file (e.g., check if `message.voice` or `message.audio` exists).  
   - Set two outputs:  
     - Output 1: Audio messages  
     - Output 2: Non-audio messages  

3. **Add Telegram Node (Telegram3)**  
   - Type: Telegram  
   - Use same Telegram credentials.  
   - Connect Switch output 1 (audio messages) to this node.  
   - This node prepares audio message data for further processing.  

4. **Add HTTP Request Node (initialize upload session)**  
   - Type: HTTP Request  
   - Configure to POST to Google Gemini’s upload session endpoint.  
   - Add authentication header with your Google Gemini API key.  
   - Connect Telegram3 output to this node.  

5. **Add Merge Node**  
   - Type: Merge  
   - Configure to merge two inputs:  
     - Input 1: Response from initialize upload session  
     - Input 2: Audio file data from Telegram3  
   - Connect initialize upload session output to input 1, Telegram3 output to input 2.  

6. **Add HTTP Request Node (Upload file)**  
   - Type: HTTP Request  
   - Configure to upload the audio file to Google Gemini using the session info from Merge node.  
   - Use PUT or POST as required by Gemini API.  
   - Connect Merge output to this node.  

7. **Add HTTP Request Node (Ask Gemini to transcribe)**  
   - Type: HTTP Request  
   - Configure to POST a transcription request to Google Gemini API referencing the uploaded file/session.  
   - Use API key authentication.  
   - Connect Upload file output to this node.  

8. **Add Telegram Node (Reply in Telegram)**  
   - Type: Telegram  
   - Use Telegram credentials.  
   - Configure to send a message to the chat ID from the original Telegram message.  
   - Message content: transcription text from Ask Gemini to transcribe node.  
   - Connect Ask Gemini to transcribe output to this node.  

9. **Add NoOp Node (Text messages go this way)**  
   - Type: NoOp  
   - Connect Switch output 2 (non-audio messages) to this node.  
   - This node acts as a placeholder for non-audio messages.  

10. **Credential Setup**  
    - Telegram credentials: Use your Telegram bot token.  
    - Google Gemini credentials: Use your API key with appropriate permissions.  

11. **Optional: Add nodes for alternative audio sources**  
    - Google Drive Trigger and Google Drive nodes for processing files from Drive.  
    - Webhook and file read nodes for direct uploads.  
    - These require additional configuration and are not connected to main flow by default.  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                         |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow uses Google Gemini’s free tier for transcription, requiring an API key.           | Google Gemini API documentation                         |
| Telegram bot token is required; create your bot via BotFather on Telegram.                       | Telegram Bot API documentation                          |
| Creator’s other templates available at:                                                         | https://n8n.io/creators/solomon/                        |
| Workflow supports adaptation for other audio sources like WhatsApp, Google Drive, or uploads.   | See optional nodes for Google Drive and webhook inputs |
| Visual workflow screenshot included in original template for reference.                         | (Image not included here)                               |

---

This document fully describes the workflow structure, node configurations, and logic flow to enable reproduction, modification, and troubleshooting.