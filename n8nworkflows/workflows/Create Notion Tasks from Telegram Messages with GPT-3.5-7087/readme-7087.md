Create Notion Tasks from Telegram Messages with GPT-3.5

https://n8nworkflows.xyz/workflows/create-notion-tasks-from-telegram-messages-with-gpt-3-5-7087


# Create Notion Tasks from Telegram Messages with GPT-3.5

### 1. Workflow Overview

This workflow automates the creation of Notion tasks based on messages received via Telegram, leveraging GPT-3.5 to interpret the message content intelligently. It supports both text and voice messages, transcribing voice recordings to text before processing. The workflow is designed for users who want to add tasks to a Notion database simply by sending messages to a Telegram bot.

Logical blocks:

- **1.1 Input Reception:** Receives incoming Telegram messages (text or voice).
- **1.2 Message Type Branching:** Distinguishes between text and voice messages to process accordingly.
- **1.3 Voice Message Processing:** Downloads and transcribes voice messages to text.
- **1.4 Text Handling:** Extracts text from messages or transcriptions.
- **1.5 AI Interpretation:** Uses GPT-3.5 to parse the message, extract task details, and generate confirmation text.
- **1.6 Notion Task Creation:** Creates a new task in the Notion database with extracted details.
- **1.7 Confirmation Message:** Sends a confirmation back to the Telegram user.
- **1.8 Documentation Notes:** Sticky notes explaining configuration tips and workflow purpose.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures new Telegram messages directed to the bot, filtering to accept messages only from a specific user and downloading media when applicable.

**Nodes Involved:**  
- Telegram Trigger

**Node Details:**  
- **Telegram Trigger**  
  - Type: Telegram Trigger (Webhook-based)  
  - Config: Listens for "message" updates, filters by user ID "8353510776", downloads media files automatically.  
  - Inputs: None (entry point)  
  - Outputs: Message JSON objects including text or voice data  
  - Notes: Requires Telegram API credentials linked to a Telegram bot.  
  - Edge cases: Unauthorized users sending messages are ignored; media download failures possible if file ID is invalid or network issues occur.

#### 1.2 Message Type Branching

**Overview:**  
Determines if the incoming message contains voice or text content to route the workflow appropriately.

**Nodes Involved:**  
- Switch text vs audio

**Node Details:**  
- **Switch text vs audio**  
  - Type: Switch node  
  - Config: Two branches:  
    - Voice: Checks if `$json.message.voice` exists.  
    - Text: Checks if `$json.message.text` exists.  
  - Inputs: Telegram Trigger  
  - Outputs:  
    - Branch 0 (voice message path)  
    - Branch 1 (text message path)  
  - Edge cases: Messages lacking both voice and text properties will not proceed further; this case is not explicitly handled.

#### 1.3 Voice Message Processing

**Overview:**  
Downloads voice message recordings from Telegram and transcribes audio into text using OpenAI’s transcription.

**Nodes Involved:**  
- Download recording  
- Transcribe recording

**Node Details:**  
- **Download recording**  
  - Type: Telegram (Download file)  
  - Config: Downloads file using `file_id` from `$json.message.voice.file_id`.  
  - Inputs: Switch node voice branch  
  - Outputs: Binary audio file data  
  - Edge cases: Invalid or expired file IDs, Telegram API errors, network failures.

- **Transcribe recording**  
  - Type: Langchain OpenAI node (audio transcription)  
  - Config: Operation set to "transcribe" on the audio resource, using OpenAI credentials.  
  - Inputs: Download recording  
  - Outputs: JSON containing transcribed text property  
  - Edge cases: Audio format unsupported, transcription failures, OpenAI API quota or auth errors.

#### 1.4 Text Handling

**Overview:**  
Extracts the text content either directly from the text message or from the transcription result for further processing.

**Nodes Involved:**  
- Handle Text Message

**Node Details:**  
- **Handle Text Message**  
  - Type: Code node (JavaScript)  
  - Config: Returns JSON with `text` property extracted from the first input’s `$json.message.text`.  
  - Inputs: Switch node text branch  
  - Outputs: JSON with `{ text: <message text> }`  
  - Edge cases: Missing or empty text properties could cause empty outputs.

#### 1.5 AI Interpretation

**Overview:**  
Processes the extracted text with GPT-3.5 to interpret task details such as title and due date, and generates a confirmation message for the user.

**Nodes Involved:**  
- Interpret Request

**Node Details:**  
- **Interpret Request**  
  - Type: Langchain OpenAI node (chat completion)  
  - Config: Uses GPT-3.5-turbo model with a system prompt explaining the task (adding tasks to Notion including optional due date).  
  - Input message: The extracted text from the previous step.  
  - Outputs: JSON with AI-generated task fields and confirmation message.  
  - Edge cases: API failures, rate limits, unexpected AI outputs, possible empty or irrelevant responses.  
  - Notes: The system prompt is adjustable via a sticky note to customize AI behavior.

#### 1.6 Notion Task Creation

**Overview:**  
Creates a new task in the configured Notion database using the interpreted title and optional due date extracted by the AI.

**Nodes Involved:**  
- Create new Task in Notion

**Node Details:**  
- **Create new Task in Notion**  
  - Type: Notion Tool node  
  - Config: Creates a database page in the Notion database with ID `"22c1b7ff-072a-807a-b0c7-cb5733cb7997"` (Tasks).  
  - Title and Do Date properties use AI-generated overrides from the Interpret Request node outputs.  
  - Inputs: AI tool input from Interpret Request node (indicated by `ai_tool` connection).  
  - Outputs: JSON with created page details.  
  - Edge cases: Notion API auth errors, invalid database ID, property mapping errors, timezone mismatches (configured timezone is America/New_York).  
  - Notes: Timezone should be customized to user preference.

#### 1.7 Confirmation Message

**Overview:**  
Sends a confirmation message back to the Telegram user confirming task creation with a friendly and witty note generated by GPT.

**Nodes Involved:**  
- Confirmation message

**Node Details:**  
- **Confirmation message**  
  - Type: Telegram (Send message)  
  - Config: Sends text from AI’s confirmation message (`$json.message.content`).  
  - Fixed chat ID: `8353510776` (the user)  
  - Inputs: Interpret Request output  
  - Outputs: None (terminal node)  
  - Edge cases: Telegram API send failures, invalid chat ID.

#### 1.8 Documentation Notes

**Overview:**  
Sticky notes providing documentation, configuration hints, and optional instructions.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**  
- **Sticky Note**  
  - Content: Explains workflow purpose, how Telegram and Notion are configured, and general usage.  
- **Sticky Note1**  
  - Content: Advises adjusting the timezone to preferred setting.  
- **Sticky Note2**  
  - Content: Notes the system prompt can be optionally adjusted for customization.

---

### 3. Summary Table

| Node Name             | Node Type                           | Functional Role                       | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                 |
|-----------------------|-----------------------------------|------------------------------------|------------------------|---------------------------|---------------------------------------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger                  | Receives Telegram messages          | -                      | Switch text vs audio       | See "Telegram Add Notion Tasks Workflow" sticky note for setup overview                     |
| Switch text vs audio   | Switch                           | Routes messages by type (voice/text)| Telegram Trigger        | Download recording, Handle Text Message |                                                                                             |
| Download recording     | Telegram (Download file)          | Downloads voice message audio       | Switch text vs audio    | Transcribe recording       |                                                                                             |
| Transcribe recording   | Langchain OpenAI (audio)          | Transcribes audio to text           | Download recording      | Interpret Request          |                                                                                             |
| Handle Text Message    | Code                             | Extracts text from message          | Switch text vs audio    | Interpret Request          |                                                                                             |
| Interpret Request      | Langchain OpenAI (chat)           | Interprets message and generates AI output | Handle Text Message, Transcribe recording | Confirmation message (main), Create new Task in Notion (ai_tool) | "Optional: Adjust the system prompt..." sticky note                                         |
| Create new Task in Notion | Notion Tool                    | Creates task in Notion database     | Interpret Request (ai_tool) | -                         | "Make sure to adjust the timezone..." sticky note                                          |
| Confirmation message   | Telegram (Send message)            | Sends confirmation to user          | Interpret Request       | -                         |                                                                                             |
| Sticky Note            | Sticky Note                      | Documentation and setup instructions | -                      | -                         | Explains workflow purpose, Telegram and Notion setup                                      |
| Sticky Note1           | Sticky Note                      | Timezone adjustment reminder        | -                      | -                         | Advises timezone customization                                                             |
| Sticky Note2           | Sticky Note                      | System prompt customization hint   | -                      | -                         | Notes about optional system prompt adjustment                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: `message`  
     - Additional Fields: `userIds` set to Telegram user ID `"8353510776"`  
     - Enable media download  
   - Credentials: Select or create Telegram API credentials for your bot  
   - Position: Start node  

2. **Add Switch node named "Switch text vs audio"**  
   - Type: Switch  
   - Rules:  
     - Branch 1 (Voice): Condition – check if `$json.message.voice` exists (object exists)  
     - Branch 2 (Text): Condition – check if `$json.message.text` exists (string exists)  
   - Connect Telegram Trigger output to this node’s input  

3. **Add Telegram node "Download recording"**  
   - Type: Telegram  
   - Parameters:  
     - Resource: `file`  
     - File ID: Set to expression `{{$json.message.voice.file_id}}`  
   - Credentials: Use same Telegram API credentials  
   - Connect Switch node voice branch output (branch 1) to this node  

4. **Add OpenAI Langchain node "Transcribe recording"**  
   - Type: Langchain OpenAI  
   - Resource: `audio`  
   - Operation: `transcribe`  
   - Credentials: OpenAI API credentials with access to transcription (e.g., Whisper)  
   - Connect Download recording node output to this node  

5. **Add Code node "Handle Text Message"**  
   - Type: Code  
   - Code (JavaScript):  
     ```javascript
     return [{ json: { text: $input.first().json.message.text } }];
     ```  
   - Connect Switch text branch output (branch 2) to this node  

6. **Add OpenAI Langchain node "Interpret Request"**  
   - Type: Langchain OpenAI  
   - Model ID: `gpt-3.5-turbo`  
   - Messages:  
     - System role with prompt:  
       ```
       You are a friendly helpful assistant!

       Today’s date is: {{ (new Date()).toISOString() }}

       The incoming messages you receive will typically be requests to add new Tasks to a Notion Database. I will probably preface these messages with something like "remember to" (or "don't forget") which basically all mean to create a new task. Use the Notion tool to create those new tasks.  Some tasks will simply be a title while others may also include a Do Date.  Be sure to set the Do Date field if provided in the original message.

       Whenever you do successfully create a new task, please respond with a simple confirmation message. Make the message clever/funny and feel free to include a random or appropriate emoji.  Remember you are only confirming that you've added the task/event, the context of your response doesn't necessarily need to have anything to do with the actual item itself.
       ```
     - User message: Use expression `{{$json.text}}` from previous node  
   - Credentials: OpenAI API credentials  
   - Connect outputs of both Transcribe recording and Handle Text Message nodes to this node’s input (two inputs merged into one AI tool input)  

7. **Add Notion Tool node "Create new Task in Notion"**  
   - Type: Notion Tool (Create database page)  
   - Database ID: Set to your Tasks database ID (e.g., `"22c1b7ff-072a-807a-b0c7-cb5733cb7997"`)  
   - Properties:  
     - Title: Use AI output override variable `$fromAI('Title', '', 'string')`  
     - Do Date (Date property): Use AI output override variable `$fromAI('propertyValues0_Date', '', 'string')`  
   - Credentials: Notion API credentials with access to the database  
   - Connect Interpret Request node’s AI tool output to this node  

8. **Add Telegram node "Confirmation message"**  
   - Type: Telegram (Send message)  
   - Chat ID: Fixed to `"8353510776"` (the user)  
   - Text: Expression `{{$json.message.content}}` from Interpret Request node output  
   - Credentials: Telegram API credentials  
   - Connect Interpret Request node main output to this node  

9. **Add Sticky Notes for Documentation (Optional but recommended)**  
   - Create three Sticky Note nodes with the same content as in the original workflow to help users remember configuration details:  
     - Workflow overview and usage instructions  
     - Timezone adjustment reminder  
     - Optional system prompt customization hint  

10. **Finalize and Test**  
    - Verify all credentials are properly configured  
    - Adjust timezone if necessary in Notion node  
    - Test by sending text and voice messages to your Telegram bot and verify tasks are created in Notion and confirmations sent back.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow accepts Telegram messages and creates Notion tasks using AI to interpret intent and task details.                   | Workflow purpose and architecture explanation.                                                    |
| Configure your Telegram bot and Notion integration credentials carefully to enable smooth operation.                              | Credential setup and API access.                                                                  |
| Adjust the OpenAI system prompt in the "Interpret Request" node to customize AI behavior and task interpretation logic.           | Located in the Langchain OpenAI "Interpret Request" node.                                         |
| Make sure the timezone in the Notion task creation matches your local timezone for correct due date handling.                     | Sticky Note1 content reminder.                                                                    |
| Workflow supports voice messages by downloading and transcribing them to text before AI processing.                               | Enables multi-modal input handling (text and voice).                                             |
| Confirmation messages sent back to Telegram users are designed to be friendly and witty, including emojis.                        | Enhances user experience and confirmation clarity.                                               |
| For more information about n8n and node configuration, visit https://docs.n8n.io/                                                 | Official n8n documentation.                                                                       |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.