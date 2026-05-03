Transcribing Telegram voice messages using Whisper and Gemini with a fallback mechanism

https://n8nworkflows.xyz/workflows/transcribing-telegram-voice-messages-using-whisper-and-gemini-with-a-fallback-mechanism-9625


# Transcribing Telegram voice messages using Whisper and Gemini with a fallback mechanism

### 1. Workflow Overview

This workflow automates the transcription of Telegram voice messages or audio files using OpenAI Whisper as the primary engine and Google Gemini as a fallback. It is designed for use cases where users send voice notes or audio content via Telegram, and the workflow transcribes these into text messages sent back to the Telegram chat. The workflow incorporates authorization checks, audio type recognition, format validation, and text length management with chunked messaging for long transcriptions.

**Logical blocks:**

- **1.1 Input Reception and Permission Check**  
  Receives incoming Telegram messages, verifies if the sender is authorized.

- **1.2 Audio Type Recognition and File ID Extraction**  
  Determines if the message contains a voice note or audio, extracts the file ID accordingly.

- **1.3 Audio Format Validation**  
  Validates the audio MIME type to ensure the file format is supported.

- **1.4 File Retrieval for Transcription**  
  Downloads the audio file from Telegram for transcription by Whisper (OpenAI) or Gemini (Google).

- **1.5 Primary Transcription via OpenAI Whisper**  
  Performs the transcription using OpenAI’s Whisper model.

- **1.6 Fallback Transcription via Google Gemini**  
  If OpenAI transcription fails, attempts transcription with Google Gemini.

- **1.7 Text Processing and Length Management**  
  Checks transcription length; if over 4000 characters, splits the text into chunks.

- **1.8 Output Transmission to Telegram**  
  Sends the transcribed text back to the Telegram chat, either as a single message or chunked messages.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Permission Check

- **Overview:**  
  This block listens for any Telegram message and verifies if the sender is authorized to use the transcription service.

- **Nodes Involved:**  
  - Telegram Trigger1  
  - Access Check  
  - MSG - Access denied!

- **Node Details:**

  - **Telegram Trigger1**  
    - Type: Telegram Trigger  
    - Role: Entry point, listens for incoming Telegram messages with the "message" update type.  
    - Configuration: Uses Telegram API credentials; triggers on any message.  
    - Input: None (webhook trigger).  
    - Output: Telegram message JSON.  
    - Failure types: Webhook misconfiguration, Telegram API authentication failure.

  - **Access Check**  
    - Type: IF node  
    - Role: Checks sender username against a whitelist ("User 1" or "User 2").  
    - Configuration: Combinator OR condition checking $json.message.from.username.  
    - Input: Telegram Trigger1 output.  
    - Output: Two branches - authorized (true) and unauthorized (false).  
    - Edge cases: Username missing or changed; case sensitivity impacts matching.  
    - Failure: Expression evaluation errors if message or username missing.

  - **MSG - Access denied!**  
    - Type: Telegram  
    - Role: Sends "Access denied!" message to unauthorized users.  
    - Configuration: Sends to chat ID derived from the triggering message; disables attribution.  
    - Input: Access Check unauthorized branch.  
    - Output: None (end node).  
    - Failure: Telegram API send message errors.

#### 1.2 Audio Type Recognition and File ID Extraction

- **Overview:**  
  Identifies whether the incoming message contains a voice note or an audio file and extracts the corresponding file ID for download.

- **Nodes Involved:**  
  - Determining The Type Of Document  
  - Assigning file_id For Voice  
  - Assigning file_id For Audio  
  - MSG - No file!

- **Node Details:**

  - **Determining The Type Of Document**  
    - Type: Switch  
    - Role: Identifies message type: "Voice" if voice note exists, "Audio" if audio file exists, otherwise fallback.  
    - Configuration: Checks existence of $json.message.voice or $json.message.audio.  
    - Input: Access Check authorized branch.  
    - Output: Three branches: Voice, Audio, extra (fallback).  
    - Edge cases: Message without voice or audio, e.g., text messages.

  - **Assigning file_id For Voice**  
    - Type: Set  
    - Role: Sets "file_id" variable to voice note’s file_id.  
    - Configuration: Assigns $json.message.voice.file_id to "file_id".  
    - Input: Determining The Type Of Document / Voice branch.  
    - Output: Next node for transcription decision.  
    - Edge cases: Missing file_id in voice object.

  - **Assigning file_id For Audio**  
    - Type: Set  
    - Role: Sets "file_id" variable to audio file’s file_id.  
    - Configuration: Assigns $json.message.audio.file_id to "file_id".  
    - Input: Determining The Type Of Document / Audio branch.  
    - Output: Next node for transcription decision.  
    - Edge cases: Missing file_id in audio object.

  - **MSG - No file!**  
    - Type: Telegram  
    - Role: Sends "No file!" message when neither voice nor audio is detected.  
    - Configuration: Sends to chat ID from original message; no attribution.  
    - Input: Determining The Type Of Document / extra branch.  
    - Output: None (ends flow).  
    - Failure: Telegram API errors.

#### 1.3 Audio Format Validation

- **Overview:**  
  Validates that the audio's MIME type is among supported types (ogg, mp3, mp4, m4a) before attempting transcription.

- **Nodes Involved:**  
  - Transcription Decision Node  
  - MSG - File not recognized

- **Node Details:**

  - **Transcription Decision Node**  
    - Type: IF  
    - Role: Checks if the MIME type of the voice or audio matches supported types.  
    - Configuration: OR condition comparing MIME types: audio/ogg, audio/mpeg, audio/mp4, audio/m4a.  
    - Input: From Assigning file_id For Voice or Audio.  
    - Output: True branch → proceed to download file; False branch → send unrecognized format message.  
    - Edge cases: Unsupported MIME types; missing MIME type fields.

  - **MSG - File not recognized**  
    - Type: Telegram  
    - Role: Sends "File not recognized" when MIME type unsupported.  
    - Configuration: Sends to chat ID from message; no attribution.  
    - Input: Transcription Decision Node false branch.  
    - Output: None (ends flow).  
    - Failure: Telegram API errors.

#### 1.4 File Retrieval for Transcription

- **Overview:**  
  Downloads the audio file from Telegram servers for transcription. The file is fetched twice: once for OpenAI Whisper and again for Gemini fallback if needed.

- **Nodes Involved:**  
  - Get File GPT  
  - Get File Gemini  
  - MSG - Starting transcription. Please wait.

- **Node Details:**

  - **Get File GPT**  
    - Type: Telegram (Get File)  
    - Role: Downloads the file by file_id for OpenAI transcription.  
    - Configuration: Uses file_id from previous steps.  
    - Input: Transcription Decision Node true branch.  
    - Output: Binary data of the file.  
    - Edge cases: Telegram API file retrieval failures, invalid file_id.

  - **Get File Gemini**  
    - Type: Telegram (Get File)  
    - Role: Downloads the same file for Gemini transcription fallback.  
    - Configuration: Uses file_id from Get File GPT node’s output JSON path (result.file_id).  
    - Input: OpenAI transcription error branch.  
    - Output: Binary file data.  
    - Edge cases: File not found, latency in re-downloading file.

  - **MSG - Starting transcription. Please wait.**  
    - Type: Telegram  
    - Role: Notifies user transcription is starting.  
    - Configuration: Sends to chat ID from the original Telegram message; no attribution.  
    - Input: After Get File GPT node (parallel with transcription start).  
    - Output: None (informational).  
    - Failure: Telegram API send message errors.

#### 1.5 Primary Transcription via OpenAI Whisper

- **Overview:**  
  Uses OpenAI Whisper API to transcribe the downloaded audio file.

- **Nodes Involved:**  
  - Transcription by OpenAi  
  - Assignment of variable TEXT

- **Node Details:**

  - **Transcription by OpenAi**  
    - Type: OpenAI (LangChain Node)  
    - Role: Transcribes audio file to text using Whisper.  
    - Configuration: Resource set to "audio", operation "transcribe".  
    - Credentials: OpenAI API (OAuth or API key).  
    - Input: Binary audio data from Get File GPT.  
    - Output: JSON containing "text" field with transcription.  
    - Error Handling: On error continues to next node (fallback to Gemini).  
    - Edge cases: API quota limits, network errors, audio format issues.

  - **Assignment of variable TEXT**  
    - Type: Set  
    - Role: Extracts transcription text into "text" variable for downstream processing.  
    - Configuration: Sets "text" to $json.text from transcription output.  
    - Input: Transcription by OpenAi success branch.  
    - Output: Text length check node.  
    - Edge cases: Missing or empty transcription text.

#### 1.6 Fallback Transcription via Google Gemini

- **Overview:**  
  If OpenAI Whisper transcription fails, this block attempts transcription using Google Gemini API.

- **Nodes Involved:**  
  - Get File Gemini  
  - Transcription by Gemini  
  - Assignment of variable TEXT1

- **Node Details:**

  - **Get File Gemini**  
    - (Described above in 1.4) Downloads the file again for Gemini.

  - **Transcription by Gemini**  
    - Type: Google Gemini (LangChain Node)  
    - Role: Transcribes audio file binary using Google Gemini model "models/gemini-2.5-flash".  
    - Configuration: Input type is binary, binary property named "data".  
    - Credentials: Google Palm API.  
    - Input: Binary audio from Get File Gemini.  
    - Output: JSON containing transcription parts in "content.parts[0].text".  
    - Edge cases: API errors, unsupported audio, network issues.

  - **Assignment of variable TEXT1**  
    - Type: Set  
    - Role: Extracts first part of Gemini transcription into "text" variable.  
    - Configuration: Sets "text" to $json.content.parts[0].text.  
    - Input: Transcription by Gemini output.  
    - Output: Text length check node.  
    - Edge cases: Missing parts or empty transcription.

#### 1.7 Text Processing and Length Management

- **Overview:**  
  Evaluates the transcription length to decide between sending the whole message directly or splitting it into chunks if it exceeds 4000 characters.

- **Nodes Involved:**  
  - Checking the length of a message  
  - Text Division

- **Node Details:**

  - **Checking the length of a message**  
    - Type: IF  
    - Role: Checks if the "text" variable length is less than 4000 characters.  
    - Configuration: Number less than operation on $json["text"].length < 4000.  
    - Input: From Assignment of variable TEXT or TEXT1.  
    - Output: True branch (short text), False branch (long text).  
    - Edge cases: Empty or undefined text, encoding issues affecting length.

  - **Text Division**  
    - Type: Code  
    - Role: Splits the long text into chunks of 4000 characters each.  
    - Configuration: JavaScript code slicing the "text" string into 4000 char segments, outputs array of JSON objects with "body" property.  
    - Input: False branch from length check.  
    - Output: Array of message chunks.  
    - Edge cases: Unicode characters split incorrectly, empty chunks, large texts causing performance issues.

#### 1.8 Output Transmission to Telegram

- **Overview:**  
  Sends the transcribed text back to the user in Telegram, either as a single message or multiple chunked messages.

- **Nodes Involved:**  
  - MSG - Output  
  - MSG - Output with chunking

- **Node Details:**

  - **MSG - Output**  
    - Type: Telegram  
    - Role: Sends final transcription text if under 4000 characters.  
    - Configuration: Sends $json.text to the original chat ID; no attribution appended.  
    - Input: Length check true branch.  
    - Output: None (ends flow).  
    - Edge cases: Telegram message length limits, API errors.

  - **MSG - Output with chunking**  
    - Type: Telegram  
    - Role: Sends each chunk of a long transcription sequentially.  
    - Configuration: Sends $json.body per chunk to chat ID; no attribution.  
    - Input: Output of Text Division node.  
    - Output: None (ends flow).  
    - Edge cases: Rate limits from sending multiple messages, chunk size errors.

---

### 3. Summary Table

| Node Name                    | Node Type                | Functional Role                      | Input Node(s)                    | Output Node(s)                         | Sticky Note                                    |
|------------------------------|--------------------------|------------------------------------|---------------------------------|--------------------------------------|-----------------------------------------------|
| Telegram Trigger1             | Telegram Trigger         | Entry point, receives messages     | None                            | Access Check                        |                                               |
| Access Check                 | IF                       | Authorize user                     | Telegram Trigger1               | Determining The Type Of Document, MSG - Access denied! | ## Access permission                           |
| MSG - Access denied!          | Telegram                 | Notify unauthorized access         | Access Check                   | None                                | ## Access denied message                       |
| Determining The Type Of Document | Switch                   | Detect message type (voice/audio)  | Access Check                   | Assigning file_id For Voice, Assigning file_id For Audio, MSG - No file! | ## Recognition of record type                  |
| Assigning file_id For Voice   | Set                      | Extract file_id from voice message | Determining The Type Of Document | Transcription Decision Node          |                                               |
| Assigning file_id For Audio   | Set                      | Extract file_id from audio message | Determining The Type Of Document | Transcription Decision Node          |                                               |
| MSG - No file!               | Telegram                 | Notify no audio file found          | Determining The Type Of Document | None                                | ## File not found message                      |
| Transcription Decision Node   | IF                       | Validate audio MIME type            | Assigning file_id For Voice, Assigning file_id For Audio | Get File GPT, MSG - File not recognized | ## Recognition of record type                  |
| MSG - File not recognized     | Telegram                 | Notify unsupported file format     | Transcription Decision Node     | None                                | ## File not recognized message                 |
| Get File GPT                 | Telegram                 | Download file for OpenAI Whisper   | Transcription Decision Node     | Transcription by OpenAi, MSG - Starting transcription. Please wait. | ## Get a file for transcribe by GPT            |
| MSG - Starting transcription. Please wait. | Telegram                 | Notify transcription start         | Get File GPT                   | None                                | ## notification of commencement of work       |
| Transcription by OpenAi       | OpenAI                   | Transcribe audio with Whisper      | Get File GPT                   | Assignment of variable TEXT, Get File Gemini (on error) | ## If transcription via GPT fails, we start doing it via Gemini. |
| Get File Gemini              | Telegram                 | Download file for Gemini fallback  | Transcription by OpenAi (error) | Transcription by Gemini             |                                               |
| Transcription by Gemini       | Google Gemini            | Transcribe audio with Gemini fallback | Get File Gemini               | Assignment of variable TEXT1         |                                               |
| Assignment of variable TEXT   | Set                      | Store OpenAI transcription text    | Transcription by OpenAi         | Checking the length of a message      |                                               |
| Assignment of variable TEXT1  | Set                      | Store Gemini transcription text    | Transcription by Gemini         | Checking the length of a message      |                                               |
| Checking the length of a message | IF                       | Check if text length < 4000 chars  | Assignment of variable TEXT, Assignment of variable TEXT1 | MSG - Output, Text Division             | ## Send the result if the text is less than 4000 characters; ## if the text is longer than 4000 characters, we split it into parts |
| Text Division                | Code                     | Split long text into 4000-char chunks | Checking the length of a message | MSG - Output with chunking           | ## sending parts of a message                  |
| MSG - Output                 | Telegram                 | Send short transcription text      | Checking the length of a message | None                                |                                               |
| MSG - Output with chunking    | Telegram                 | Send chunked transcription parts   | Text Division                  | None                                |                                               |
| Sticky Note (multiple)        | Sticky Note              | Descriptive and instructional notes | Various                       | None                                | See sticky notes content in node descriptions |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Set updates to "message" only.  
   - Credentials: Use Telegram API credentials with bot token.  
   - Position: Entry point.

2. **Add IF Node "Access Check"**  
   - Condition: Check if `$json.message.from.username` equals "User 1" or "User 2" (case sensitive).  
   - Input: Connect Telegram Trigger output to this node.

3. **Add Telegram Node "MSG - Access denied!"**  
   - Text: "Access denied!"  
   - Chat ID: Set to `={{ $json.message.chat.id }}` from trigger.  
   - Credentials: Same Telegram API credentials.  
   - Connect from Access Check’s False output.

4. **Add Switch Node "Determining The Type Of Document"**  
   - Rules:  
     - Output Key "Voice": Condition exists `$json.message.voice`  
     - Output Key "Audio": Condition exists `$json.message.audio`  
     - Fallback output for others.  
   - Connect from Access Check True output.

5. **Add Set Node "Assigning file_id For Voice"**  
   - Set variable "file_id" to `={{ $json.message.voice.file_id }}`  
   - Include other fields.  
   - Connect from Switch’s Voice output.

6. **Add Set Node "Assigning file_id For Audio"**  
   - Set variable "file_id" to `={{ $json.message.audio.file_id }}`  
   - Include other fields.  
   - Connect from Switch’s Audio output.

7. **Add Telegram Node "MSG - No file!"**  
   - Text: "No file!"  
   - Chat ID: `={{ $json.message.chat.id }}`  
   - Connect from Switch fallback output.

8. **Add IF Node "Transcription Decision Node"**  
   - Condition OR: Check if the MIME type matches any of:  
     - `audio/ogg` (voice mime_type)  
     - `audio/mpeg` (voice or audio mime_type)  
     - `audio/mp4` (audio mime_type)  
     - `audio/m4a` (audio mime_type)  
   - Expressions example:  
     - `={{ $json.message.voice.mime_type }}` equals "audio/ogg" etc.  
   - Connect both Set nodes (for voice and audio) to this node.

9. **Add Telegram Node "MSG - File not recognized"**  
   - Text: "File not recognized"  
   - Chat ID: `={{ $json.message.chat.id }}`  
   - Connect from IF node False output.

10. **Add Telegram Node "Get File GPT"**  
    - Resource: File  
    - File ID: `={{ $json.file_id }}` from Set nodes.  
    - Connect from IF node True output.

11. **Add Telegram Node "MSG - Starting transcription. Please wait."**  
    - Text: "Starting transcription. Please wait."  
    - Chat ID: `={{ $json.message.chat.id }}`  
    - Connect from "Get File GPT" node (parallel).  

12. **Add OpenAI Node "Transcription by OpenAi"**  
    - Resource: Audio  
    - Operation: Transcribe  
    - Credentials: OpenAI API (set up OAuth or API key).  
    - Input: Binary from "Get File GPT".  
    - On Error: Continue (do not fail workflow).  
    - Connect from "Get File GPT".

13. **Add Telegram Node "Get File Gemini"**  
    - Resource: File  
    - File ID: Use `={{ $('Get File GPT').item.json.result.file_id }}` from Get File GPT node output.  
    - Connect from "Transcription by OpenAi" error output.

14. **Add Google Gemini Node "Transcription by Gemini"**  
    - Model ID: "models/gemini-2.5-flash"  
    - Input Type: Binary  
    - Binary Property Name: "data"  
    - Credentials: Google Palm API.  
    - Connect from "Get File Gemini".

15. **Add Set Node "Assignment of variable TEXT"**  
    - Set variable "text" to `={{ $json.text }}` (OpenAI transcription text).  
    - Connect from "Transcription by OpenAi" success output.

16. **Add Set Node "Assignment of variable TEXT1"**  
    - Set variable "text" to `={{ $json.content.parts[0].text }}` (Gemini transcription text).  
    - Connect from "Transcription by Gemini".

17. **Add IF Node "Checking the length of a message"**  
    - Condition: Check if length of `text` is less than 4000 characters.  
    - Expression: `={{ $json["text"].length }} < 4000`  
    - Connect from both "Assignment of variable TEXT" and "TEXT1".

18. **Add Telegram Node "MSG - Output"**  
    - Text: `={{ $json.text }}`  
    - Chat ID: `={{ $json.message.chat.id }}`  
    - Connect from length check True output.

19. **Add Code Node "Text Division"**  
    - JavaScript code:  
      ```js
      const input = $json["text"];
      const chunkSize = 4000;
      const chunks = [];
      for (let i = 0; i < input.length; i += chunkSize) {
        chunks.push({ body: input.substring(i, i + chunkSize) });
      }
      return chunks.map(chunk => ({ json: chunk }));
      ```  
    - Connect from length check False output.

20. **Add Telegram Node "MSG - Output with chunking"**  
    - Text: `={{ $json.body }}`  
    - Chat ID: `={{ $json.message.chat.id }}`  
    - Connect from "Text Division".

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                            |
|-----------------------------------------------------------------------------------------------|--------------------------------------------|
| 🎙️ Voice Transcription Flow — Quick Guide: Describes flow steps and use cases.                 | Workflow Sticky Note12                      |
| Use case: For teams converting voice messages to searchable text with security and format validation. | Sticky Note12                              |
| Video Tutorial available at: https://youtu.be/ckXszQCkncM                                     | Sticky Note13                              |
| Text length chunking limit set to 4000 characters to comply with Telegram message size limits. | General Workflow Constraint                 |
| OpenAI Whisper is primary transcription engine; Gemini fallback ensures robustness.           | Workflow design principle                   |
| Telegram API credentials required with appropriate bot token and permissions.                  | Credential setup for Telegram nodes         |
| OpenAI and Google Palm API credentials required for transcription nodes.                      | Credential setup for transcription nodes    |

---

**Disclaimer:**  
This document describes an n8n workflow automating transcription of Telegram voice and audio messages with fallback transcription engines. It adheres strictly to content policies and handles only legal, public data. No raw JSON is included except where necessary for clarity.