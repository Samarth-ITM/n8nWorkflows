Telegram Chatbot with Voice Recognition and Message Batching using OpenAI

https://n8nworkflows.xyz/workflows/telegram-chatbot-with-voice-recognition-and-message-batching-using-openai-8314


# Telegram Chatbot with Voice Recognition and Message Batching using OpenAI

### 1. Workflow Overview

This workflow implements a **Telegram Chatbot with Voice Recognition and Message Batching** using OpenAI services. It is designed to receive messages from Telegram users, including voice messages, transcribe voice to text using OpenAI Whisper, batch multiple messages per user over a short period, and then process the combined input through an AI agent (OpenAI language models) before sending a response back to the user.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Classification**: Receiving Telegram messages and distinguishing between text, voice, or unsupported media.
- **1.2 Voice Message Transcription**: Downloading voice messages and transcribing them into text.
- **1.3 Message Normalization and Merging**: Normalizing all inputs to a common format and merging messages for batching.
- **1.4 Message Retention and Waiting Management**: Storing incoming messages and managing a debounce/wait mechanism to batch messages within a time window.
- **1.5 AI Processing and Reply Generation**: Feeding batched messages into an AI agent enhanced optionally with memory and vector store tools, generating AI replies.
- **1.6 Cleanup and Response Delivery**: Cleaning up temporary stored messages and sending the AI-generated response back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Classification

- **Overview:**  
  This block receives incoming Telegram messages, filters them by message type (voice vs text), and prepares the data for further processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Link Filter  
  - Pass text message  
  - Get Audio File  

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point for new Telegram messages (all message updates).  
    - *Config:* Listens for message updates from a Telegram Bot credential.  
    - *Inputs:* Telegram webhook from Telegram servers.  
    - *Outputs:* Raw Telegram message JSON.  
    - *Edge cases:* Webhook registration failure, invalid tokens, message type unsupported.  
    - *Credentials:* Telegram API token via n8n credentials.

  - **Link Filter**  
    - *Type:* If node  
    - *Role:* Checks if the incoming message has a voice file (`message.voice.file_id`) or not.  
    - *Config:*  
      - Condition 1: Checks if `message.voice.file_id` exists.  
      - Condition 2: Checks if `message.chat.id` exists (message from a chat).  
      - The logic is an AND condition.  
    - *Inputs:* From Telegram Trigger.  
    - *Outputs:*  
      - True branch: Messages with voice file → leads to "Get Audio File".  
      - False branch: Text messages → leads to "Pass text message".  
    - *Edge cases:* Missing fields, malformed messages.

  - **Pass text message**  
    - *Type:* Set node  
    - *Role:* Extracts and normalizes text messages to a standard format with properties: `processedMessage`, `sessionId`, `name`.  
    - *Config:* Extracts:  
      - `processedMessage` = text content of the message  
      - `sessionId` = chat id of the message  
      - `name` = first name of the sender  
    - *Inputs:* False output from Link Filter.  
    - *Outputs:* Standardized JSON for text messages.  
    - *Edge cases:* Missing `message.text` in JSON.

  - **Get Audio File**  
    - *Type:* Telegram node  
    - *Role:* Downloads the voice file using Telegram API based on `file_id`.  
    - *Config:* Uses `message.voice.file_id` to download the audio.  
    - *Inputs:* True output from Link Filter.  
    - *Outputs:* Binary audio file data for transcription.  
    - *Credentials:* Telegram API.  
    - *Edge cases:* File not found, API errors, timeouts.

---

#### 1.2 Voice Message Transcription

- **Overview:**  
  Transcribes downloaded audio files into text using OpenAI Whisper API and normalizes the result.

- **Nodes Involved:**  
  - Transcribe a recording  
  - Transcript Result  
  - Transcript Error Output  

- **Node Details:**

  - **Transcribe a recording**  
    - *Type:* OpenAI Audio transcribe node (Langchain OpenAI node)  
    - *Role:* Sends audio binary to OpenAI Whisper for transcription.  
    - *Config:* Operation set to "transcribe", passes audio data.  
    - *Inputs:* From "Get Audio File".  
    - *Outputs:* Two outputs:  
      - Main: transcription result  
      - On error: error object (continues workflow)  
    - *Credentials:* OpenAI API key.  
    - *Edge cases:* API quota exceeded, network errors, invalid audio format.

  - **Transcript Result**  
    - *Type:* Set node  
    - *Role:* Assigns transcription text to the field `processedMessage`, along with sessionId and user name for downstream use.  
    - *Config:*  
      - `processedMessage` = transcribed text  
      - `sessionId` and `name` extracted from original Telegram message.  
    - *Inputs:* Success output from transcription.  
    - *Outputs:* Normalized JSON for further processing.

  - **Transcript Error Output**  
    - *Type:* Set node  
    - *Role:* Captures transcription errors, assigning an `error` field with the error message and sessionId.  
    - *Inputs:* Error output from transcription node.  
    - *Outputs:* JSON with error info for merging with other message types.

---

#### 1.3 Message Normalization and Merging

- **Overview:**  
  Merges outputs from the voice transcription, text messages, and errors into a single stream with a normalized message field.

- **Nodes Involved:**  
  - Merge  
  - Edit Fields  

- **Node Details:**

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines three inputs: Transcript Result, Transcript Error Output, and Pass text message into one stream.  
    - *Config:* Number of inputs = 3, outputs all data.  
    - *Inputs:* From three nodes producing standardized JSON.  
    - *Outputs:* Unified stream with one record per message containing either processedMessage or error.

  - **Edit Fields**  
    - *Type:* Set node  
    - *Role:* Cleans and ensures the `processedMessage` field is always populated (fallback to error or empty string). Also sets `sessionId` from Telegram message.  
    - *Inputs:* From Merge node.  
    - *Outputs:* Cleaned, unified message JSON for storage.

---

#### 1.4 Message Retention and Waiting Management

- **Overview:**  
  Stores incoming messages in Google Sheets for retention and manages a waiting flag that batches messages sent within a 30-second window before processing.

- **Nodes Involved:**  
  - Append row in sheet  
  - Get row(s) in sheet1 (Message Checkup)  
  - No Rows (Code)  
  - What to do (If)  
  - Update row in sheet  
  - Update row in sheet1  
  - Wait  
  - Get row(s) in sheet (Msg Retention)  
  - Batch Messages (Code)  
  - Delete rows or columns from sheet1 (Msg Retention cleanup)  

- **Node Details:**

  - **Append row in sheet**  
    - *Type:* Google Sheets Append  
    - *Role:* Adds each processed message to "Msg Retention" sheet with columns: `date`, `user_id`, `message`.  
    - *Inputs:* From Edit Fields.  
    - *Outputs:* Confirmation and row data for further processing.  
    - *Credentials:* Google Sheets OAuth2.

  - **Get row(s) in sheet1**  
    - *Type:* Google Sheets Get Rows  
    - *Role:* Fetches existing "Msg Waiting" record for the user (by user_id) to check waiting status.  
    - *Sheet:* "Msg Waiting" with columns `user_id`, `is_waiting`, `last_updated`.  
    - *Inputs:* From Append row node.  
    - *Outputs:* User's current waiting status record.

  - **No Rows (Code)**  
    - *Type:* Code node  
    - *Role:* Normalizes the "Msg Waiting" data to always have at least one record with defaults if no rows exist (i.e., first message from user).  
    - *Logic:*  
      - If no rows, create a default `{ user_id, is_waiting:false, last_updated:null }`.  
      - Otherwise, pass existing rows with ensured keys.  
    - *Inputs:* From Get row(s) in sheet1.  
    - *Outputs:* Normalized row(s) for condition check.

  - **What to do (If)**  
    - *Type:* If node  
    - *Role:* Decides whether to start a wait/batching cycle or just update the timestamp based on:  
      - Condition 1: `last_updated` is older than 30 seconds ago  
      - Condition 2: `is_waiting` is false  
    - *Logic:* OR condition.  
    - *Inputs:* From No Rows node.  
    - *Outputs:*  
      - True: Initiate wait and batch processing.  
      - False: Only update waiting flag timestamp.

  - **Update row in sheet**  
    - *Type:* Google Sheets Update  
    - *Role:* Updates "Msg Waiting" row: sets `is_waiting=true`, updates `last_updated` to now.  
    - *Inputs:* From What to do (True branch).  
    - *Outputs:* Confirmation.

  - **Update row in sheet1**  
    - *Type:* Google Sheets Update  
    - *Role:* Updates "Msg Waiting" row similarly but on What to do (False branch) — refreshes timestamp without starting wait.  
    - *Inputs:* From What to do (False branch).

  - **Wait**  
    - *Type:* Wait node  
    - *Role:* Pauses workflow for 30 seconds to let messages accumulate before batching.  
    - *Inputs:* From Update row in sheet (True branch).  
    - *Outputs:* After wait, triggers batch processing.

  - **Get row(s) in sheet (Msg Retention)**  
    - *Type:* Google Sheets Get Rows  
    - *Role:* After wait, fetches all messages for user from "Msg Retention" to be batched.  
    - *Inputs:* From Wait node.

  - **Batch Messages**  
    - *Type:* Code node  
    - *Role:* Merges multiple messages into a single string, collects row numbers to facilitate deletion.  
    - *Logic:* Concatenates non-empty messages with spaces, returns combined message, start row number, row count, and session id.  
    - *Inputs:* From Get row(s) in sheet.

  - **Delete rows or columns from sheet1**  
    - *Type:* Google Sheets Delete Rows  
    - *Role:* Deletes the batched message rows from "Msg Retention" to keep sheet clean after batching.  
    - *Inputs:* From Batch Messages.

---

#### 1.5 AI Processing and Reply Generation

- **Overview:**  
  Takes the batched message, optionally uses AI memory and vector store for context, processes the input with an AI agent, and prepares the text response.

- **Nodes Involved:**  
  - AI Agent  
  - Simple Memory (optional)  
  - Embeddings OpenAI1 (optional)  
  - Supabase Vector Store (optional)  
  - OpenAI Chat Model (optional)  

- **Node Details:**

  - **AI Agent**  
    - *Type:* Langchain Agent node  
    - *Role:* Core AI node that consumes batched user messages and produces a reply.  
    - *Config:*  
      - Input text expression: `{{ $('Batch Messages').item.json.message }}`  
      - System prompt placeholder: "ADD YOUR PROMPT HERE" (customizable).  
      - Max retries: 3, onError continues with regular output.  
    - *Inputs:* From Delete rows or columns from sheet1 node.  
    - *Outputs:* AI-generated response text.  
    - *Edge cases:* API failures, timeouts, malformed input.

  - **Simple Memory** (optional)  
    - *Type:* Buffer window memory node  
    - *Role:* Maintains conversation history to provide context to AI Agent.  
    - *Inputs:* Connected as AI memory input to AI Agent.

  - **Embeddings OpenAI1 & Supabase Vector Store** (optional)  
    - *Type:* Embedding and vector store retrieval nodes  
    - *Role:* Provide knowledge base access to AI Agent for enhanced contextual replies.  
    - *Inputs:* Embeddings feed into vector store, which connects as a tool input to AI Agent.

  - **OpenAI Chat Model** (optional)  
    - *Type:* Chat model node  
    - *Role:* Provides a language model backend for AI Agent.  
    - *Inputs:* Connected as AI language model input to AI Agent.

---

#### 1.6 Cleanup and Response Delivery

- **Overview:**  
  Sends the AI-generated response back to Telegram user and cleans up waiting entries.

- **Nodes Involved:**  
  - Send message Back to TG  
  - Delete Row from Message Checkup  

- **Node Details:**

  - **Send message Back to TG**  
    - *Type:* Telegram node  
    - *Role:* Sends the AI agent's response text back to the Telegram chat, using the session id from batched messages.  
    - *Config:*  
      - Text: AI agent output text  
      - ChatId: session_id from batch messages  
      - Options: force reply enabled, no attribution appended.  
    - *Inputs:* From AI Agent.  
    - *Credentials:* Telegram API.  
    - *Edge cases:* Message send failures, chat id invalid.

  - **Delete Row from Message Checkup**  
    - *Type:* Google Sheets Delete Rows  
    - *Role:* Deletes the user's "Msg Waiting" row from the sheet, resetting their waiting status for the next message cycle.  
    - *Inputs:* From Send message Back to TG.  
    - *Credentials:* Google Sheets OAuth2.

---

### 3. Summary Table

| Node Name                   | Node Type                    | Functional Role                              | Input Node(s)                               | Output Node(s)                     | Sticky Note                                                                                                        |
|-----------------------------|------------------------------|----------------------------------------------|---------------------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger             | Entry point: receives Telegram messages      | -                                           | Link Filter                        | See Sticky Note2 for Telegram setup instructions                                                                   |
| Link Filter                 | If                          | Filters message type: voice vs text           | Telegram Trigger                            | Get Audio File, Pass text message  | See Sticky Note1 about transcription pipeline                                                                     |
| Pass text message            | Set                         | Normalizes text message fields                 | Link Filter (false)                         | Merge                            | See Sticky Note1 about transcription pipeline                                                                     |
| Get Audio File               | Telegram                     | Downloads voice message audio                  | Link Filter (true)                          | Transcribe a recording            | See Sticky Note1 about transcription pipeline                                                                     |
| Transcribe a recording       | OpenAI Audio Transcription   | Transcribes voice audio to text                | Get Audio File                              | Transcript Result, Transcript Error Output | See Sticky Note1 about transcription pipeline                                                                     |
| Transcript Result            | Set                         | Normalizes transcription result                | Transcribe a recording (success)            | Merge                            | See Sticky Note1 about transcription pipeline                                                                     |
| Transcript Error Output      | Set                         | Normalizes transcription error                  | Transcribe a recording (error)              | Merge                            | See Sticky Note1 about transcription pipeline                                                                     |
| Merge                       | Merge                       | Merges text, transcription, and error streams | Transcript Result, Transcript Error Output, Pass text message | Edit Fields                      |                                                                                                                    |
| Edit Fields                  | Set                         | Ensures processedMessage and sessionId fields | Merge                                      | Append row in sheet              |                                                                                                                    |
| Append row in sheet          | Google Sheets Append         | Stores messages in "Msg Retention" sheet       | Edit Fields                                | Get row(s) in sheet1             | See Sticky Note about Message Batching setup                                                                      |
| Get row(s) in sheet1         | Google Sheets Get Rows       | Retrieves "Msg Waiting" row for user           | Append row in sheet                         | No Rows                         | See Sticky Note about Message Batching setup                                                                      |
| No Rows                     | Code                        | Normalizes missing "Msg Waiting" rows           | Get row(s) in sheet1                        | What to do                      |                                                                                                                    |
| What to do                  | If                          | Determines whether to wait or update timestamp | No Rows                                   | Update row in sheet, Update row in sheet1 | See Sticky Note about Message Batching setup                                                                      |
| Update row in sheet          | Google Sheets Update         | Sets `is_waiting=true` and updates timestamp   | What to do (true)                          | Wait                           | See Sticky Note about Message Batching setup                                                                      |
| Update row in sheet1         | Google Sheets Update         | Updates timestamp only on waiting flag false  | What to do (false)                         | -                              | See Sticky Note about Message Batching setup                                                                      |
| Wait                        | Wait                        | Waits 30 seconds to batch incoming messages     | Update row in sheet                        | Get row(s) in sheet             |                                                                                                                    |
| Get row(s) in sheet          | Google Sheets Get Rows       | Retrieves messages from "Msg Retention" sheet   | Wait                                       | Batch Messages                 |                                                                                                                    |
| Batch Messages              | Code                        | Merges multiple messages into one                | Get row(s) in sheet                        | Delete rows or columns from sheet1 | See Sticky Note about Message Batching setup                                                                      |
| Delete rows or columns from sheet1 | Google Sheets Delete Rows | Deletes batched rows from "Msg Retention" sheet | Batch Messages                            | AI Agent                      |                                                                                                                    |
| AI Agent                    | Langchain Agent              | Processes batched message with AI               | Delete rows or columns from sheet1         | Send message Back to TG         | See Sticky Note4 for AI Agent setup instructions                                                                  |
| Simple Memory               | Langchain Memory Buffer      | Provides contextual memory to AI Agent          | -                                          | AI Agent                      |                                                                                                                    |
| Embeddings OpenAI1          | Langchain Embeddings OpenAI | Generates embeddings for vector store           | -                                          | Supabase Vector Store          |                                                                                                                    |
| Supabase Vector Store       | Langchain Vector Store       | Provides knowledge base retrieval tool          | Embeddings OpenAI1                         | AI Agent                      |                                                                                                                    |
| OpenAI Chat Model           | Langchain Chat Model         | Language model for AI Agent                       | -                                          | AI Agent                      |                                                                                                                    |
| Send message Back to TG      | Telegram                     | Sends AI response back to Telegram user          | AI Agent                                   | Delete Row from Message Checkup |                                                                                                                    |
| Delete Row from Message Checkup | Google Sheets Delete Rows  | Deletes "Msg Waiting" row after response sent    | Send message Back to TG                     | -                              |                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Use @BotFather in Telegram to create a new bot and obtain the API token.  
   - Add the token as a Telegram API credential in n8n.

2. **Add Telegram Trigger Node**  
   - Set "updates" to listen for "message" events.  
   - Assign Telegram API credential.  
   - Execute once to register webhook (requires public HTTPS n8n URL).

3. **Add Link Filter (If Node)**  
   - Check if `message.voice.file_id` exists (voice message).  
   - Check if `message.chat.id` exists.  
   - If true, route to voice processing; else route to text processing.

4. **Create "Pass text message" Set Node**  
   - Assign:  
     - `processedMessage` = `{{$json.message.text}}`  
     - `sessionId` = `{{$json.message.chat.id}}`  
     - `name` = `{{$json.message.from.first_name}}`  
   - Connect Link Filter false branch here.

5. **Create "Get Audio File" Telegram Node**  
   - Set fileId = `{{$json.message.voice.file_id}}`  
   - Resource = "file"  
   - Assign Telegram API credentials.  
   - Connect Link Filter true branch here.

6. **Create "Transcribe a recording" OpenAI Audio Node**  
   - Resource: Audio  
   - Operation: Transcribe  
   - Connect "Get Audio File" output here.  
   - Add OpenAI API credentials.

7. **Create "Transcript Result" Set Node**  
   - Assign:  
     - `processedMessage` = `{{$json.text}}`  
     - `sessionId` = `{{$json.message.chat.id}}` (from Telegram Message node context)  
     - `name` = `{{$json.message.from.first_name}}`  
   - Connect transcription success output here.

8. **Create "Transcript Error Output" Set Node**  
   - Assign:  
     - `error` = `{{$json.error}}`  
     - `sessionId` = `{{$json.message.chat.id}}` (from Telegram Message node context)  
   - Connect transcription error output here.

9. **Create Merge Node**  
   - Number of inputs: 3  
   - Connect:  
     - Transcript Result  
     - Transcript Error Output  
     - Pass text message

10. **Create "Edit Fields" Set Node**  
    - Assign:  
      - `processedMessage` = `{{$json.processedMessage || $json.error || ''}}`  
      - `sessionId` = `{{$json.message.from.id}}` (from Telegram Message node)  
    - Connect from Merge node.

11. **Create "Append row in sheet" Google Sheets Append Node**  
    - Document ID and Sheet Name: "Msg Retention" sheet.  
    - Columns: `date` = `{{$now}}`, `message` = `{{$json.processedMessage}}`, `user_id` = `{{$json.sessionId}}`  
    - Credentials: Google Sheets OAuth2.  
    - Connect from Edit Fields.

12. **Create "Get row(s) in sheet1" Google Sheets Get Rows Node**  
    - Sheet: "Msg Waiting"  
    - Filter: `user_id` = `{{$json.sessionId}}` from previous node  
    - Credentials: Google Sheets OAuth2.  
    - Connect from Append row node.

13. **Create "No Rows" Code Node**  
    - JavaScript code to normalize empty rows to default object with `is_waiting:false`.  
    - Connect from Get row(s) in sheet1.

14. **Create "What to do" If Node**  
    - Conditions (OR):  
      - `last_updated` is before `{{ new Date(Date.now() - 30000).toISOString() }}`  
      - `is_waiting` is false (boolean or string)  
    - Connect from No Rows.

15. **Create "Update row in sheet" Google Sheets Update Node (True branch)**  
    - Sets `is_waiting`=true, `last_updated`=$now for the user in "Msg Waiting".  
    - Connect from What to do true output.

16. **Create "Update row in sheet1" Google Sheets Update Node (False branch)**  
    - Updates timestamp only (`last_updated`=$now) without starting wait.  
    - Connect from What to do false output.

17. **Create "Wait" Node**  
    - Set wait time to 30 seconds.  
    - Connect from Update row in sheet (True branch).

18. **Create "Get row(s) in sheet" Google Sheets Get Rows Node**  
    - Sheet: "Msg Retention"  
    - Filter by `user_id` matching the session id.  
    - Connect from Wait node.

19. **Create "Batch Messages" Code Node**  
    - JavaScript code to merge all messages into a single string, returning merged message, start row number, row count, and session id.  
    - Connect from Get row(s) in sheet.

20. **Create "Delete rows or columns from sheet1" Google Sheets Delete Rows Node**  
    - Deletes rows from "Msg Retention" starting at `start_row_number` for `row_count` rows.  
    - Connect from Batch Messages.

21. **Create AI Processing Nodes:**  
    - Optional nodes for Simple Memory, Embeddings OpenAI, Supabase Vector Store, and OpenAI Chat Model depending on AI complexity.  
    - Connect all to the "AI Agent" node.

22. **Create "AI Agent" Langchain Agent Node**  
    - Text input: `{{ $('Batch Messages').item.json.message }}`  
    - Set system prompt as desired.  
    - Connect from Delete rows or columns from sheet1 node (or optionally from the AI memory or vector store nodes).  
    - Credentials: OpenAI API.

23. **Create "Send message Back to TG" Telegram Node**  
    - Text: `{{ $json.output.text }}` (AI Agent output)  
    - ChatId: `{{ $('Batch Messages').item.json.session_id }}`  
    - Credentials: Telegram API.  
    - Connect from AI Agent.

24. **Create "Delete Row from Message Checkup" Google Sheets Delete Rows Node**  
    - Deletes user's row from "Msg Waiting" sheet (cleaning waiting status).  
    - Connect from Send message Back to TG.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| **Message Batching:** Two Google Sheets are required: a "Msg Retention" sheet storing date, user_id, and message columns for incoming messages, and a "Msg Waiting" sheet that tracks user_id, is_waiting (boolean), and last_updated timestamp to manage batching and debounce waits. The batching merges multiple messages over a 30-second window before sending them as one to the AI agent.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | See Sticky Note at position [416,-304]                                                                 |
| **Transcription Pipeline:** Voice messages are downloaded from Telegram and transcribed using OpenAI Whisper. Text messages are passed through unchanged. Files/media other than text or voice are politely refused. The workflow normalizes all inputs to maintain a consistent `processedMessage` field for downstream nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | See Sticky Note at position [-752,-96]                                                                 |
| **Telegram Setup:** Create a Telegram bot with @BotFather, add credentials in n8n, and ensure webhook registration by executing the Telegram Trigger node once. Enable the "message" update type. Share Google Sheets with the service account email from your Google Sheets credential for editing rights.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | See Sticky Note at position [-1296,-864]                                                               |
| **Common Pitfalls:** Ensure webhook is registered and n8n URL is HTTPS and public. Google Sheets permissions must allow edits by the service account to avoid silent failures. Use ISO date formats for date comparisons. Avoid duplicate wait timers by only triggering wait on the true condition branch; the false branch should only refresh timestamps. Clean up batched messages from retention sheet before sending replies and delete the waiting row last. Use loose type validation in IF nodes for boolean fields stored as strings. Voice transcripts require proper filtering to ensure text and voice channels feed correctly into the merge node.                                                                                                                                            | See Sticky Note at position [416,-688]                                                                 |
| **AI Agent Node Setup:** The AI Agent node is the core component that receives batched messages. A system prompt must be defined to control the AI's behavior. Optionally, connect memory and vector store nodes to enhance context and knowledge base access. Use the expression `{{ $('Batch Messages').item.json.message }}` for the user input text.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | See Sticky Note at position [2720,-480]                                                                |

---

This completes the comprehensive reference documentation for the "Telegram Chatbot with Voice Recognition and Message Batching using OpenAI" workflow. It enables reproduction, modification, and troubleshooting for advanced users and automation agents.