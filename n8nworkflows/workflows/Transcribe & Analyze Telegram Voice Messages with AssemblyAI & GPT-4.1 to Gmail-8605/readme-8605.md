Transcribe & Analyze Telegram Voice Messages with AssemblyAI & GPT-4.1 to Gmail

https://n8nworkflows.xyz/workflows/transcribe---analyze-telegram-voice-messages-with-assemblyai---gpt-4-1-to-gmail-8605


# Transcribe & Analyze Telegram Voice Messages with AssemblyAI & GPT-4.1 to Gmail

### 1. Workflow Overview

This workflow automates the process of capturing voice messages sent to a Telegram bot, transcribing the audio via AssemblyAI, analyzing the transcript with OpenAI’s GPT-4.1 model for summary and sentiment, and finally sending a formatted email report via Gmail. It is designed primarily for SMBs or users who prefer recording meetings or communications as voice notes on Telegram instead of using traditional conferencing tools.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Telegram Voice Message Trigger and routing to distinguish voice vs text messages.
- **1.2 Audio Processing & Transcription:** Download voice file, upload to AssemblyAI, request transcription, and poll until completion.
- **1.3 AI Transcript Analysis:** Use OpenAI GPT-4.1 to analyze the transcript for summary, sentiment, key points, action items, quotes, and topics.
- **1.4 Email Delivery:** Format and send the analysis results as a structured email via Gmail.
- **1.5 Control Flow & Utilities:** Includes waiting nodes and conditional logic to manage asynchronous transcription and error conditions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures incoming Telegram messages via webhook, differentiates voice messages from text messages, and routes accordingly.

**Nodes Involved:**  
- Voice Message (Telegram Trigger)  
- Route: Text vs Audio (Switch)  
- Sticky Note (Execution Flow from Telegram)  

**Node Details:**  

- **Voice Message**  
  - Type: Telegram Trigger  
  - Role: Listens for Telegram updates, specifically new messages sent to the bot.  
  - Configuration: Listens to "message" updates; requires Telegram Bot API credentials.  
  - Input: Telegram webhook event.  
  - Output: JSON payload containing message data, including potential voice message or text content.  
  - Failures: Network errors, invalid webhook setup, or incorrect bot token can cause failure.  

- **Route: Text vs Audio**  
  - Type: Switch  
  - Role: Checks if the incoming message contains a voice file (`message.voice.file_id`) or text (`message.text`) and routes accordingly.  
  - Configuration: Two outputs, "Voice" and "Text message", using existence checks on JSON fields.  
  - Input: Telegram message JSON from trigger node.  
  - Output: Routes to either voice processing or (not implemented here) text message path.  
  - Edge Cases: Messages without voice or text will not match either route.

- **Sticky Note (Execution Flow from Telegram)**  
  - Provides a high-level annotation about the Telegram voice message reception step.  

---

#### 2.2 Audio Processing & Transcription

**Overview:**  
Downloads the voice message file from Telegram, uploads the audio to AssemblyAI, requests a transcript, and waits until the transcription is completed.

**Nodes Involved:**  
- Get a file (Telegram)  
- Upload Audio to Assembly AI (HTTP Request)  
- Request Transcript from AssemblyAI2 (HTTP Request)  
- Wait (Delay node)  
- Get Transcript (HTTP Request)  
- If (Condition)  
- Sticky Notes (Assembly AI annotations)  

**Node Details:**  

- **Get a file**  
  - Type: Telegram Node (file resource)  
  - Role: Downloads the voice message file from Telegram servers using the `file_id`.  
  - Configuration: Uses dynamic expression to get `file_id` from incoming JSON. Requires Telegram API credentials.  
  - Input: Routed from "Voice" output of the switch node.  
  - Output: Binary data of the audio file, passed to next node.  
  - Failures: Invalid file ID, expired file, or Telegram API errors.

- **Upload Audio to Assembly AI**  
  - Type: HTTP Request (POST)  
  - Role: Uploads the binary audio data to AssemblyAI's upload endpoint to get a temporary URL.  
  - Configuration: Sends binary data from previous node; includes AssemblyAI API key in headers.  
  - Input: Binary audio data from "Get a file".  
  - Output: JSON containing `upload_url` for the audio.  
  - Failures: Authentication error, network timeout, invalid binary data.

- **Request Transcript from AssemblyAI2**  
  - Type: HTTP Request (POST)  
  - Role: Initiates transcription request with AssemblyAI using the uploaded audio URL.  
  - Configuration: Sends JSON body with `audio_url` set from upload response; requires AssemblyAI API key in header.  
  - Input: JSON from upload node containing `upload_url`.  
  - Output: JSON with transcription job `id`.  
  - Failures: Invalid API key, malformed request, rate limiting.

- **Wait**  
  - Type: Wait node  
  - Role: Delays for 10 seconds to allow transcription processing.  
  - Configuration: Fixed 10-second delay.  
  - Input: Triggered after transcription request.  
  - Output: Triggers next polling step.  
  - Edge Cases: May need adjustment if transcript takes longer.

- **Get Transcript**  
  - Type: HTTP Request (GET)  
  - Role: Polls AssemblyAI for transcript status and content using transcription job `id`.  
  - Configuration: GET request to transcript endpoint with `id` in URL; API key in headers.  
  - Input: From Wait node.  
  - Output: JSON with transcription status and text.  
  - Failures: Job not found, authorization error, incomplete transcript.

- **If**  
  - Type: Conditional  
  - Role: Checks if transcript status equals "completed".  
  - Configuration: Condition on `$json.status === "completed"`.  
  - Input: Transcript JSON from previous node.  
  - Output: If true, proceeds to AI analysis; else loops back to Wait node for retry.  
  - Edge Cases: Long transcriptions requiring multiple retries.

- **Sticky Notes**  
  - Provide context about AssemblyAI usage.

---

#### 2.3 AI Transcript Analysis

**Overview:**  
Analyzes the completed transcript using OpenAI GPT-4.1 to generate a detailed summary, sentiment analysis, key points, action items, notable quotes, and topics in strict JSON format.

**Nodes Involved:**  
- Transcript Analysis (AI)  
- Sticky Note3 (AI Analysis annotation)  

**Node Details:**  

- **Transcript Analysis (AI)**  
  - Type: OpenAI Node (LangChain)  
  - Role: Sends transcript text to GPT-4.1 with a prompt to produce structured JSON output including summary and sentiment.  
  - Configuration: Uses model "gpt-4.1-mini", system prompt instructing JSON-only output with keys like summary, sentiment_label, sentiment_score, key_points, action_items, notable_quotes, topics.  
  - Input: Transcript text from "Get Transcript" node’s JSON property `text`.  
  - Output: JSON object parsed from AI response.  
  - Failures: API key issues, timeout, prompt misformatting, or malformed AI response.

---

#### 2.4 Email Delivery

**Overview:**  
Formats the AI analysis results into a styled HTML email and sends it to a configured email address using Gmail.

**Nodes Involved:**  
- Send Analysis Email  
- The End  
- Sticky Note3 (also related)  

**Node Details:**  

- **Send Analysis Email**  
  - Type: Gmail Node  
  - Role: Sends an email with the transcript analysis formatted as an HTML table including summary, sentiment, key points, action items, notable quotes, and topics.  
  - Configuration:  
    - `sendTo`: User-configured recipient email.  
    - `subject`: Includes current timestamp.  
    - `message`: Dynamically generated HTML using JavaScript expression referencing AI node output.  
    - Requires Gmail OAuth2 credentials.  
  - Input: AI JSON output from "Transcript Analysis (AI)".  
  - Output: Email sent confirmation.  
  - Failures: OAuth token expiration, invalid recipient address, Gmail API quota limits.

- **The End**  
  - Type: NoOp  
  - Role: Marks the end of the workflow chain.  
  - Input: From Send Email node.  
  - Output: None.

---

#### 2.5 Control Flow & Utilities

**Overview:**  
Manages workflow timing, retries, and helps annotate the workflow with sticky notes.

**Nodes Involved:**  
- Wait (for polling)  
- If (check transcript status)  
- Multiple Sticky Notes (documenting workflow steps)  

**Node Details:**  
- Wait and If nodes manage asynchronous transcription polling with retry logic.  
- Sticky Notes provide instructions, usage information, and branding details.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                               | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                                      |
|----------------------------|----------------------------|-----------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Voice Message              | Telegram Trigger           | Capture Telegram voice/text messages           | —                            | Route: Text vs Audio          | # Execution flow from Telegram: Send a voice message on Telegram                                                |
| Route: Text vs Audio       | Switch                    | Route messages into voice or text processing   | Voice Message                | Get a file                   |                                                                                                                 |
| Get a file                | Telegram (file resource)    | Download audio file from Telegram               | Route: Text vs Audio         | Upload Audio to Assembly AI   | # Generate Transcript from Voice to text                                                                        |
| Upload Audio to Assembly AI | HTTP Request               | Upload audio to AssemblyAI for transcription   | Get a file                  | Request Transcript from AssemblyAI2 | ### Assembly AI                                                                                                  |
| Request Transcript from AssemblyAI2 | HTTP Request               | Request transcription job from AssemblyAI      | Upload Audio to Assembly AI  | Wait                        | ### Assembly AI                                                                                                  |
| Wait                      | Wait                      | Delay for transcription processing             | Request Transcript from AssemblyAI2 | Get Transcript            | ### Assembly AI                                                                                                  |
| Get Transcript            | HTTP Request               | Poll AssemblyAI for transcript status & text   | Wait                        | If                          |                                                                                                                 |
| If                        | Conditional               | Check if transcript status is "completed"      | Get Transcript              | Transcript Analysis (AI), Wait |                                                                                                                 |
| Transcript Analysis (AI)  | OpenAI (LangChain)         | Analyze transcript text via GPT-4.1             | If                         | Send Analysis Email          | # Generate summary, Sentiment Analysis and send E-mail                                                        |
| Send Analysis Email       | Gmail                     | Send formatted transcript analysis report email | Transcript Analysis (AI)    | The End                     |                                                                                                                 |
| The End                   | NoOp                      | Marks workflow completion                        | Send Analysis Email          | —                            |                                                                                                                 |
| Sticky Note (Execution Flow) | Sticky Note               | Documentation                                   | —                            | —                            | # Execution flow from Telegram                                                                                   |
| Sticky Note2              | Sticky Note               | Documentation                                   | —                            | —                            | # Generate Transcript from Voice to text                                                                         |
| Sticky Note3              | Sticky Note               | Documentation                                   | —                            | —                            | # Generate summary, Sentiment Analysis and send E-mail                                                        |
| Sticky Note5              | Sticky Note               | Documentation                                   | —                            | —                            | ### Assembly AI                                                                                                  |
| Sticky Note6              | Sticky Note               | Documentation                                   | —                            | —                            | ### Assembly AI                                                                                                  |
| Sticky Note1              | Sticky Note               | Usage instructions and overview                 | —                            | —                            | # Try it Out! Capture voice messages from Telegram, transcribe, analyze, and email report. Setup instructions. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates.  
   - Set credentials with your Telegram Bot API token.  
   - Position: Start of workflow.

2. **Add Switch Node ("Route: Text vs Audio")**  
   - Create two outputs: "Voice" and "Text message".  
   - For "Voice": Condition is that `message.voice.file_id` exists.  
   - For "Text message": Condition is that `message.text` exists.  
   - Connect Telegram Trigger output to this node.

3. **Add Telegram Node ("Get a file")**  
   - Type: Telegram node with resource = file.  
   - Set `fileId` parameter to `{{$json.message.voice.file_id}}`.  
   - Connect from "Voice" output of the Switch node.  
   - Use Telegram credentials.

4. **Add HTTP Request Node ("Upload Audio to Assembly AI")**  
   - Method: POST  
   - URL: `https://api.assemblyai.com/v2/upload`  
   - Set to send binary data from previous node (inputDataFieldName = "data").  
   - Add headers: `authorization` with your AssemblyAI API key, `Content-Type` as `application/json`.  
   - Connect from "Get a file".

5. **Add HTTP Request Node ("Request Transcript from AssemblyAI2")**  
   - Method: POST  
   - URL: `https://api.assemblyai.com/v2/transcript`  
   - Body Parameters: JSON with key `audio_url` set to `{{$json.upload_url}}`.  
   - Headers as above for AssemblyAI API key and content-type.  
   - Connect from "Upload Audio to Assembly AI".

6. **Add Wait Node ("Wait")**  
   - Duration: 10 seconds (or appropriate delay).  
   - Connect from "Request Transcript from AssemblyAI2".

7. **Add HTTP Request Node ("Get Transcript")**  
   - Method: GET  
   - URL: `https://api.assemblyai.com/v2/transcript/{{$json.id}}`  
   - Headers: Authorization with AssemblyAI API key, Content-Type `application/json`.  
   - Connect from "Wait".

8. **Add If Node ("If")**  
   - Condition: Check if `$json.status` equals `"completed"`.  
   - Connect from "Get Transcript".  
   - True branch goes to AI analysis, False branch loops back to "Wait" node for polling.

9. **Add OpenAI Node ("Transcript Analysis (AI)")**  
   - Use LangChain OpenAI node.  
   - Model: `gpt-4.1-mini` or equivalent GPT-4.1 model.  
   - System prompt: Instruct to analyze transcript text (`{{$node["Get Transcript"].json.text}}`) and output strict JSON with keys: summary, sentiment_label, sentiment_score, key_points, action_items, notable_quotes, topics.  
   - Enable JSON output parsing.  
   - Connect from "If" True branch.

10. **Add Gmail Node ("Send Analysis Email")**  
    - Set recipient email address (`sendTo`), e.g., your own email.  
    - Subject: Include timestamp, e.g., `"Transcript summary & sentiment – {{$now.toISO()}}"`.  
    - Message: Use JavaScript expression to format AI JSON output into an HTML table with sections for summary, sentiment, key points, action items, notable quotes, and topics.  
    - Use Gmail OAuth2 credentials.  
    - Connect from OpenAI node.

11. **Add NoOp Node ("The End")**  
    - Connect from Gmail node to mark workflow completion.

12. **Add Sticky Notes for Documentation (Optional)**  
    - Add descriptive sticky notes at key points for clarity.

13. **Save and Activate Workflow**  
    - Test by sending a voice message to your Telegram bot.  
    - Monitor execution and logs for errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                               | Context or Link                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| This workflow is ideal for SMBs that use Telegram voice messages instead of Zoom or Teams for meetings. It automates transcription, AI analysis, and sends a structured report by email.                                                                                     | Workflow description                        |
| Requires Telegram Bot API token, AssemblyAI API key, OpenAI API key, and Gmail OAuth2 credentials set up in n8n.                                                                                                                                                           | Credentials setup                           |
| For help or community support, visit n8n Discord at https://discord.gg/n8n or n8n Forum at https://community.n8n.io                                                                                                                                                      | Support links                              |
| The AI prompt is designed to produce strict JSON output with keys in snake_case for easy parsing and reliable automation downstream.                                                                                                                                       | Node "Transcript Analysis (AI)" prompt     |
| Gmail email content is carefully styled with HTML for readability including tables, lists, and sections for easy insight consumption.                                                                                                                                        | "Send Analysis Email" node message field  |
| Adjust the Wait node duration if transcription jobs take longer or shorter to complete based on audio length and AssemblyAI performance.                                                                                                                                   | Polling strategy                            |
| Make sure to replace placeholders like `<<YOUR BOT NAME>>`, `<<YOUR_ASSEMBLYAI_API_KEY>>`, `<<YOUR_EMAIL ID>>` with your actual credentials and addresses before running.                                                                                                   | Credential placeholders                     |

---

**Disclaimer:** The provided content is derived exclusively from an automated n8n workflow. It complies fully with applicable content policies and contains no illegal or offensive elements. All processed data is legal and publicly accessible.