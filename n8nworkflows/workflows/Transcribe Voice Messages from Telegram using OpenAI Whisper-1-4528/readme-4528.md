Transcribe Voice Messages from Telegram using OpenAI Whisper-1

https://n8nworkflows.xyz/workflows/transcribe-voice-messages-from-telegram-using-openai-whisper-1-4528


# Transcribe Voice Messages from Telegram using OpenAI Whisper-1

### 1. Workflow Overview

This n8n workflow automates the transcription of voice messages sent via Telegram using OpenAI’s Whisper-1 model. It listens for incoming Telegram messages, distinguishes between text and voice messages, downloads the audio files from voice messages, sends them to OpenAI for transcription, and then posts the transcription back to the Telegram chat. The workflow is structured into logical blocks to handle Telegram input, routing based on message type, audio file retrieval, transcription via OpenAI, and sending responses.

Logical blocks:  
- **1.1 Input Reception:** Listens for new Telegram messages using a Telegram trigger node.  
- **1.2 Message Routing:** Uses a switch node to separate text messages from voice messages.  
- **1.3 Audio Retrieval:** Downloads the audio file from Telegram when a voice message is detected.  
- **1.4 Transcription Processing:** Sends the audio file to OpenAI Whisper-1 for transcription.  
- **1.5 Response Sending:** Sends the transcription (or original text) back to the originating Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming Telegram messages (any message update) and triggers the workflow execution.

- **Nodes Involved:**  
  - *Message Trigger*

- **Node Details:**  
  - **Message Trigger**  
    - Type: Telegram Trigger  
    - Role: Entry point that listens for new Telegram messages (update type: “message”).  
    - Configuration: Watches for all message updates without filters.  
    - Credentials: Uses a Telegram Bot API credential named “cbd bot”.  
    - Input: Incoming webhook from Telegram.  
    - Output: Emits the Telegram message JSON for downstream processing.  
    - Edge cases: Potential failures include webhook registration errors, Telegram API downtime, or invalid bot token.  
    - Version: 1.2  

#### 1.2 Message Routing

- **Overview:**  
  Routes incoming messages based on their content type—whether it is a text message or a voice message—using a switch node.

- **Nodes Involved:**  
  - *Route Chat Input*

- **Node Details:**  
  - **Route Chat Input**  
    - Type: Switch  
    - Role: Checks if the incoming message contains text or voice message data.  
    - Configuration:  
      - Two outputs:  
        - “text” output triggered if `$json.message.text` exists (strict type validation).  
        - “voice” output triggered if `$json.message.voice` exists (strict type validation).  
    - Inputs: Connected from *Message Trigger*.  
    - Outputs:  
      - “text” output goes directly to *Send transcription message* (to echo text).  
      - “voice” output goes to *Get audio file* for further processing.  
    - Edge cases: Messages without text or voice fields will not match either output and thus be dropped silently. Expression errors if message structure differs.  
    - Version: 3.2  

#### 1.3 Audio Retrieval

- **Overview:**  
  Downloads the audio file from Telegram servers when a voice message is detected.

- **Nodes Involved:**  
  - *Get audio file*

- **Node Details:**  
  - **Get audio file**  
    - Type: Telegram Node (file resource)  
    - Role: Retrieves the audio file linked to the voice message using the Telegram file ID.  
    - Configuration:  
      - File ID expression: `{{$json.message.voice.file_id}}` extracts the file identifier from the incoming message.  
      - Resource: “file” to download the actual audio content.  
    - Credentials: Same Telegram bot credential “cbd bot” as the trigger.  
    - Inputs: Connected from “voice” output of *Route Chat Input*.  
    - Outputs: Provides binary audio data for transcription.  
    - Edge cases: Possible failures include invalid file ID, Telegram API errors, large file size/timeouts, or network issues.  
    - Version: 1.2  

#### 1.4 Transcription Processing

- **Overview:**  
  Sends the downloaded audio file to OpenAI’s Whisper-1 model for transcription.

- **Nodes Involved:**  
  - *Transcribe audio*

- **Node Details:**  
  - **Transcribe audio**  
    - Type: OpenAI Node (LangChain integration)  
    - Role: Transcribes audio content using OpenAI Whisper model.  
    - Configuration:  
      - Resource set to “audio”  
      - Operation: “transcribe”  
      - No additional options configured (default transcription behavior).  
    - Credentials: OpenAI API credential named “OpenAi account”.  
    - Inputs: Receives binary audio from *Get audio file*.  
    - Outputs: Returns the transcription text in JSON property `text`.  
    - Edge cases: Possible errors include OpenAI API key invalid, rate limits, audio format incompatibility, or transcription timeout.  
    - Version: 1.8  

#### 1.5 Response Sending

- **Overview:**  
  Sends the transcription result (or original text message) back to the Telegram chat.

- **Nodes Involved:**  
  - *Send transcription message*

- **Node Details:**  
  - **Send transcription message**  
    - Type: Telegram Node  
    - Role: Sends a text message to the originating Telegram chat with the transcription or echoes the original text.  
    - Configuration:  
      - Chat ID: Extracted dynamically from the original message’s chat ID via expression `{{$('Message Trigger').item.json.message.chat.id}}`.  
      - Text: Combines the original message text and transcription result: `{{$json.message.text}}\n{{$json.text}}`. For voice messages, `$json.message.text` is usually empty, so the transcription appears after a newline.  
      - appendAttribution: Disabled to avoid bot attribution text.  
    - Credentials: Telegram bot credential “cbd bot”.  
    - Inputs:  
      - From *Route Chat Input* “text” output (for text messages) or  
      - From *Transcribe audio* output (for voice messages after transcription).  
    - Outputs: None (endpoint node).  
    - Edge cases: Sending can fail if chat ID is invalid, bot banned, or Telegram API errors occur.  
    - Version: 1.2  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                         | Input Node(s)             | Output Node(s)               | Sticky Note                                                  |
|-------------------------|-------------------------|---------------------------------------|---------------------------|-----------------------------|--------------------------------------------------------------|
| Message Trigger         | Telegram Trigger        | Listens for incoming Telegram messages | (Webhook entry)            | Route Chat Input             |                                                              |
| Route Chat Input        | Switch                  | Routes messages by type (text/voice)  | Message Trigger            | Send transcription message (text output), Get audio file (voice output) |                                                              |
| Get audio file          | Telegram (File resource)| Downloads voice message audio file     | Route Chat Input (voice)   | Transcribe audio            |                                                              |
| Transcribe audio        | OpenAI (LangChain)      | Transcribes audio using OpenAI Whisper | Get audio file             | Send transcription message   |                                                              |
| Send transcription message | Telegram              | Sends transcription or text back to chat | Route Chat Input (text) and Transcribe audio | (None)                      |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Set “Updates” to `message`  
   - Authenticate with your Telegram Bot API credential (create if needed)  
   - Position: Entry point  
   - This node listens for incoming messages and triggers the workflow.

2. **Create Switch Node ("Route Chat Input")**  
   - Type: Switch  
   - Connect input from Telegram Trigger node.  
   - Add two outputs:  
     - Output 1: Named “text”  
       - Condition: Check if `{{$json.message.text}}` exists and is a non-empty string.  
     - Output 2: Named “voice”  
       - Condition: Check if `{{$json.message.voice}}` exists as an object.  
   - This node routes messages based on whether they contain text or voice.

3. **Create Telegram Node ("Get audio file")**  
   - Type: Telegram (File resource)  
   - Connect input from the “voice” output of the Switch node.  
   - Set “Resource” to `file`.  
   - Set “File ID” to expression: `{{$json.message.voice.file_id}}` to get the voice message file.  
   - Authenticate using the same Telegram Bot credential.  
   - This node downloads the voice message audio file.

4. **Create OpenAI Node ("Transcribe audio")**  
   - Type: OpenAI (LangChain integration)  
   - Connect input from “Get audio file” node.  
   - Set “Resource” to `audio`.  
   - Set “Operation” to `transcribe`.  
   - No additional options necessary unless specific transcription settings are desired.  
   - Authenticate using an OpenAI API credential.  
   - This node sends audio to OpenAI Whisper and receives transcription.

5. **Create Telegram Node ("Send transcription message")**  
   - Type: Telegram (Send Message)  
   - Connect two inputs:  
     - From “text” output of the Switch node (for text messages).  
     - From “Transcribe audio” node (for voice messages after transcription).  
   - Set “Chat ID” to expression: `{{$('Message Trigger').item.json.message.chat.id}}` to reply in the correct chat.  
   - Set “Text” to expression: `{{$json.message.text}}\n{{$json.text}}` to send original text plus transcription.  
   - Disable “appendAttribution” (set to false) to keep messages clean.  
   - Authenticate with the Telegram Bot credential.  
   - This node sends the transcription or echoes text back to Telegram.

6. **Connect nodes as follows:**  
   - Telegram Trigger → Route Chat Input  
   - Route Chat Input “text” output → Send transcription message  
   - Route Chat Input “voice” output → Get audio file → Transcribe audio → Send transcription message

7. **Verify credentials:**  
   - Telegram Bot API OAuth or token-based credential with permissions to read messages and send messages.  
   - OpenAI API key with access to audio transcription (Whisper-1).

8. **Test the workflow:**  
   - Send a text message to the Telegram bot → Should be echoed back unchanged.  
   - Send a voice message → Should be downloaded, transcribed, and transcription sent back.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                      |
|------------------------------------------------------------------------------|-----------------------------------------------------|
| Uses OpenAI Whisper-1 model for audio transcription within n8n workflow.    | OpenAI Whisper documentation                        |
| Telegram Bot credentials must have webhook enabled and proper permissions.   | Telegram Bot API documentation                       |
| Workflow handles only voice and text messages; other message types ignored.  | n8n switch node documentation                        |
| Expression syntax examples: `{{$json.message.voice.file_id}}` and `{{$('Node Name').item.json...}}` | n8n expression editor guide                         |

---

**Disclaimer:** The provided text is generated from an automated n8n workflow export. It complies fully with current content policies and contains only legal, public data without offensive or protected content.