Manage Calendar with Voice & Text using GPT-4, Telegram & Google Calendar

https://n8nworkflows.xyz/workflows/manage-calendar-with-voice---text-using-gpt-4--telegram---google-calendar-4102


# Manage Calendar with Voice & Text using GPT-4, Telegram & Google Calendar

### 1. Workflow Overview

This workflow, titled **"Manage Calendar with Voice & Text using GPT-4, Telegram & Google Calendar"**, is designed to enable users to manage their Google Calendar events via natural language inputs sent through Telegram. It supports both text messages and voice notes (audio in .ogg format), leveraging AI capabilities (OpenAI Whisper for transcription and GPT-4 for understanding user intent) to automate calendar operations such as creating, updating, fetching, or deleting events.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**  
  Captures incoming Telegram messages (text or voice) and determines the message type.

- **1.2 Voice Processing**  
  If the input is a voice message, downloads and transcribes it to text.

- **1.3 Text Processing**  
  Extracts plain text from text messages or from transcribed voice messages.

- **1.4 AI Calendar Assistant**  
  Uses GPT-4 via LangChain to analyze user intent based on the extracted text and decide the calendar action to perform.

- **1.5 Google Calendar Operations**  
  Executes the appropriate Google Calendar API actions (create, update, fetch, delete events) dynamically based on AI instructions.

- **1.6 Response Sending**  
  Sends confirmation or informational messages back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming Telegram messages and routes them based on whether they contain voice data or text.

- **Nodes Involved:**  
  - Receive User Input (Telegram)  
  - Is Voice Message?

- **Node Details:**

  - **Receive User Input (Telegram)**  
    - Type: Telegram Trigger  
    - Role: Entry point listening to Telegram updates (messages).  
    - Configuration: Triggers workflow on any Telegram message (text or voice). Uses Telegram API credentials.  
    - Inputs: Telegram webhook with message updates.  
    - Outputs: Passes message JSON downstream.  
    - Edge Cases: Telegram API connectivity issues, webhook misconfiguration, unsupported message types.

  - **Is Voice Message?**  
    - Type: If node  
    - Role: Conditional branch checking if the message contains a voice note with MIME type "audio/ogg".  
    - Configuration: Checks `$json.message.voice.mime_type` equals `"audio/ogg"`.  
    - Inputs: Output from Telegram trigger.  
    - Outputs: Two outputs — true branch (voice message flow), false branch (text message flow).  
    - Edge Cases: Missing or malformed voice object, messages with other audio formats, expression evaluation errors.

---

#### 1.2 Voice Processing

- **Overview:**  
  Handles voice messages by downloading the audio file and transcribing it into text using OpenAI Whisper.

- **Nodes Involved:**  
  - Download Voice File  
  - Transcribe Voice to Text

- **Node Details:**

  - **Download Voice File**  
    - Type: Telegram node (file download)  
    - Role: Retrieves the voice message file from Telegram servers.  
    - Configuration: Uses the file ID from the incoming message to fetch the voice file. Requires Telegram API credentials.  
    - Inputs: Voice message file ID from "Is Voice Message?" node.  
    - Outputs: Binary audio file for transcription.  
    - Edge Cases: File not found, Telegram API rate limits, network errors.

  - **Transcribe Voice to Text**  
    - Type: LangChain OpenAI audio node (Whisper)  
    - Role: Converts downloaded audio to plain text transcription.  
    - Configuration: Language set to English (`"en"`), uses OpenAI API credentials.  
    - Inputs: Audio file binary from previous node.  
    - Outputs: JSON containing the transcribed text.  
    - Edge Cases: Audio format incompatibility, transcription inaccuracies, API errors, timeouts.

---

#### 1.3 Text Processing

- **Overview:**  
  Extracts plain text either directly from text messages or from the transcribed voice text for further AI processing.

- **Nodes Involved:**  
  - Extract Text Message

- **Node Details:**

  - **Extract Text Message**  
    - Type: Set node  
    - Role: Assigns the plain text of the message to a field named `Text`.  
    - Configuration: Sets `Text` to `$json.message.text` from the Telegram message.  
    - Inputs: From false branch of "Is Voice Message?" node (text messages).  
    - Outputs: JSON with property `Text` for AI processing.  
    - Edge Cases: Missing text property, empty messages.

---

#### 1.4 AI Calendar Assistant

- **Overview:**  
  Uses GPT-4 via LangChain to analyze the user’s text input and determine the calendar action to take, factoring in priorities, work-life balance, and recurring events.

- **Nodes Involved:**  
  - AI Calendar Assistant (LangChain)  
  - GPT-4 Language Model

- **Node Details:**

  - **AI Calendar Assistant (LangChain)**  
    - Type: LangChain Agent node  
    - Role: Main AI agent interpreting user input, generating calendar instructions.  
    - Configuration:  
      - Prompt includes current datetime (`{{ $now }}`) and user text variables (`{{ $json.text }}`, `{{ $json.Text }}`).  
      - Purpose directive to manage calendar intelligently.  
      - Uses GPT-4 as the language model backend.  
    - Inputs: Text from either transcription or extracted text node.  
    - Outputs: JSON with AI output determining calendar action and parameters.  
    - Edge Cases: AI misinterpretation, prompt errors, API call failures, rate limits.

  - **GPT-4 Language Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Provides GPT-4 language understanding to the LangChain agent.  
    - Configuration: Uses model `gpt-4.1`, OpenAI API credentials required.  
    - Inputs: Linked internally to the AI Calendar Assistant node as language model.  
    - Outputs: Processed AI chat completions.  
    - Edge Cases: API quota exhaustion, network errors.

---

#### 1.5 Google Calendar Operations

- **Overview:**  
  Executes the calendar operations as decided by the AI assistant, dynamically performing create, update, fetch, or delete actions on Google Calendar.

- **Nodes Involved:**  
  - Create Event  
  - Get events  
  - Update Calendar  
  - Detele Event

- **Node Details:**

  - **Create Event**  
    - Type: Google Calendar Tool node  
    - Role: Creates a new event with start/end time, description, and calendar ID specified by AI output.  
    - Configuration: Fields dynamically filled by AI via expressions like `$fromAI('Start')`, `$fromAI('End')`, `$fromAI('Description')`, and calendar email ID. Uses Google Calendar OAuth2 credentials.  
    - Inputs: From AI Calendar Assistant node (AI output).  
    - Outputs: Confirmation of created event.  
    - Edge Cases: Invalid date/time formats, calendar access denied, quota limits.

  - **Get events**  
    - Type: Google Calendar Tool node  
    - Role: Retrieves events within a specified time range from a given calendar.  
    - Configuration: Uses `$fromAI('After')` and `$fromAI('Before')` for time filtering, `$fromAI('Return_All')` to decide if all events are fetched, and calendar ID.  
    - Inputs: From AI Calendar Assistant node.  
    - Outputs: List of events matching criteria.  
    - Edge Cases: Large result sets, API timeouts.

  - **Update Calendar**  
    - Type: Google Calendar Tool node  
    - Role: Updates an existing event identified by event ID with updated fields.  
    - Configuration: Event ID and calendar ID from AI output, including whether to use default reminders.  
    - Inputs: From AI Calendar Assistant node.  
    - Outputs: Confirmation of updated event.  
    - Edge Cases: Event not found, permission errors.

  - **Detele Event**  
    - Type: Google Calendar Tool node  
    - Role: Deletes an event using event ID and calendar ID provided by AI.  
    - Configuration: Event ID and calendar ID from AI output.  
    - Inputs: From AI Calendar Assistant node.  
    - Outputs: Confirmation of deletion.  
    - Edge Cases: Event already deleted, insufficient permissions.

---

#### 1.6 Response Sending

- **Overview:**  
  Sends back a user-friendly message to the Telegram chat reflecting the result of the calendar operation or AI response.

- **Nodes Involved:**  
  - Send Response to Telegram

- **Node Details:**

  - **Send Response to Telegram**  
    - Type: Telegram node (send message)  
    - Role: Replies to the user with the output generated by the AI assistant.  
    - Configuration: Sends text from `$json.output`, uses the chat ID from the original Telegram message to target response, disables attribution. Requires Telegram API credentials.  
    - Inputs: Output from AI Calendar Assistant node.  
    - Outputs: Confirmation of sent message.  
    - Edge Cases: Invalid chat ID, Telegram API errors.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                                  | Input Node(s)                 | Output Node(s)                       | Sticky Note                                                                                      |
|---------------------------|--------------------------------|-------------------------------------------------|------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------|
| Receive User Input (Telegram)   | telegramTrigger               | Entry point: receives Telegram messages          | —                            | Is Voice Message?                   | **Start / Input:** Listen for Telegram input (text or voice message)                            |
| Is Voice Message?          | if                            | Checks if message is a voice note (.ogg)         | Receive User Input            | Download Voice File (true), Extract Text Message (false) | **Decision: Voice or Text:** Check if incoming message is a voice note (.ogg)                   |
| Download Voice File        | telegram                      | Downloads voice message audio file                | Is Voice Message? (true)      | Transcribe Voice to Text            | **Voice Transcription:** Download voice file and transcribe to text                             |
| Transcribe Voice to Text   | @n8n/n8n-nodes-langchain.openAi | Transcribes audio to text using OpenAI Whisper   | Download Voice File           | AI Calendar Assistant              | **Voice Transcription:** Download voice file and transcribe to text                             |
| Extract Text Message       | set                           | Extracts plain text from Telegram text message   | Is Voice Message? (false)     | AI Calendar Assistant              | **Text Extraction:** Extract plain text from user’s message (if no voice)                       |
| AI Calendar Assistant (LangChain) | @n8n/n8n-nodes-langchain.agent | Uses GPT-4 to interpret message and decide action | Transcribe Voice to Text / Extract Text Message / GPT-4 Language Model | Send Response to Telegram, Create Event, Get events, Update Calendar, Detele Event | **AI Interpretation:** Analyze user intent with GPT-4; decide calendar action; **Google Calendar Actions:** Execute chosen calendar operation |
| GPT-4 Language Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4 model for AI assistant             | —                            | AI Calendar Assistant              | **AI Interpretation:** Analyze user intent with GPT-4; decide calendar action; **Google Calendar Actions:** Execute chosen calendar operation |
| Create Event              | googleCalendarTool            | Creates calendar event                            | AI Calendar Assistant         | —                                  | **Google Calendar Actions:** Perform selected calendar action                                 |
| Get events                | googleCalendarTool            | Retrieves calendar events                         | AI Calendar Assistant         | —                                  | **Google Calendar Actions:** Perform selected calendar action                                 |
| Update Calendar           | googleCalendarTool            | Updates existing calendar event                   | AI Calendar Assistant         | —                                  | **Google Calendar Actions:** Perform selected calendar action                                 |
| Detele Event              | googleCalendarTool            | Deletes calendar event                            | AI Calendar Assistant         | —                                  | **Google Calendar Actions:** Perform selected calendar action                                 |
| Send Response to Telegram | telegram                      | Sends AI response back to Telegram user          | AI Calendar Assistant         | —                                  | **Send Response:** Send confirmation or result message back to the user                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `telegramTrigger`  
   - Parameters: Listen to `message` updates.  
   - Credentials: Set up Telegram API credentials with access to your bot.  
   - Position: Start node.  

2. **Add If Node "Is Voice Message?"**  
   - Type: `if` node, v2.2.  
   - Condition: Check if expression `$json.message.voice.mime_type` equals `"audio/ogg"`.  
   - Connect Telegram Trigger output to this node.  

3. **Branch True (Voice Message): Add Telegram Node "Download Voice File"**  
   - Type: `telegram` node.  
   - Parameters: Resource = `file`, File ID = `{{$node["Receive User Input (Telegram)"].json["message"]["voice"]["file_id"]}}`.  
   - Credentials: Use Telegram API credentials.  
   - Connect true output from "Is Voice Message?" to this node.  

4. **Add LangChain Node "Transcribe Voice to Text"**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`, resource = `audio`, operation = `transcribe`.  
   - Set language option to `"en"`.  
   - Credentials: OpenAI API credentials.  
   - Connect output of "Download Voice File" to this node.  

5. **Branch False (Text Message): Add Set Node "Extract Text Message"**  
   - Type: `set` node.  
   - Create field `Text` with value `{{$json.message.text}}`.  
   - Connect false output from "Is Voice Message?" to this node.  

6. **Add LangChain Agent Node "AI Calendar Assistant"**  
   - Type: `@n8n/n8n-nodes-langchain.agent`.  
   - Parameters:  
     - Prompt includes:  
       ```
       You're my personal digital assistant. Your purpose is to help me manage my personal calendar efficiently, taking into account priorities, work-life balance, and recurring events.

       Today is {{$now}}

       Instructions:
       {{$json.text}}
       {{$json.Text}}
       ```  
   - Connect outputs of both "Transcribe Voice to Text" and "Extract Text Message" to this node.  
   - Attach the GPT-4 language model node as AI backend.  

7. **Add LangChain Chat Model Node "GPT-4 Language Model"**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`.  
   - Model: Select `gpt-4.1`.  
   - Credentials: Set OpenAI API credentials.  
   - Connect internally to "AI Calendar Assistant" as language model.  

8. **Add Google Calendar Tool Nodes:**  
   - **Create Event**: Set operation `create`. Parameters: start, end, description, calendar ID from AI output using expressions like `$fromAI('Start')`.  
   - **Get events**: Operation `getAll` with filters `timeMin`, `timeMax`, calendar ID, and `returnAll` from AI output.  
   - **Update Calendar**: Operation `update` with event ID, calendar ID, update fields, and reminders from AI output.  
   - **Detele Event**: Operation `delete` with event ID and calendar ID from AI output.  
   - Credentials: Use Google Calendar OAuth2 credentials.  
   - Connect all Google Calendar nodes as parallel outputs of "AI Calendar Assistant" node under appropriate AI tool paths.  

9. **Add Telegram Node "Send Response to Telegram"**  
   - Type: `telegram` node (send message).  
   - Parameters: Text = `{{$json.output}}`, Chat ID = `{{$node["Receive User Input (Telegram)"].json.message.chat.id}}`.  
   - Credentials: Telegram API credentials.  
   - Connect output of "AI Calendar Assistant" node to this node.  

10. **Set up Credentials:**  
    - Telegram: OAuth/token for your bot.  
    - OpenAI: API key with access to GPT-4 and Whisper.  
    - Google Calendar OAuth2: Authorized account with calendar access.  

11. **Configure Webhooks and Test:**  
    - Ensure Telegram webhook URL points to your n8n instance.  
    - Test sending text and voice messages to the Telegram bot to verify end-to-end flow.  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| AI-Powered Calendar Assistant: Voice & Text Input → GPT-4 → Google Calendar                               | Workflow description clarifies integration of Telegram, OpenAI Whisper and GPT-4, and Google Calendar.  |
| Start: Listen for Telegram input (text or voice message)                                                  | Entry node listens to user input via Telegram bot messages.                                             |
| Decision: Voice or Text - check for voice note (.ogg)                                                     | Route workflow between voice transcription or direct text processing paths.                             |
| Voice Transcription uses OpenAI Whisper for audio-to-text conversion                                      | Requires OpenAI Whisper support and audio resource configuration.                                       |
| AI Interpretation uses GPT-4 to analyze intent and select calendar action                                 | Defines prompt guiding AI to manage calendar intelligently with user context and preferences.           |
| Google Calendar Actions dynamically create, update, fetch, or delete events                               | Requires Google Calendar OAuth2 credentials with appropriate scopes.                                     |
| Send Response sends confirmation or results back to Telegram user                                        | Closes the loop with user feedback on requested action.                                                 |
| Telegram API, OpenAI API, and Google Calendar OAuth2 credentials must be pre-configured and valid         | Credential setup is critical for workflow execution.                                                     |
| Workflow handles common edge cases such as missing voice or text, API errors, and permission issues      | Robust error handling should be considered during deployment and testing.                               |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. The processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.