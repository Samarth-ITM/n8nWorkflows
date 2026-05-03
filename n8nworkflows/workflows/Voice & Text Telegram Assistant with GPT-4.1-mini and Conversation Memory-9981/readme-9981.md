Voice & Text Telegram Assistant with GPT-4.1-mini and Conversation Memory

https://n8nworkflows.xyz/workflows/voice---text-telegram-assistant-with-gpt-4-1-mini-and-conversation-memory-9981


# Voice & Text Telegram Assistant with GPT-4.1-mini and Conversation Memory

---

### 1. Workflow Overview

This workflow implements a **Personal AI Assistant on Telegram** that handles both voice and text messages using GPT-4.1-mini with conversation memory. It is designed to:

- Receive messages (text or voice) from Telegram users via a webhook.
- Detect message type and, if voice, download and transcribe it to text.
- Use a conversation memory buffer to maintain context over recent messages.
- Send the text input to a GPT-4.1-mini AI model to generate a polite and relevant response.
- Return the AI-generated response back to the user on Telegram.

**Logical blocks:**

- **1.1 Input Reception & Message Type Detection**  
  Receive incoming Telegram messages and determine if they are text or voice.

- **1.2 Voice Processing & Transcription**  
  Download voice messages and transcribe them into text for AI processing.

- **1.3 Text Preparation & Session Management**  
  Standardize inputs (text or transcribed voice) and assign session IDs based on Telegram chat IDs.

- **1.4 AI Processing with Memory**  
  Use a conversation memory buffer and GPT-4.1-mini chat model to generate AI responses.

- **1.5 Output Delivery**  
  Send the AI-generated response back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Message Type Detection

**Overview:**  
This block activates on incoming Telegram messages, filtering updates to only messages, then routes based on message type (text or voice).

**Nodes Involved:**  
- Telegram Trigger  
- Switch  

**Node Details:**  

- **Telegram Trigger**  
  - Type: `telegramTrigger`  
  - Role: Listens to Telegram webhook updates for new messages.  
  - Config: Listens for `message` updates only, uses Telegram API credentials.  
  - Inputs: External webhook call from Telegram.  
  - Outputs: Emits incoming message JSON.  
  - Failures: Possible webhook registration issues, credential errors, or malformed updates.

- **Switch**  
  - Type: `switch`  
  - Role: Routes workflow paths based on whether a message contains text or a voice file.  
  - Config:  
    - Rule "text": checks if `message.text` exists.  
    - Rule "voice": checks if `message.voice.file_id` exists.  
  - Inputs: Telegram Trigger output.  
  - Outputs: Two branches - one for text messages, one for voice messages.  
  - Failures: Expression evaluation failures if message structure unexpected.

---

#### 1.2 Voice Processing & Transcription

**Overview:**  
Handles voice messages by downloading the voice file from Telegram, then transcribing audio to text using OpenAI’s transcription API.

**Nodes Involved:**  
- Get a file  
- Transcribe a recording  
- Edit Fields1  

**Node Details:**  

- **Get a file**  
  - Type: `telegram` node (resource: file)  
  - Role: Downloads voice message file from Telegram servers using `voice.file_id`.  
  - Config: Uses Telegram API credentials, fileId sourced dynamically from incoming message.  
  - Inputs: Switch node (voice branch).  
  - Outputs: Returns file data for transcription.  
  - Failures: File download errors, invalid file IDs, network issues.

- **Transcribe a recording**  
  - Type: `openAi` (audio transcription operation)  
  - Role: Converts the downloaded voice file into text.  
  - Config: Uses OpenAI API credentials, operation set to audio transcription.  
  - Inputs: Output file from Get a file node.  
  - Outputs: Transcribed text under key `text`.  
  - Failures: API quota limits, transcription errors, unsupported audio formats.

- **Edit Fields1**  
  - Type: `set`  
  - Role: Prepares standardized message object with transcribed text and session ID for downstream AI processing.  
  - Config:  
    - Sets `message.text` to transcribed text.  
    - Sets `sessionId` to Telegram chat ID as string.  
  - Inputs: Transcribe a recording output.  
  - Outputs: Standardized JSON for AI Agent input.  
  - Failures: Expression errors if transcription output missing.

---

#### 1.3 Text Preparation & Session Management

**Overview:**  
For direct text messages, extracts the text and session ID for AI processing.

**Nodes Involved:**  
- Edit Fields  

**Node Details:**  

- **Edit Fields**  
  - Type: `set`  
  - Role: Extracts and sets `message.text` and session ID from Telegram text messages.  
  - Config:  
    - Sets `message.text` = incoming message text.  
    - Sets `sessionId` = chat ID as number.  
  - Inputs: Switch node (text branch).  
  - Outputs: Standardized JSON for AI Agent input.  
  - Failures: Expression errors if message text missing.

---

#### 1.4 AI Processing with Memory

**Overview:**  
Combines conversation memory context with the current input to generate a response using GPT-4.1-mini.

**Nodes Involved:**  
- Simple Memory  
- OpenAI Chat Model  
- AI Agent  

**Node Details:**  

- **Simple Memory**  
  - Type: `memoryBufferWindow` (LangChain)  
  - Role: Maintains a sliding window of the last 7 messages per session to preserve conversation context.  
  - Config:  
    - Uses Telegram chat ID as session key.  
    - Context window length set to 7 messages.  
  - Inputs: AI Agent memory input.  
  - Outputs: Provides conversation history to AI Agent.  
  - Failures: Memory overflow, session key errors.

- **OpenAI Chat Model**  
  - Type: `lmChatOpenAi` (LangChain)  
  - Role: Defines the language model used by AI Agent (GPT-4.1-mini).  
  - Config:  
    - Model: gpt-4.1-mini.  
    - Uses OpenAI API credentials.  
  - Inputs: AI Agent language model input.  
  - Outputs: AI-generated text responses.  
  - Failures: Auth errors, rate limits, model unavailability.

- **AI Agent**  
  - Type: `agent` (LangChain)  
  - Role: Core AI processing node that sends standardized text input and conversation memory to OpenAI and receives responses.  
  - Config:  
    - Input: Text from Edit Fields or Edit Fields1 node.  
    - System message: Personalized assistant instructions including dynamic current date/time injection.  
    - Output: Polite, task-focused response text.  
  - Inputs: Text input JSON, memory context, language model.  
  - Outputs: Response JSON with AI-generated output text.  
  - Failures: Expression parsing errors, external API failures, malformed inputs.

---

#### 1.5 Output Delivery

**Overview:**  
Sends the AI-generated response text back to the Telegram user.

**Nodes Involved:**  
- Send a text message  

**Node Details:**  

- **Send a text message**  
  - Type: `telegram` (send message)  
  - Role: Sends the AI-generated output back to the Telegram chat.  
  - Config:  
    - Text: Uses AI Agent output `output`.  
    - Chat ID: Extracted dynamically from Telegram Trigger original message.  
    - Additional: Attribution disabled.  
    - Credentials: Telegram API keys.  
  - Inputs: AI Agent output.  
  - Outputs: Confirmation of message sent.  
  - Failures: API errors, rate limits, invalid chat IDs.

---

### 3. Summary Table

| Node Name           | Node Type                            | Functional Role                                    | Input Node(s)      | Output Node(s)           | Sticky Note                                                                                                          |
|---------------------|------------------------------------|---------------------------------------------------|--------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger     | telegramTrigger                    | Receives incoming Telegram messages               | -                  | Switch                   | 💡 It activates this workflow and triggers/initiates the Telegram message!                                            |
| Switch              | switch                             | Routes message based on type: text or voice       | Telegram Trigger   | Edit Fields, Get a file   | 💡 It split the input from the user. If the user input is text. Then it goes directly to the test field. Else it will be converted into Text Format. |
| Edit Fields         | set                                | Extracts text and session ID from text messages   | Switch (text)       | AI Agent                 | 💡 These nodes get the input text message from Telegram. Then it throws to the AI Agent to give the response back with the help of the OpenAI Chat Model! |
| Get a file          | telegram (file resource)           | Downloads voice message file from Telegram         | Switch (voice)      | Transcribe a recording   | 💡 This node helps your agent to fetch the voice note file                                                           |
| Transcribe a recording | openAi (audio transcription)     | Converts voice audio to text                        | Get a file          | Edit Fields1             | 💡 Transcriber helps to convert the voice into words/text format to analyze the word from the user.                   |
| Edit Fields1        | set                                | Prepares standardized text and session ID after transcription | Transcribe a recording | AI Agent             |                                                                                                                      |
| Simple Memory       | memoryBufferWindow (LangChain)     | Maintains conversation context window              | AI Agent            | AI Agent                 | 💡 This is the **GPT 4.1 mini** chat model. Here we need to connect with our **API key** to utilize this AI model. <br> 💡 And this is a simple memory to help your agent remember the last few messages to stay on topic. |
| OpenAI Chat Model   | lmChatOpenAi (LangChain)            | Provides GPT-4.1-mini language model for AI Agent | AI Agent            | AI Agent                 |                                                                                                                      |
| AI Agent            | agent (LangChain)                   | Processes input text + memory, calls OpenAI model | Edit Fields, Edit Fields1, Simple Memory, OpenAI Chat Model | Send a text message |                                                                                                                      |
| Send a text message | telegram                           | Sends AI-generated text response to user on Telegram | AI Agent          | -                        | 💡 This will send back the response to the user in Telegram. <br> 💡 Here we need to connect with our **Telegram API key** to make a connection. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: `Telegram Trigger`  
   - Parameters: listen for "message" updates.  
   - Credentials: Link your Telegram Bot API credentials.  
   - Position: start of workflow.  
   - Purpose: receive Telegram messages.

2. **Add Switch node:**  
   - Type: `Switch`  
   - Parameters:  
     - Rule 1 "text": check if `message.text` exists (boolean exists).  
     - Rule 2 "voice": check if `message.voice.file_id` exists.  
   - Connect Telegram Trigger main output to Switch input.  
   - Purpose: branch workflow by input type.

3. **For text messages branch:**  
   - Add `Set` node (name: Edit Fields):  
     - Assign `message.text` = `{{$json.message.text}}` (string)  
     - Assign `sessionId` = `{{$json.message.chat.id}}` (number)  
   - Connect Switch "text" output to Edit Fields input.

4. **For voice messages branch:**  
   - Add `Telegram` node (name: Get a file):  
     - Resource: "file"  
     - Operation: "get" (download file)  
     - File ID: `{{$json.message.voice.file_id}}`  
     - Credentials: Telegram API credentials  
   - Connect Switch "voice" output to Get a file input.

   - Add `OpenAI` node (name: Transcribe a recording):  
     - Resource: "audio"  
     - Operation: "transcribe"  
     - Credentials: OpenAI API credentials (with transcription enabled)  
   - Connect Get a file output to Transcribe a recording input.

   - Add `Set` node (name: Edit Fields1):  
     - Assign `message.text` = `{{$json.text}}` (transcribed text)  
     - Assign `sessionId` = `{{$json.message.chat.id}}` (string)  
   - Connect Transcribe a recording output to Edit Fields1 input.

5. **Combine both branches to AI Agent:**  
   - Add `Simple Memory` node (LangChain memoryBufferWindow):  
     - Session Key: `{{$json.message.chat.id}}`  
     - Session ID Type: customKey  
     - Context Window Length: 7  
   - Connect the output of Edit Fields and Edit Fields1 nodes as input to AI Agent (for "main" input).  
   - Connect Simple Memory to AI Agent's `ai_memory` input.

6. **Add OpenAI Chat Model node:**  
   - Model: `gpt-4.1-mini`  
   - Credentials: OpenAI API key  
   - Connect to AI Agent node's `ai_languageModel` input.

7. **Add AI Agent node:**  
   - Input text: bind to `message.text` from Edit Fields or Edit Fields1.  
   - System message:  
     ```
     You are a personal assistant that helps user fulfill their requests.

     When you are asked to perform a task on the current date, please use the current time and date :  {{ $now }}

     ## Output

     You should output the result and don't include any thing unnecessary. Just  make sure that you politely answer user question and ask if any other information is required.
     ```  
   - Connect Edit Fields and Edit Fields1 output to AI Agent main input.  
   - Connect Simple Memory and OpenAI Chat Model as AI Agent memory and languageModel inputs.

8. **Add Telegram node (Send a text message):**  
   - Text: `{{$json.output}}` (AI Agent output)  
   - Chat ID: `{{$node["Telegram Trigger"].json.message.chat.id}}`  
   - Credentials: Telegram API key  
   - Connect AI Agent main output to this node input.

9. **Verify all credential connections:**  
   - Telegram API credentials for Telegram Trigger, Get a file, and Send a text message nodes.  
   - OpenAI API credentials for Transcribe a recording, OpenAI Chat Model, and AI Agent nodes.

10. **Activate workflow and test:**  
    - Send text and voice messages to the Telegram bot.  
    - Confirm AI responses are sent back correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| 💡 This workflow uses the **GPT 4.1 mini** chat model for AI responses and requires an OpenAI API key with chat and transcription permissions.           | OpenAI API documentation                  |
| 💡 Conversation memory is limited to last 7 messages per session to maintain context without excessive token usage.                                        | LangChain memoryBufferWindow documentation|
| 💡 Telegram API credentials must allow reading messages and sending messages for the Telegram bot.                                                        | Telegram Bot API documentation            |
| 💡 System message in AI Agent dynamically injects current date/time to improve temporal awareness in responses.                                          | Dynamic expression usage in n8n            |
| 💡 Voice messages are transcribed before processing, enabling seamless multi-modal input handling.                                                       | OpenAI Whisper transcription API          |
| 💡 Sticky Notes in the workflow provide visual guidance and explanations on key nodes and workflow flow.                                                 | Internal workflow annotations              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---