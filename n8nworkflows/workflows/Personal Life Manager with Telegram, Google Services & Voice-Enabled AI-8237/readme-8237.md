Personal Life Manager with Telegram, Google Services & Voice-Enabled AI

https://n8nworkflows.xyz/workflows/personal-life-manager-with-telegram--google-services---voice-enabled-ai-8237


# Personal Life Manager with Telegram, Google Services & Voice-Enabled AI

### 1. Workflow Overview

This workflow implements a **Personal Life Manager** integrating Telegram, Google services (Calendar, Tasks, Gmail), and Voice-Enabled AI using OpenAI/OpenRouter models. Its primary purpose is to allow users to interact with their digital life through Telegram messages and voice notes, enabling hands-free management of emails, calendar events, and tasks via conversational AI.

Logical blocks of the workflow:

- **1.1 Input Reception and Preprocessing:** Receives Telegram messages or voice notes, distinguishes input type (voice or text), and prepares data for processing.
  
- **1.2 Voice Transcription:** Converts incoming voice messages from Telegram into text using OpenAI's transcription API.
  
- **1.3 Contextual Memory Management:** Maintains a conversational memory buffer keyed per user to allow contextual AI interactions.
  
- **1.4 AI Processing and Task Handling:** The core AI agent “Caylee” processes user queries by accessing Gmail, Google Calendar, Tasks, and responding accordingly.
  
- **1.5 Google Services Integration:** Nodes specifically fetch emails, calendar events, and manage Google Tasks as requested by the user.
  
- **1.6 Response Delivery:** Sends the AI-generated replies back to the user via Telegram.

Supporting elements include sticky notes providing setup instructions, usage tips, and video tutorial links.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preprocessing

**Overview:**  
This block captures incoming Telegram updates (messages or voice notes), distinguishes between voice and text inputs, and filters empty messages.

**Nodes Involved:**  
- Listen for incoming events  
- Voice or Text (Set node)  
- If (Conditional node)  
- Get Voice File

**Node Details:**  

- **Listen for incoming events**  
  - Type: Telegram Trigger  
  - Role: Entry point to capture Telegram messages (text or voice).  
  - Config: Listens to "message" updates; uses Telegram credentials.  
  - Input: External webhook from Telegram.  
  - Output: JSON message payload containing Telegram message data.  
  - Failure modes: Telegram API downtime, webhook misconfiguration, invalid message format.

- **Voice or Text**  
  - Type: Set  
  - Role: Extracts message text or sets empty string if absent.  
  - Config: Sets a variable `text` with `{{$json?.message?.text || ""}}` to handle both text and voice inputs uniformly.  
  - Input: From Telegram trigger node.  
  - Output: Contains `text` field for downstream conditional check.

- **If**  
  - Type: If (Conditional)  
  - Role: Checks if the incoming message text is empty (distinguishes voice messages).  
  - Config: Condition tests if `{{$json.message.text}}` is empty.  
  - Input: From "Voice or Text" node.  
  - Output:  
    - True: No text present → proceed to get voice file.  
    - False: Text present → proceed to AI assistant directly.  
  - Edge cases: Empty messages (no text or voice), malformed messages.

- **Get Voice File**  
  - Type: Telegram node (File resource)  
  - Role: Fetches the voice file from Telegram using the voice message file ID.  
  - Config: Uses `{{$('Listen for incoming events').item.json.message.voice.file_id}}` to get the voice file.  
  - Input: Triggered if message has no text (voice message).  
  - Output: Voice file binary data for transcription.  
  - Failure modes: File ID missing or expired, Telegram file API errors.

---

#### 2.2 Voice Transcription

**Overview:**  
Converts the Telegram voice message audio into text using OpenAI’s transcription capabilities.

**Nodes Involved:**  
- Transcribe a recording

**Node Details:**  

- **Transcribe a recording**  
  - Type: OpenAI node (audio resource)  
  - Role: Transcribes voice audio to text.  
  - Config: Operation set to "transcribe"; requires OpenAI API credentials.  
  - Input: Receives audio binary from "Get Voice File" node.  
  - Output: Transcribed text for AI processing.  
  - Edge cases: Audio quality issues, transcription API errors, rate limits.

---

#### 2.3 Contextual Memory Management

**Overview:**  
Maintains a sliding window memory buffer per user session to keep conversation context for the AI assistant.

**Nodes Involved:**  
- Window Buffer Memory

**Node Details:**  

- **Window Buffer Memory**  
  - Type: LangChain memory buffer (window)  
  - Role: Stores recent conversation messages keyed by Telegram user ID to maintain context.  
  - Config: Session key derived from `{{$('Listen for incoming events').first().json.message.from.id}}`.  
  - Input: Connected to AI assistant node.  
  - Output: Provides memory context for AI prompt.  
  - Edge cases: Memory overflow, session key missing, concurrency issues.

---

#### 2.4 AI Processing and Task Handling

**Overview:**  
Main AI agent “Caylee” interprets user input, accesses Google APIs as needed, and generates responses.

**Nodes Involved:**  
- Caylee, AI Assistant 👩🏻‍🏫  
- OpenRouter  
- Get Email  
- Google Calendar  
- Create a task in Google Tasks  
- Get many tasks in Google Tasks

**Node Details:**  

- **Caylee, AI Assistant 👩🏻‍🏫**  
  - Type: LangChain Agent  
  - Role: Core conversational AI agent processing user queries.  
  - Config: Uses a system message defining assistant behavior (personal assistant called Caylee with specific guidelines on email summary, calendar filtering, and date assumptions).  
  - Inputs: Text from transcription or direct message, AI language model from OpenRouter, contextual memory, Google services nodes input.  
  - Outputs: Text response to send back to Telegram.  
  - Edge cases: Model API errors, prompt formatting issues, external API failures.

- **OpenRouter**  
  - Type: LangChain OpenRouter Language Model  
  - Role: Provides AI language model backend for Caylee.  
  - Config: Uses OpenRouter API key credential.  
  - Input: Receives prompt from Caylee node.  
  - Output: AI-generated completions.  
  - Failure modes: API key invalid, rate limits, network errors.

- **Get Email**  
  - Type: Gmail Tool  
  - Role: Retrieves recent unread emails from user inbox.  
  - Config: Limit 20 emails, filters for INBOX and UNREAD labels.  
  - Input: Triggered by AI agent when email info requested.  
  - Output: Email summaries to AI agent.  
  - Failure modes: Gmail API auth errors, quota limits, network issues.

- **Google Calendar**  
  - Type: Google Calendar Tool  
  - Role: Fetches calendar events after a date specified by AI.  
  - Config: Uses dynamic date from AI input; retrieves summary and start time fields.  
  - Input: AI agent request.  
  - Output: Calendar events data to AI agent.  
  - Failure modes: OAuth token expiry, invalid date format, API errors.

- **Create a task in Google Tasks**  
  - Type: Google Tasks Tool  
  - Role: Creates new task with title provided by AI.  
  - Config: Task list ID hardcoded; title dynamically set from AI output.  
  - Input: AI agent triggers task creation.  
  - Output: Confirmation of task creation.  
  - Failure modes: OAuth errors, invalid task data.

- **Get many tasks in Google Tasks**  
  - Type: Google Tasks Tool  
  - Role: Retrieves all tasks from specified task list.  
  - Config: Task list ID hardcoded.  
  - Input: AI agent requests task list.  
  - Output: Task list data.  
  - Failure modes: OAuth failures, empty task list.

---

#### 2.5 Response Delivery

**Overview:**  
Delivers the AI-generated textual response back to the Telegram user in Markdown format.

**Nodes Involved:**  
- Telegram

**Node Details:**  

- **Telegram**  
  - Type: Telegram node (send message)  
  - Role: Sends text messages to Telegram user chat.  
  - Config: Text from AI output; chat ID from incoming Telegram message; Markdown parse mode; attribution disabled.  
  - Input: From AI assistant node output.  
  - Output: Message sent to Telegram user.  
  - Failure modes: Telegram API rate limits, invalid chat ID, network errors.  
  - On error: Set to continue without stopping the workflow.

---

#### 2.6 Supportive Sticky Notes

Sticky notes provide setup instructions, usage guidance, and links, including:

- Setup instructions for OpenRouter API key.  
- OpenAI API key setup for transcription.  
- Explanation of memory buffer role.  
- Google Tasks, Gmail, Google Calendar node purposes.  
- Video tutorial link: [https://youtu.be/ROgf5dVqYPQ](https://youtu.be/ROgf5dVqYPQ).  
- Introductory user instructions and community link: [AI Automation Engineering Community](https://www.skool.com/ai-automation-engineering-3014).

---

### 3. Summary Table

| Node Name                    | Node Type                               | Functional Role                        | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                           |
|------------------------------|---------------------------------------|-------------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------|
| Listen for incoming events    | Telegram Trigger                      | Entry point for Telegram messages   | —                           | Voice or Text                |                                                                                                     |
| Voice or Text                | Set                                   | Extract text or set empty string    | Listen for incoming events   | If                           | ## Process Telegram Request                                                                          |
| If                          | If                                    | Checks if message text is empty     | Voice or Text               | Get Voice File / Caylee AI    |                                                                                                     |
| Get Voice File              | Telegram node (file resource)           | Retrieves voice file from Telegram  | If (true branch)             | Transcribe a recording        |                                                                                                     |
| Transcribe a recording       | OpenAI (audio transcription)           | Converts voice audio to text        | Get Voice File              | Caylee AI                    | Uses OpenAI to convert voice to text. [OpenAI API Key setup link](https://platform.openai.com/api-keys) |
| Window Buffer Memory         | LangChain memory buffer (window)       | Maintains conversation memory       | —                           | Caylee AI                    | This node helps your agent remember the last few messages to stay on topic.                          |
| Caylee, AI Assistant 👩🏻‍🏫    | LangChain Agent                        | Core AI assistant processing user   | Voice or Text / Transcribe / Google nodes / Window Buffer Memory | Telegram                     | Caylee, your personal AI assistant: get email, calendar, tasks. Edit system message to adjust behavior. |
| OpenRouter                  | LangChain language model (OpenRouter)  | AI language model backend           | Caylee AI                    | Caylee AI                    | © 2025 Lucas Peyrin. Setup instructions in sticky note.                                            |
| Get Email                   | Gmail Tool                            | Fetches unread emails               | Caylee AI                    | Caylee AI                    | This node allows your agent access your Gmail.                                                      |
| Google Calendar             | Google Calendar Tool                  | Fetches calendar events             | Caylee AI                    | Caylee AI                    | This node allows your agent access your Google calendar.                                           |
| Create a task in Google Tasks| Google Tasks Tool                    | Creates new Google task             | Caylee AI                    | Caylee AI                    | This node allows your agent create and get tasks from Google Tasks.                                |
| Get many tasks in Google Tasks| Google Tasks Tool                   | Retrieves Google tasks list         | Caylee AI                    | Caylee AI                    |                                                                                                     |
| Telegram                   | Telegram node (send message)            | Sends AI response back to user      | Caylee AI                    | —                            | Sends message back to Telegram.                                                                     |
| Sticky Note                 | Sticky Note                          | Setup and instructional notes       | —                           | —                            | # Try It Out! Instructions and community link: [AI Automation Engineering Community](https://www.skool.com/ai-automation-engineering-3014) |
| Sticky Note1                | Sticky Note                          | OpenRouter API key setup             | —                           | —                            | In OpenRouter create API key and configure OpenRouter node.                                        |
| Sticky Note13               | Sticky Note                          | Overview of Caylee AI Assistant      | —                           | —                            | Caylee, your personal AI assistant: get email, calendar, and tasks.                                |
| Sticky Note15               | Sticky Note                          | Explains memory buffer node          | —                           | —                            | This node helps your agent remember the last few messages to stay on topic.                        |
| Sticky Note16               | Sticky Note                          | Explains Google Tasks integration    | —                           | —                            | This node allows your agent create and get tasks from Google Tasks.                                |
| Sticky Note18               | Sticky Note                          | Explains Gmail integration           | —                           | —                            | This node allows your agent access your Gmail.                                                    |
| Sticky Note19               | Sticky Note                          | Explains Google Calendar integration| —                           | —                            | This node allows your agent access your Google calendar.                                          |
| Sticky Note20               | Sticky Note                          | Explains OpenAI transcription setup | —                           | —                            | Uses OpenAI to convert voice to text. [OpenAI API Key setup link](https://platform.openai.com/api-keys) |
| Sticky Note2                | Sticky Note                          | Video tutorial link                  | —                           | —                            | [Video Tutorial](https://youtu.be/ROgf5dVqYPQ)                                                     |
| Sticky Note3                | Sticky Note                          | User instructions and community link| —                           | —                            | # Try It Out! with Telegram commands examples and community link.                                 |
| Sticky Note4                | Sticky Note                          | Explains Telegram response node     | —                           | —                            | Send message back to Telegram.                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node ("Listen for incoming events")**  
   - Type: Telegram Trigger  
   - Configure: Listen to "message" updates, set webhook ID.  
   - Credentials: Connect Telegram API account.  
   - Position: Start node.

2. **Add Set node ("Voice or Text")**  
   - Type: Set  
   - Parameters: Create field `text` with expression `{{$json?.message?.text || ""}}`.  
   - Connect from Telegram Trigger node.

3. **Add If node ("If")**  
   - Type: If (version 2)  
   - Condition: Check if `{{$json.message.text}}` is empty (string empty operator).  
   - Connect from "Voice or Text" node.

4. **Add Telegram node ("Get Voice File")**  
   - Type: Telegram  
   - Resource: File  
   - Parameters: File ID set to `{{$('Listen for incoming events').item.json.message.voice.file_id}}`.  
   - Credentials: Telegram API account.  
   - Connect from If node, True branch (empty text).

5. **Add OpenAI node ("Transcribe a recording")**  
   - Type: OpenAI (resource audio)  
   - Operation: Transcribe  
   - Credentials: OpenAI API key configured.  
   - Connect from "Get Voice File".

6. **Add LangChain Window Buffer Memory node ("Window Buffer Memory")**  
   - Type: @n8n/n8n-nodes-langchain.memoryBufferWindow  
   - Parameters: `sessionKey` set to `{{$('Listen for incoming events').first().json.message.from.id}}`, sessionIdType: customKey.  
   - Position near AI node.

7. **Add LangChain OpenRouter Language Model node ("OpenRouter")**  
   - Type: @n8n/n8n-nodes-langchain.lmChatOpenRouter  
   - Credentials: Create and attach OpenRouter API key credential.  
   - No specific parameters required.

8. **Add LangChain Agent node ("Caylee, AI Assistant 👩🏻‍🏫")**  
   - Type: @n8n/n8n-nodes-langchain.agent  
   - Parameters:  
     - Text input: `{{$json.text}}` (from transcription or text message).  
     - System message: Define assistant behavior (see node for example).  
     - Prompt type: Define.  
   - Connect inputs:  
     - AI Language Model: Connect from OpenRouter node.  
     - AI Memory: Connect from Window Buffer Memory node.  
     - AI Tools (multiple): Connect from Gmail, Google Calendar, Google Tasks nodes.  
   - Connect from If node False branch (text message) and from Transcribe node.

9. **Add Gmail Tool node ("Get Email")**  
   - Type: Gmail Tool  
   - Operation: Get All  
   - Filters: LabelIds: INBOX, UNREAD; Limit: 20.  
   - Credentials: Gmail OAuth2 account.  
   - Connect to AI Tools input of AI Agent node.

10. **Add Google Calendar Tool node ("Google Calendar")**  
    - Type: Google Calendar Tool  
    - Operation: Get All  
    - Parameters: TimeMin set dynamically from AI input using expression:  
      `={{$fromAI("date","the date after which to fetch the messages in format YYYY-MM-DDTHH:MM:SS")}}`  
    - Credentials: Google Calendar OAuth2 account.  
    - Connect to AI Tools input of AI Agent node.

11. **Add Google Tasks Tool node ("Create a task in Google Tasks")**  
    - Type: Google Tasks Tool  
    - Operation: Create  
    - Parameters: Task list ID pre-set, title dynamically set from AI input:  
      `={{$fromAI('Title', '', 'string')}}`  
    - Credentials: Google Tasks OAuth2 account.  
    - Connect to AI Tools input of AI Agent node.

12. **Add Google Tasks Tool node ("Get many tasks in Google Tasks")**  
    - Type: Google Tasks Tool  
    - Operation: Get All  
    - Task list ID pre-set.  
    - Credentials: Google Tasks OAuth2 account.  
    - Connect to AI Tools input of AI Agent node.

13. **Add Telegram node ("Telegram")**  
    - Type: Telegram (send message)  
    - Parameters:  
      - Text: `={{ $json.output }}` (AI agent output).  
      - Chat ID: `={{ $('Listen for incoming events').first().json.message.from.id }}`  
      - Additional fields: Parse mode Markdown, no attribution.  
    - Credentials: Telegram API account.  
    - Connect from AI Agent node output.  
    - On error: Set to continue (do not stop workflow).

14. **Connect Nodes:**  
    - Telegram Trigger → Voice or Text → If  
    - If True → Get Voice File → Transcribe a recording → Caylee AI  
    - If False → Caylee AI  
    - Caylee AI → Telegram (send message)  
    - Google services nodes → Caylee AI (as AI tools inputs)  
    - Window Buffer Memory → Caylee AI (AI memory input)  
    - OpenRouter → Caylee AI (AI language model input)

15. **Credential Setup:**  
    - Telegram API account (bot token)  
    - OpenAI API key (for transcription)  
    - OpenRouter API key (for AI language model)  
    - Gmail OAuth2 account  
    - Google Calendar OAuth2 account  
    - Google Tasks OAuth2 account

16. **Final Checks:**  
    - Webhook URLs registered properly with Telegram.  
    - All credentials authorized and tested.  
    - System message for AI assistant customized for desired behavior.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| # Try It Out! Launch Caylee—your personal AI assistant that handles voice & text via Telegram to manage your digital life. Commands example included. | Sticky Note3                                                                                     |
| OpenRouter API key creation instructions and node configuration details.                                              | Sticky Note1, OpenRouter node credential setup                                                  |
| OpenAI API key creation instructions for voice transcription.                                                        | Sticky Note20, Transcribe node credential setup                                                 |
| This node helps your agent remember the last few messages to stay on topic.                                           | Sticky Note15, Window Buffer Memory node                                                        |
| This node allows your agent create and get tasks from Google Tasks.                                                  | Sticky Note16, Google Tasks nodes                                                               |
| This node allows your agent access your Gmail.                                                                       | Sticky Note18, Gmail Tool node                                                                  |
| This node allows your agent access your Google calendar.                                                             | Sticky Note19, Google Calendar node                                                             |
| Video tutorial for setup and usage: [https://youtu.be/ROgf5dVqYPQ](https://youtu.be/ROgf5dVqYPQ)                       | Sticky Note2                                                                                     |
| AI Automation Engineering Community for support and learning: [https://www.skool.com/ai-automation-engineering-3014](https://www.skool.com/ai-automation-engineering-3014) | Sticky Note3                                                                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, a workflow automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.