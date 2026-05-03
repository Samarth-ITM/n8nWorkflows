Voice & Text Assistant with Telegram, Gemini AI, Calendar, Gmail & Notion

https://n8nworkflows.xyz/workflows/voice---text-assistant-with-telegram--gemini-ai--calendar--gmail---notion-8648


# Voice & Text Assistant with Telegram, Gemini AI, Calendar, Gmail & Notion

---

### 1. Workflow Overview

This workflow implements a **Personal Telegram AI Assistant Bot** that leverages Google Gemini AI and integrates with multiple productivity services including Gmail, Google Calendar, and Notion. It listens to incoming Telegram messages (both text and voice), transcribes audio messages if present, and processes user requests through an AI agent that can perform actions such as sending emails, managing calendar events, and creating notes in Notion. The workflow is designed for personal productivity enhancement via conversational AI on Telegram.

**Logical Blocks:**

- **1.1 Telegram Input & Authentication:** Trigger on Telegram messages, verify authorized users, and detect message type (audio or text).
- **1.2 Audio Handling & Transcription:** If voice message detected, download audio and transcribe it to text.
- **1.3 AI Processing & Memory:** Use Google Gemini AI with conversational memory to process input text and decide actions.
- **1.4 Integration with External Services:** Perform actions like sending emails via Gmail, managing Google Calendar events, and creating Notion notes as requested by the AI.
- **1.5 Output Response:** Send AI-generated responses back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input & Authentication

- **Overview:**  
  This block initiates the workflow upon receiving a Telegram message, verifies the user’s authorization, and determines if the message contains text or an audio voice note.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Account Check (Switch)  
  - Check If Audio file (IF Node)  
  - Sticky Note (descriptive)

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Listens to incoming Telegram messages (update type: message).  
    - *Config:* Uses Telegram bot API token credential.  
    - *Input/Output:* Starts workflow, outputs message JSON.  
    - *Failures:* Invalid token/auth failure; missing permissions for bot.  
    - *Notes:* Requires Telegram bot token from BotFather.  
    - *Sticky Note Content:* Explains start of workflow and user verification.

  - **Account Check**  
    - *Type:* Switch  
    - *Role:* Authenticates chat by matching incoming chat ID with authorized ID(s).  
    - *Config:* Compares `message.chat.id` to a predefined number (user’s Telegram chat ID).  
    - *Input/Output:* Input from Telegram Trigger; output routes only authorized messages forward.  
    - *Failures:* Misconfiguration or missing chat ID will block all messages.  
    - *Notes:* Ensures only authorized users interact with the bot.

  - **Check If Audio file**  
    - *Type:* IF  
    - *Role:* Checks if incoming message contains a voice note (audio file).  
    - *Config:* Condition checks for presence of `message.voice.file_id`.  
    - *Input/Output:* Branches workflow to either audio processing or direct AI processing.  
    - *Failures:* Null or unexpected message structure may cause condition failure.

  - **Sticky Note (Telegram Trigger & Checks)**  
    - *Content:* Highlights that this block starts the workflow, verifies users, and checks message type.

#### 2.2 Audio Handling & Transcription

- **Overview:**  
  Handles voice note messages by downloading the audio from Telegram, then transcribing it into text using Google Gemini's speech-to-text capabilities.

- **Nodes Involved:**  
  - Get a file (Telegram)  
  - Transcribe a recording (Google Gemini)  
  - Sticky Note (Audio Handling)

- **Node Details:**

  - **Get a file**  
    - *Type:* Telegram node  
    - *Role:* Downloads the voice file from Telegram servers using `voice.file_id`.  
    - *Config:* File ID dynamically pulled from the incoming message JSON.  
    - *Input/Output:* Input from "Check If Audio file"; outputs binary audio file.  
    - *Failures:* Invalid file ID, Telegram API errors, network problems.

  - **Transcribe a recording**  
    - *Type:* Google Gemini Audio-to-Text node  
    - *Role:* Transcribes audio binary input into text output.  
    - *Config:* Model set to `models/gemini-2.5-flash` for transcription.  
    - *Input/Output:* Receives binary audio from "Get a file"; outputs transcribed text.  
    - *Failures:* API key invalid, transcription timeouts, audio format issues.  
    - *Notes:* Requires Google Gemini API key credential.

  - **Sticky Note (Audio Handling)**  
    - *Content:* Explains retrieval of audio file and transcription process.

#### 2.3 AI Processing & Memory

- **Overview:**  
  Processes the input text (direct or transcribed) via the Google Gemini Chat AI model. Maintains conversational context using a simple memory buffer to allow contextual conversations.

- **Nodes Involved:**  
  - Simple Memory (Langchain Memory Buffer)  
  - Google Gemini Chat Model  
  - AI Agent (Langchain Agent)  
  - Sticky Notes (AI Setup)

- **Node Details:**

  - **Simple Memory**  
    - *Type:* Langchain Memory Buffer Window  
    - *Role:* Keeps track of conversation context using a session key based on input text.  
    - *Config:* Uses a custom session key derived from the combined input text.  
    - *Input/Output:* Input from incoming text; output memory context to AI Agent.  
    - *Failures:* Expression errors if text input is missing.

  - **Google Gemini Chat Model**  
    - *Type:* Langchain Google Gemini Chat Model node  
    - *Role:* Provides the conversational AI language model.  
    - *Config:* Uses Google Gemini API key credential; no additional options configured.  
    - *Input/Output:* Receives input text and memory context; outputs AI-generated text.  
    - *Failures:* API key issues, rate limiting, malformed input.

  - **AI Agent**  
    - *Type:* Langchain Agent  
    - *Role:* Core logic that integrates AI model, memory, and AI tools (e.g., Gmail, Calendar, Notion).  
    - *Config:*  
      - Text input is dynamically taken from either the Telegram message text or transcribed audio text.  
      - A system message defines the assistant’s role and capabilities (send emails, calendar events, create notes).  
      - Prompt type is “define.”  
    - *Input/Output:* Inputs are text, memory, and AI language model; outputs AI response and commands.  
    - *Failures:* Expression errors on input text, API errors.

  - **Sticky Notes (AI Setup)**  
    - *Content:* Explains AI model definition and memory context.

#### 2.4 Integration with External Services

- **Overview:**  
  Enables the AI agent to interact with Gmail, Google Calendar, and Notion to perform requested actions such as sending emails, managing events, and creating notes.

- **Nodes Involved:**  
  - Send a message in Gmail  
  - Create an event in Google Calendar  
  - Read event in Google Calendar  
  - Create notes in Notion

- **Node Details:**

  - **Send a message in Gmail**  
    - *Type:* Gmail Tool node  
    - *Role:* Sends an email using Gmail API based on AI agent instructions.  
    - *Config:*  
      - Recipient, subject, and message body are dynamically overridden by AI agent outputs.  
      - Attribution is disabled.  
    - *Input/Output:* Invoked as an AI tool by AI Agent node; no direct input from Telegram.  
    - *Failures:* OAuth2 token expiration, Gmail API permission issues.  
    - *Notes:* Requires Gmail OAuth2 credentials properly set up.

  - **Create an event in Google Calendar**  
    - *Type:* Google Calendar Tool node  
    - *Role:* Creates calendar events as per AI agent instructions.  
    - *Config:*  
      - Start and end times set dynamically from AI outputs.  
      - Calendar ID is fixed to specific user’s calendar.  
    - *Input/Output:* Invoked by AI agent; outputs event creation confirmation.  
    - *Failures:* OAuth2 issues, invalid date/time formats.

  - **Read event in Google Calendar**  
    - *Type:* Google Calendar Tool node  
    - *Role:* Retrieves calendar events for a specified time range.  
    - *Config:* TimeMin and TimeMax parameters dynamically set by AI outputs.  
    - *Input/Output:* Invoked by AI agent; returns events to AI.  
    - *Failures:* Permission issues, invalid date ranges.

  - **Create notes in Notion**  
    - *Type:* Notion Tool node  
    - *Role:* Creates notes in a specified Notion page as directed by AI.  
    - *Config:*  
      - Title and page URL dynamically set from AI outputs.  
      - Supports bulleted list items and images as blocks.  
    - *Input/Output:* Invoked by AI agent; outputs confirmation.  
    - *Failures:* Invalid Notion API key, page access restrictions.

#### 2.5 Output Response

- **Overview:**  
  Sends the AI-generated textual response back to the Telegram user.

- **Nodes Involved:**  
  - Send a text message (Telegram)

- **Node Details:**

  - **Send a text message**  
    - *Type:* Telegram node  
    - *Role:* Sends chat messages from the bot to Telegram user.  
    - *Config:*  
      - Text is taken from AI Agent’s output.  
      - Chat ID is dynamically extracted from the incoming Telegram message.  
      - Attribution disabled.  
    - *Input/Output:* Receives AI output; sends message to user.  
    - *Failures:* Network errors, invalid chat ID, Telegram API errors.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                        | Input Node(s)          | Output Node(s)              | Sticky Note                                           |
|-------------------------|--------------------------------------|-------------------------------------|-----------------------|-----------------------------|------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger                     | Receives Telegram messages           | —                     | Account Check               | Enter your Telegram bot token from BotFather after creating the bot. |
| Account Check           | Switch                              | Authenticates Telegram user          | Telegram Trigger      | Check If Audio file          | Provide the message ID from your Telegram account in the parameter value field to authenticate your chat. |
| Check If Audio file     | IF Node                            | Determines if message is audio or text | Account Check         | Get a file, AI Agent        | Determines if the incoming message is an audio file or a text message, then routes it accordingly. |
| Get a file              | Telegram node                      | Downloads voice note audio file       | Check If Audio file   | Transcribe a recording       |                                                      |
| Transcribe a recording  | Google Gemini Audio-to-Text        | Transcribes audio to text             | Get a file            | AI Agent                   | This node transcribes audio to text via Google Gemini; provide the Gemini credentials to establish the connection. |
| Simple Memory           | Langchain Memory Buffer Window     | Maintains conversation memory        | (Implicit, from input) | AI Agent                   |                                                      |
| Google Gemini Chat Model| Langchain Language Model           | AI conversational model               |                      | AI Agent                   | Enter the Google Gemini API key to enable Gemini model. |
| AI Agent                | Langchain Agent                    | Core AI logic and orchestration       | Simple Memory, Gemini Model, Transcription, Text input | Send a text message, Gmail, Calendar, Notion nodes |                                                      |
| Send a text message     | Telegram node                      | Sends AI response to user             | AI Agent              | —                           | Sends the bot’s response after the workflow finishes executing. |
| Send a message in Gmail | Gmail Tool                        | Sends emails as instructed by AI     | AI Agent (ai_tool)    | —                           | Connect a Gmail account by opening console.cloud.google.com, enabling the Gmail API for the project, and completing Google account authorization. |
| Create an event in Google Calendar | Google Calendar Tool          | Creates calendar events               | AI Agent (ai_tool)    | —                           | Connect Google Calendar to this node. This node enables the Create Event action in Google Calendar. |
| Read event in Google Calendar  | Google Calendar Tool          | Reads calendar events                 | AI Agent (ai_tool)    | —                           | Connect Google Calendar to this node. This node enables the Read Event action in Google Calendar. |
| Create notes in Notion  | Notion Tool                      | Creates notes in Notion workspace     | AI Agent (ai_tool)    | —                           | Create a Notion integration and paste its API key into the Notion credentials, then provide the public page URL where the bot should post messages. |
| Sticky Note             | Sticky Note                      | Documentation block                   | —                     | —                           | ## Telegram Trigger & Checks - Starts the workflow when a message is received on Telegram. - Verifies the account to ensure only authorized users can access the bot. - Checks if the incoming message contains an audio file or just text. |
| Sticky Note1            | Sticky Note                      | Documentation block                   | —                     | —                           | ## Audio Handling - Retrieves the audio file if the input is a voice note. - Transcribes the recording into text so it can be processed by the AI. |
| Sticky Note2            | Sticky Note                      | Documentation block                   | —                     | —                           | ## AI Setup - Defines the AI model (Google Gemini Chat Model). - Adds Simple Memory to maintain conversation context. |
| Sticky Note3            | Sticky Note                      | Documentation block                   | —                     | —                           | ## AI Setup - Defines the AI model (Google Gemini Chat Model). - Adds Simple Memory to maintain conversation context. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Set update type to `message`.  
   - Add Telegram API credentials by creating a bot via BotFather and entering the token.  
   
2. **Add Switch Node for Account Check**  
   - Type: Switch  
   - Condition: Check that `message.chat.id` equals your authorized Telegram chat ID (numeric).  
   - Connect output to next node.

3. **Add IF Node to Check for Audio File**  
   - Type: IF  
   - Condition: Check if `message.voice.file_id` exists.  
   - If true, connect to "Get a file" node.  
   - If false, connect directly to AI processing.

4. **Add Telegram “Get a file” Node**  
   - Type: Telegram node  
   - Configure to download file using `message.voice.file_id`.  
   - Connect output to transcription.

5. **Add Google Gemini Transcription Node**  
   - Type: Google Gemini Audio-to-Text node  
   - Configure model as `models/gemini-2.5-flash`.  
   - Provide Google Palm API credentials (Google Gemini).  
   - Input: Binary audio from previous node.  
   - Output: Transcribed text.  
   - Connect output to AI agent.

6. **Add Simple Memory Node**  
   - Type: Langchain Memory Buffer Window  
   - Configure `sessionKey` dynamically using input text or transcribed text.  
   - Connect memory output to AI Agent node.

7. **Add Google Gemini Chat Model Node**  
   - Type: Langchain LM Chat Google Gemini  
   - Provide Google Palm API credentials.  
   - Connect to AI Agent as language model input.

8. **Add AI Agent Node**  
   - Type: Langchain Agent  
   - Text input: Use expression to select either Telegram message text or transcribed text.  
   - System message: Define assistant role and capabilities (email, calendar, notes).  
   - Connect inputs from memory and Gemini Chat Model.  
   - Configure AI tools: Gmail, Google Calendar (create/read), Notion nodes.  
   - Connect main output to Send a text message node.

9. **Add Send a text message Node**  
   - Type: Telegram node  
   - Configure to send messages back using chat ID from Telegram Trigger.  
   - Text content from AI Agent output.

10. **Add Gmail Tool Node**  
    - Type: Gmail Tool  
    - Connect as AI tool input for AI Agent.  
    - Configure Gmail OAuth2 credentials with proper API access.  
    - Map recipient, subject, and message dynamically from AI.

11. **Add Google Calendar Tool Nodes**  
    - Create Event Node: Configure calendar ID, dynamic start/end times from AI response.  
    - Read Event Node: Configure calendar ID, dynamic time range from AI response.  
    - Connect both to AI Agent as AI tools.  
    - Add Google Calendar OAuth2 credentials.

12. **Add Notion Tool Node**  
    - Type: Notion Tool  
    - Configure with Notion API key and public page URL.  
    - Map title and block content dynamically from AI outputs.  
    - Connect as AI tool input to AI Agent.

13. **Add Sticky Notes at appropriate places** (optional)  
    - Add descriptive sticky notes explaining blocks and node purposes.

14. **Configure Workflow Settings**  
    - Set timezone to Asia/Kolkata or as required.  
    - Configure error workflow if desired.  
    - Set execution order to ensure correct sequencing.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                                |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| To create your Telegram bot token, use BotFather on Telegram: https://core.telegram.org/bots#6-botfather         | Telegram Bot Setup                                                                                                             |
| For Gmail node, enable Gmail API at https://console.cloud.google.com/apis/library/gmail.googleapis.com and authorize OAuth2 | Gmail API Setup                                                                                                                |
| Google Gemini (PaLM) API key is required for language model and transcription functionalities                  | Google AI: https://developers.generativeai.google/api/guides/authentication                                                   |
| Notion integration requires creating an integration token and sharing the target page with it                   | Notion Developers: https://developers.notion.com/docs/getting-started                                                          |
| The workflow uses Langchain nodes for AI orchestration and memory management                                    | Langchain Docs: https://js.langchain.com/docs/                                                                                 |
| Ensure Telegram bot privacy mode is disabled to receive all messages                                           | Telegram Bot Settings                                                                                                          |

---

**Disclaimer:** The provided text is generated from an automated n8n workflow. It complies with all applicable content policies and contains no illegal or protected data. All processed data is legal and public.

---