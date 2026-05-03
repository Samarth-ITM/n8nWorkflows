Support Call Analysis & Routing with Gemini AI, ElevenLabs & Telegram Alerts

https://n8nworkflows.xyz/workflows/support-call-analysis---routing-with-gemini-ai--elevenlabs---telegram-alerts-8151


# Support Call Analysis & Routing with Gemini AI, ElevenLabs & Telegram Alerts

### 1. Workflow Overview

This workflow automates the processing, analysis, and routing of support call recordings. It is designed for customer support centers or companies that want to analyze call transcripts to extract key insights, identify client sentiment, and trigger appropriate notifications or logging actions. The workflow integrates Google Drive for file retrieval and storage, ElevenLabs for speech-to-text transcription with speaker diarization, Google Gemini AI for advanced conversation analysis, and Telegram for alerting relevant teams.

Logical blocks:

- **1.1 Scheduled Input Fetching:** Periodically checks a designated Google Drive folder for new audio call recordings.
- **1.2 Audio Download and Speech-to-Text Conversion:** Downloads audio files and converts them into multi-speaker conversational transcripts using ElevenLabs API.
- **1.3 Conversational Transcript Formatting:** Code node reformats raw transcription into readable speaker-separated text.
- **1.4 AI-Powered Call Analysis:** Uses Google Gemini AI with a structured output parser to analyze the transcript and extract speaker identities, sentiment, topics, department tags, and action items.
- **1.5 Logging and Routing Based on Sentiment:** Logs analyzed call data to Google Sheets, moves processed files to an archive folder, and routes alerts through Telegram based on client sentiment (positive or negative).

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Input Fetching

- **Overview:**  
  Periodically triggers the workflow every minute and searches a specific Google Drive folder for new `.wav` support call recordings.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Search For New Call Recordings

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Periodic start trigger for the workflow every 1 minute.  
    - Configuration: Interval set to 1 minute.  
    - Input: None (trigger node).  
    - Output: Triggers "Search For New Call Recordings".  
    - Edge Cases: Missed triggers if workflow execution time exceeds interval, potential timing conflicts.

  - **Search For New Call Recordings**  
    - Type: Google Drive (fileFolder resource)  
    - Role: Searches in a specific Google Drive folder ("Company - Support Call Recordings") for `.wav` audio files.  
    - Configuration: Query filters for files with MIME type `audio/wav` in folder `1Kz8dGr8Hful3CeUJWjsbxktAEih3-Xjt`.  
    - Credentials: Google Drive OAuth2.  
    - Input: Trigger from Schedule Trigger.  
    - Output: Passes metadata of found audio files to "Download Audio Files".  
    - Edge Cases: Folder permission errors, API rate limits, empty folder (no new files), file listing delays.

---

#### 2.2 Audio Download and Speech-to-Text Conversion

- **Overview:**  
  Downloads each audio file found and sends it to ElevenLabs' Speech-to-Text API with diarization enabled to generate detailed multi-speaker transcripts.

- **Nodes Involved:**  
  - Download Audio Files  
  - Convert Speech To Text  
  - Sticky Note (description)

- **Node Details:**  

  - **Download Audio Files**  
    - Type: Google Drive (file download)  
    - Role: Downloads the audio file binary data for transcription.  
    - Configuration: Uses file ID from previous node.  
    - Credentials: Google Drive OAuth2.  
    - Input: List of audio files from "Search For New Call Recordings".  
    - Output: Binary audio data to "Convert Speech To Text".  
    - Edge Cases: File access errors, download failures, large file timeouts.

  - **Convert Speech To Text**  
    - Type: HTTP Request  
    - Role: Sends audio binary to ElevenLabs API for speech-to-text transcription with speaker diarization.  
    - Configuration:  
      - POST to `https://api.elevenlabs.io/v1/speech-to-text`  
      - Multipart form-data with file and parameters: `diarize=true`, `model_id=scribe_v1`  
      - Auth: HTTP Header Auth with ElevenLabs API key.  
    - Input: Audio file binary from "Download Audio Files".  
    - Output: JSON response containing array of word objects including speaker IDs.  
    - Edge Cases: API authentication errors, API rate limits, malformed audio data, network timeouts.

  - **Sticky Note**  
    - Content: Describes this block as converting call recordings into conversational multi-speaker transcripts.

---

#### 2.3 Conversational Transcript Formatting

- **Overview:**  
  Converts the detailed word-level transcription with speaker IDs into a readable multi-line transcript grouped by speaker turns.

- **Nodes Involved:**  
  - Convert to Conversational Transcribe

- **Node Details:**  

  - **Convert to Conversational Transcribe**  
    - Type: Code (JavaScript)  
    - Role: Processes the array of words with speaker_id fields, concatenates words per speaker turn, and produces a formatted transcript string.  
    - Configuration: Custom JS code loops through words, detects speaker changes, formats speaker labels (`Speaker 0`, `Speaker 1`), and builds transcript lines.  
    - Input: JSON from ElevenLabs containing array `words`.  
    - Output: JSON with `formattedTranscript` string for next step.  
    - Edge Cases: Missing or empty words array, incorrect speaker_id format, transcript formatting bugs.

---

#### 2.4 AI-Powered Call Analysis

- **Overview:**  
  Uses Google Gemini AI to analyze the formatted transcript and extract structured data about speakers, sentiment, topics, departments, and actionable items.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - Call Analyze

- **Node Details:**  

  - **Google Gemini Chat Model**  
    - Type: Langchain Google Gemini AI model node  
    - Role: Sends prompt and transcript to Gemini AI for natural language understanding.  
    - Configuration: Default options; credentials set for Google PaLM API.  
    - Input: From "Convert to Conversational Transcribe" (via Call Analyze).  
    - Output: Raw AI chat response to "Structured Output Parser".  
    - Edge Cases: API quota limits, invalid credentials, API errors, latency.

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Validates and parses AI response to ensure JSON conforms to required schema.  
    - Configuration: JSON schema example provided, includes fields for speaker IDs, names, summary, sentiment, topic, department tag, action items.  
    - Input: From Google Gemini Chat Model.  
    - Output: Parsed structured JSON to "Call Analyze".  
    - Edge Cases: Schema mismatch, invalid JSON from AI, parsing errors.

  - **Call Analyze**  
    - Type: Langchain Agent  
    - Role: Coordinates the AI prompt and output parsing; formats the prompt with system instructions and transcript, collects structured JSON results.  
    - Configuration:  
      - System message instructs AI to analyze call transcript for QA, identify names, assign department tag from fixed list, output strict JSON only.  
      - Prompt includes formatted transcript.  
      - Output parser enabled with schema.  
    - Input: From "Convert to Conversational Transcribe".  
    - Output: Structured analysis JSON to logging and routing nodes.  
    - Edge Cases: AI misinterpretation, output format errors.

---

#### 2.5 Logging and Routing Based on Sentiment

- **Overview:**  
  Logs the call analysis results to Google Sheets, moves processed audio files to an archive folder, and sends Telegram alerts to managers or support team channels based on client sentiment.

- **Nodes Involved:**  
  - Log Recording Analysis  
  - Move Audio To Processed Folder  
  - Client Sentiment (Switch)  
  - Send Alert To Managers  
  - Send Kudos to Team  
  - Sticky Note (description)

- **Node Details:**  

  - **Log Recording Analysis**  
    - Type: Google Sheets (append row)  
    - Role: Appends analyzed call data including topic, agent/client names, action items, summary, full transcript, sentiment, department, and recording filename to a Google Sheet.  
    - Configuration: Mapping fields from analysis JSON and transcript, sheet `gid=0`.  
    - Credentials: Google Sheets OAuth2.  
    - Input: From "Call Analyze" output.  
    - Output: Triggers "Move Audio To Processed Folder" and "Client Sentiment" nodes.  
    - Edge Cases: Sheet permission errors, rate limits, data mapping issues.

  - **Move Audio To Processed Folder**  
    - Type: Google Drive (move file)  
    - Role: Moves the audio file to a "Processed Audio" Google Drive folder to avoid reprocessing.  
    - Configuration: Uses file ID from "Download Audio Files", target folder ID `1MDRPnlY6WYFwpR7WgNg2yhriq75gH8Bc`.  
    - Credentials: Google Drive OAuth2.  
    - Input: From "Log Recording Analysis".  
    - Output: None (end node).  
    - Edge Cases: Move permission errors, conflicts if file missing.

  - **Client Sentiment**  
    - Type: Switch node  
    - Role: Routes workflow based on the client's sentiment value from analysis JSON.  
    - Configuration:  
      - Checks if sentiment equals "Negative" → route to "Send Alert To Managers".  
      - Checks if sentiment equals "Positive" → route to "Send Kudos to Team".  
      - Neutral or other sentiments have no route (no further action).  
    - Input: From "Log Recording Analysis".  
    - Output: Routes to Telegram alert nodes.  
    - Edge Cases: Missing sentiment value, unexpected sentiment strings.

  - **Send Alert To Managers**  
    - Type: Telegram  
    - Role: Sends formatted alert message to managers' Telegram group for negative sentiment calls.  
    - Configuration:  
      - Chat ID: `-4832906342` (managers group).  
      - Message includes emoji alert, extracted agent/client names, topic, summary, and action items.  
      - No attribution appended.  
    - Credentials: Telegram API.  
    - Input: From "Client Sentiment" switch.  
    - Output: None.  
    - Edge Cases: Telegram API failures, invalid chat ID.

  - **Send Kudos to Team**  
    - Type: Telegram  
    - Role: Sends positive reinforcement message to support team Telegram channel for positive sentiment calls.  
    - Configuration:  
      - Chat ID: `-1003053970420` (support team channel).  
      - Message includes congratulatory note with agent name and topic.  
      - No attribution appended.  
    - Credentials: Telegram API.  
    - Input: From "Client Sentiment" switch.  
    - Output: None.  
    - Edge Cases: Telegram API failures, invalid chat ID.

  - **Sticky Note**  
    - Content: Explains that this block evaluates sentiment and sends Telegram notifications accordingly to either managers or support teams.

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                                          | Input Node(s)                 | Output Node(s)                          | Sticky Note                                                                                         |
|-----------------------------|-----------------------------------------|----------------------------------------------------------|-------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                        | Periodic trigger every 1 minute                           | None                          | Search For New Call Recordings          |                                                                                                   |
| Search For New Call Recordings | Google Drive (fileFolder)               | Search for new `.wav` call recordings in Drive folder    | Schedule Trigger              | Download Audio Files                    | ## New Call Recordings<br>Check the designated Google Drive folder for new call recordings and prepare them in binary format. |
| Download Audio Files        | Google Drive (file download)             | Download audio file binary                                | Search For New Call Recordings | Convert Speech To Text                  |                                                                                                   |
| Convert Speech To Text      | HTTP Request                            | Convert audio to text with speaker diarization via ElevenLabs API | Download Audio Files           | Convert to Conversational Transcribe    | ## Speech To Text<br>Convert the call recording into a conversational multi-speaker transcript.   |
| Convert to Conversational Transcribe | Code (JavaScript)                        | Format raw words array into readable multi-speaker transcript | Convert Speech To Text         | Call Analyze                           |                                                                                                   |
| Google Gemini Chat Model    | Langchain Google Gemini AI               | AI model for advanced call analysis                       | Call Analyze                  | Structured Output Parser                |                                                                                                   |
| Structured Output Parser    | Langchain Structured Output Parser       | Parse and validate AI JSON output                         | Google Gemini Chat Model       | Call Analyze                           |                                                                                                   |
| Call Analyze               | Langchain Agent                         | Coordinate AI prompt and parsing, output structured JSON | Convert to Conversational Transcribe + Structured Output Parser | Log Recording Analysis                   |                                                                                                   |
| Log Recording Analysis     | Google Sheets (append)                   | Log analyzed call data to Google Sheets                   | Call Analyze                  | Move Audio To Processed Folder, Client Sentiment |                                                                                                   |
| Move Audio To Processed Folder | Google Drive (move file)                  | Move audio file to processed folder                       | Log Recording Analysis         | None                                   |                                                                                                   |
| Client Sentiment            | Switch                                  | Route based on client sentiment                            | Log Recording Analysis         | Send Alert To Managers, Send Kudos to Team | ## Take Action<br>Evaluate the client’s sentiment. If it’s positive, send a kudos message to the support team’s Telegram channel. If it’s negative, notify the managers’ Telegram group to take the necessary actions. |
| Send Alert To Managers      | Telegram                                | Send alert message to managers for negative sentiment    | Client Sentiment              | None                                   |                                                                                                   |
| Send Kudos to Team          | Telegram                                | Send kudos message to support team for positive sentiment | Client Sentiment              | None                                   |                                                                                                   |
| Sticky Note                | Sticky Note                             | Workflow documentation notes                              | None                          | None                                   | See specific notes in blocks above.                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to every 1 minute.

2. **Add Google Drive node (Search For New Call Recordings)**  
   - Resource: fileFolder  
   - Operation: Search by query  
   - Query: `mimeType = 'audio/wav'`  
   - Folder ID: `1Kz8dGr8Hful3CeUJWjsbxktAEih3-Xjt` (replace with your folder)  
   - Connect Schedule Trigger → Search For New Call Recordings  
   - Use Google Drive OAuth2 credentials.

3. **Add Google Drive node (Download Audio Files)**  
   - Operation: Download file  
   - File ID: Set dynamically from previous node (`{{$json["id"]}}`)  
   - Connect Search For New Call Recordings → Download Audio Files  
   - Use same Google Drive OAuth2 credentials.

4. **Add HTTP Request node (Convert Speech To Text)**  
   - Method: POST  
   - URL: `https://api.elevenlabs.io/v1/speech-to-text`  
   - Authentication: HTTP Header Auth (ElevenLabs API key)  
   - Content Type: multipart/form-data  
   - Body Parameters:  
     - `file`: binary data from Download Audio Files node  
     - `diarize`: true  
     - `model_id`: `scribe_v1`  
   - Connect Download Audio Files → Convert Speech To Text

5. **Add Code node (Convert to Conversational Transcribe)**  
   - Mode: Run once per item  
   - Paste the provided JavaScript code that formats the word array into a multi-speaker transcript.  
   - Connect Convert Speech To Text → Convert to Conversational Transcribe

6. **Add Langchain Agent node (Call Analyze)**  
   - Set prompt type: Define prompt  
   - System message: Provide instructions for call analysis, speaker identification, department tagging, and JSON-only output.  
   - Text input: Inject `{{ $json.formattedTranscript }}`  
   - Enable output parser.  
   - Connect Convert to Conversational Transcribe → Call Analyze  
   - Use Google Gemini (PaLM) API credentials.

7. **Add Langchain Google Gemini Chat Model node**  
   - Use default options.  
   - Connect Call Analyze → Google Gemini Chat Model (ai_languageModel input)

8. **Add Langchain Structured Output Parser node**  
   - Paste JSON schema example describing expected output structure.  
   - Connect Google Gemini Chat Model → Structured Output Parser (ai_outputParser input)  
   - Connect Structured Output Parser → Call Analyze (ai_outputParser output)

9. **Add Google Sheets node (Log Recording Analysis)**  
   - Operation: Append row  
   - Document ID: your Google Sheet ID  
   - Sheet Name: appropriate sheet or gid=0  
   - Map columns to fields from Call Analyze output: topic, agent name, client name, action items (joined with bullets), summary, full transcript (with speaker names replaced), sentiment, department, recording filename.  
   - Connect Call Analyze → Log Recording Analysis  
   - Use Google Sheets OAuth2 credentials.

10. **Add Google Drive node (Move Audio To Processed Folder)**  
    - Operation: Move file  
    - File ID: from Download Audio Files node (`{{$json["id"]}}`)  
    - Folder ID: your archive folder ID for processed files  
    - Connect Log Recording Analysis → Move Audio To Processed Folder  
    - Use Google Drive OAuth2 credentials.

11. **Add Switch node (Client Sentiment)**  
    - Check `{{ $json.output.client_sentiment }}`  
    - Condition 1: Equals "Negative" → output "Negative"  
    - Condition 2: Equals "Positive" → output "Positive"  
    - Connect Log Recording Analysis → Client Sentiment

12. **Add Telegram node (Send Alert To Managers)**  
    - Chat ID: managers group chat ID  
    - Message: Formatted alert with emoji, agent/client names, topic, summary, and action items using expressions to extract from Call Analyze output.  
    - Connect Client Sentiment "Negative" output → Send Alert To Managers  
    - Add Telegram API credentials.

13. **Add Telegram node (Send Kudos to Team)**  
    - Chat ID: support team channel ID  
    - Message: Congratulatory message with agent and topic from Call Analyze output.  
    - Connect Client Sentiment "Positive" output → Send Kudos to Team  
    - Add Telegram API credentials.

14. **Add Sticky Notes** for documentation purposes in n8n UI to describe each major block as per above.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow integrates Google Drive, ElevenLabs, Google Gemini AI (PaLM), Google Sheets, and Telegram for end-to-end support call processing and alerting.                                                                             | General architecture overview.                                                                     |
| Google Gemini AI is used with Langchain nodes for advanced natural language understanding and structured output parsing.                                                                                                           | See Langchain documentation and Google PaLM API references.                                        |
| ElevenLabs Speech-to-Text API with diarization enabled allows speaker-separated transcription for multi-speaker calls.                                                                                                             | ElevenLabs API docs: https://docs.elevenlabs.io/api/speech-to-text                                 |
| Telegram nodes require bot token with chat permissions for channels and groups used.                                                                                                                                                | Telegram Bot API info: https://core.telegram.org/bots/api                                          |
| Replacing speaker IDs with names or labels improves transcript readability for human review and downstream AI processing.                                                                                                         | Custom JS code in "Convert to Conversational Transcribe" node.                                     |
| The workflow assumes `.wav` audio format input files in a specific Google Drive folder and moves processed files to an archive folder to prevent reprocessing.                                                                      | Folder IDs must be updated for each deployment.                                                    |
| Sentiment routing uses a switch node with strict string matching; unexpected sentiment values may require additional handling or default routes.                                                                                   | Could be extended to handle Neutral or unknown sentiments.                                         |
| Logs all data into a Google Sheets document for audit, reporting, or further analysis.                                                                                                                                                | Google Sheets must have columns matching the mapped fields.                                       |
| Sticky notes in the workflow provide helpful explanations and context for maintenance and collaboration.                                                                                                                           | Use n8n UI features to add or update these notes as needed.                                       |

---

**Disclaimer:** The content above is derived solely from an n8n automated workflow. It complies strictly with content policies and does not contain any illegal, offensive, or protected material. All data handled is legal and publicly accessible.