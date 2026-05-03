Manage Notion To-Do Tasks via Telegram with Voice Messages & OpenAI

https://n8nworkflows.xyz/workflows/manage-notion-to-do-tasks-via-telegram-with-voice-messages---openai-11817


# Manage Notion To-Do Tasks via Telegram with Voice Messages & OpenAI

### 1. Workflow Overview

This workflow, titled **"Manage Notion To-Do Tasks via Telegram with Voice Messages & OpenAI"**, is designed to enable users to manage their Notion-based Todo lists through Telegram messages. It supports both text and voice input from Telegram, converting voice messages into text via OpenAI transcription. The AI-powered assistant ("Tard") processes the user’s requests, interacts strictly with Notion to create or search tasks/pages, and replies back to the user in Telegram with clear, formatted responses. The workflow is structured into the following logical blocks:

- **1.1 Telegram Input Reception:** Captures incoming Telegram messages and routes them based on message type (text or voice).
- **1.2 Voice Message Transcription:** Downloads voice messages from Telegram and converts them into text using OpenAI’s transcription.
- **1.3 Conversation Memory Handling:** Maintains short-term context of the conversation for a coherent dialogue experience.
- **1.4 AI Assistant Processing:** Uses a defined AI agent to interpret user requests, focusing exclusively on Notion Todo management.
- **1.5 Notion Integration:** Performs Notion operations like searching or creating pages/tasks based on AI outputs.
- **1.6 Telegram Reply:** Sends formatted responses back to the user via Telegram.
- **1.7 Optional and Disabled Integrations:** Includes Gmail and Google Calendar nodes, disabled by default, for future workflow expansion.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

**Overview:**  
This block listens for new Telegram messages (both text and voice) and routes them for appropriate processing.

**Nodes Involved:**  
- Listen for incoming events  
- Voice or Text  
- If  

**Node Details:**

- **Listen for incoming events**  
  - Type: Telegram Trigger  
  - Role: Entry point, listens for Telegram updates of type "message"  
  - Configuration: Uses webhook with Telegram API credentials  
  - Inputs: None (trigger node)  
  - Outputs: Emits incoming Telegram message JSON  
  - Edge Cases: Telegram webhook misconfiguration or API downtime may cause missed messages  

- **Voice or Text**  
  - Type: Set  
  - Role: Extracts message text or initializes empty string if absent  
  - Configuration: Sets a variable `text` to either the text content of the message or an empty string if none exists  
  - Inputs: From "Listen for incoming events"  
  - Outputs: Passes enriched JSON with `text` field  
  - Edge Cases: Messages without text or voice may cause empty `text`  

- **If**  
  - Type: If  
  - Role: Checks if the message text is empty to distinguish voice messages from text  
  - Configuration: Condition checks if `text` is empty string (true if voice)  
  - Inputs: From "Voice or Text"  
  - Outputs: Routes to "Get Voice File" (voice) or "Tard, AI Assistant" (text)  
  - Edge Cases: Messages that are neither voice nor text may cause unexpected routing  

---

#### 2.2 Voice Message Transcription

**Overview:**  
Downloads the voice file from Telegram and converts it into text via OpenAI transcription.

**Nodes Involved:**  
- Get Voice File  
- Transcribe a recording  

**Node Details:**

- **Get Voice File**  
  - Type: Telegram node  
  - Role: Downloads the voice message file from Telegram using `file_id`  
  - Configuration: Uses Telegram API credentials, accesses `message.voice.file_id` from input JSON  
  - Inputs: From "If" node (voice message branch)  
  - Outputs: Passes file data to transcription  
  - Edge Cases: Invalid file_id, Telegram API errors, or missing voice message payload  

- **Transcribe a recording**  
  - Type: OpenAI Audio Transcription node  
  - Role: Sends audio file to OpenAI to transcribe voice to text  
  - Configuration: Uses OpenAI API credentials, operation set to `transcribe` on audio resource  
  - Inputs: From "Get Voice File"  
  - Outputs: Text transcription sent downstream  
  - Edge Cases: Audio format unsupported, API quota exceeded, network errors  

---

#### 2.3 Conversation Memory Handling

**Overview:**  
Stores recent messages per user session to maintain context during short conversations.

**Nodes Involved:**  
- Window Buffer Memory  

**Node Details:**

- **Window Buffer Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Keeps recent conversational data keyed by Telegram user ID  
  - Configuration: Uses custom session key derived from Telegram sender ID (`message.from.id`)  
  - Inputs: From "Tard, AI Assistant" (implied by connection)  
  - Outputs: Provides memory context to AI agent  
  - Edge Cases: Session key mismatch, memory overflow or reset causing loss of context  

---

#### 2.4 AI Assistant Processing

**Overview:**  
Interprets user text input and manages Todo list tasks in Notion. The AI agent is strictly scoped to Notion-related queries.

**Nodes Involved:**  
- Tard, AI Assistant  
- OpenAI Chat Model  
- Window Buffer Memory (memory input)  
- Create a page in Notion  
- Search a page in Notion  

**Node Details:**

- **Tard, AI Assistant**  
  - Type: Langchain Agent Node  
  - Role: Core AI logic interpreting user commands, deciding when to search/create Notion tasks  
  - Configuration:  
    - System message defines assistant persona, date, scope limited to Notion Todo management  
    - Behaves informally, concise, avoids inventing info, always includes Notion links if relevant  
  - Inputs:  
    - Text input from transcription or direct text message  
    - Memory context from Window Buffer Memory  
    - Language model connection to OpenAI Chat Model  
    - Tool connections to Notion nodes for search/create tasks  
  - Outputs: Passes reply text to Telegram node  
  - Edge Cases: AI model errors, invalid or ambiguous user requests, Notion API failures  
  - Version: 1.6  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides GPT-4.1-mini language model for AI assistant  
  - Configuration: Uses GPT-4.1-mini model with default options and credentials  
  - Inputs: From AI assistant node for chat completion  
  - Outputs: Text completions to AI assistant  
  - Edge Cases: API limits, latency, model update requirements  

- **Create a page in Notion**  
  - Type: Notion Tool node  
  - Role: Creates new pages/tasks in a specified Notion database/page  
  - Configuration:  
    - Target page URL set to the Todo List database  
    - Title and simplification flags passed from AI assistant outputs  
  - Inputs: From AI assistant (tool invocation)  
  - Outputs: Confirmation data or created page info to AI assistant  
  - Edge Cases: Notion API authentication errors, invalid page URL, data validation failures  

- **Search a page in Notion**  
  - Type: Notion Tool node  
  - Role: Searches existing Notion pages/tasks matching user query  
  - Configuration: Uses search text from AI assistant, returns all matches  
  - Inputs: From AI assistant (tool invocation)  
  - Outputs: Search results to AI assistant  
  - Edge Cases: Empty or too broad search text, Notion API limits or errors  

---

#### 2.5 Telegram Reply

**Overview:**  
Sends the AI assistant’s response back to the Telegram user in a clean and formatted manner.

**Nodes Involved:**  
- Telegram  

**Node Details:**

- **Telegram**  
  - Type: Telegram node (send message)  
  - Role: Sends formatted text reply message to the Telegram user  
  - Configuration:  
    - Uses chat ID from the incoming message's sender ID  
    - Sends text from AI assistant output  
    - Format set to Markdown, disables attribution appending  
  - Inputs: From AI assistant output  
  - Outputs: Message delivery confirmation  
  - Edge Cases: Telegram API errors, invalid chat ID, message formatting issues  

---

#### 2.6 Optional and Disabled Integrations

**Overview:**  
The workflow includes disabled nodes for Gmail and Google Calendar integrations, intended for future expansion.

**Nodes Involved:**  
- Get Email (disabled)  
- Send Email (disabled)  
- Google Calendar (disabled)  

**Node Details:**

- **Get Email & Send Email**  
  - Type: Gmail Tool  
  - Role: Fetches and sends emails (currently disabled)  
  - Configuration: Filters for unread emails, parameters for sending email subject and message from AI overrides  
  - Edge Cases: OAuth issues, email quota, disabled status  

- **Google Calendar**  
  - Type: Google Calendar Tool  
  - Role: Fetches calendar events (disabled)  
  - Configuration: Time window parameters set via AI overrides, uses calendar ID  
  - Edge Cases: Disabled node, API access errors  

---

### 3. Summary Table

| Node Name             | Node Type                           | Functional Role                                   | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                       |
|-----------------------|-----------------------------------|-------------------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Listen for incoming events | Telegram Trigger                  | Entry point, receives Telegram messages          | None                        | Voice or Text              | ## Telegram Input\n\nReceives incoming Telegram messages.\nRoutes text vs. voice messages to the correct processing path.         |
| Voice or Text          | Set                               | Extracts text or empty string from Telegram input| Listen for incoming events  | If                         | ## Telegram Input\n\nReceives incoming Telegram messages.\nRoutes text vs. voice messages to the correct processing path.         |
| If                    | If                                | Routes messages to voice or text processing       | Voice or Text               | Get Voice File, Tard, AI Assistant | ## Telegram Input\n\nReceives incoming Telegram messages.\nRoutes text vs. voice messages to the correct processing path.         |
| Get Voice File         | Telegram                          | Downloads voice message file from Telegram        | If (voice branch)           | Transcribe a recording      | ## Voice to Text\n\nDownloads Telegram voice messages and converts them into text using OpenAI.                                   |
| Transcribe a recording | OpenAI Audio Transcription        | Transcribes voice recordings to text              | Get Voice File              | Tard, AI Assistant          | ## Voice to Text\n\nDownloads Telegram voice messages and converts them into text using OpenAI.                                   |
| Window Buffer Memory   | Langchain Memory Buffer Window    | Maintains conversation context                     | (From Listen for incoming events indirectly) | Tard, AI Assistant          | ## Conversation Memory\n\nStores recent messages so the assistant can stay on topic during short conversations.                   |
| Tard, AI Assistant     | Langchain Agent                   | Core AI assistant for interpreting and managing Notion Todos | If (text branch), Transcribe a recording, OpenAI Chat Model, Window Buffer Memory, Create/Search Notion | Telegram                   | ## AI Assistant\n\nUnderstands user requests and decides when to search or create tasks in Notion.\nThe agent is strictly focused on Todo management. |
| OpenAI Chat Model      | Langchain OpenAI Chat Model       | Provides GPT-4.1-mini language model              | Tard, AI Assistant          | Tard, AI Assistant          |                                                                                                                                  |
| Create a page in Notion| Notion Tool                      | Creates new pages/tasks in Notion                  | Tard, AI Assistant          | Tard, AI Assistant          | ## Notion Integration\n\nCreates and searches tasks and pages inside Notion based on the user request.                            |
| Search a page in Notion| Notion Tool                      | Searches Notion pages/tasks                         | Tard, AI Assistant          | Tard, AI Assistant          | ## Notion Integration\n\nCreates and searches tasks and pages inside Notion based on the user request.                            |
| Telegram               | Telegram                          | Sends replies back to Telegram users                | Tard, AI Assistant          | None                       | ## Reply to User\n\nSends the assistant’s response back to Telegram in a clean, readable format.                                  |
| Get Email              | Gmail Tool (disabled)             | (Disabled) Fetches unread emails                    | None                        | Tard, AI Assistant          | ## Optional Integrations\n\nGmail and Google Calendar nodes are disabled by default.\nEnable and configure them only if you want to extend the workflow beyond Notion. |
| Send Email             | Gmail Tool (disabled)             | (Disabled) Sends emails                             | None                        | Tard, AI Assistant          | ## Optional Integrations\n\nGmail and Google Calendar nodes are disabled by default.\nEnable and configure them only if you want to extend the workflow beyond Notion. |
| Google Calendar        | Google Calendar Tool (disabled)  | (Disabled) Fetches Google Calendar events          | None                        | Tard, AI Assistant          | ## Optional Integrations\n\nGmail and Google Calendar nodes are disabled by default.\nEnable and configure them only if you want to extend the workflow beyond Notion. |
| Sticky - Overview      | Sticky Note                      | Overview and general workflow explanation           | None                        | None                       | ## How it works\n\nThis workflow turns Telegram messages (text or voice) into actions in Notion.\nUsers can send a message to a Telegram bot, and the workflow will:\n• Convert voice messages to text using OpenAI\n• Keep short conversational context\n• Use an AI agent to read, search, and create tasks in Notion\n• Reply back to the user in Telegram with clear, formatted results\n\nThe assistant is focused on managing a Todo List in Notion.\nIt does not interact with email or calendars, even if nodes exist for future expansion. |
| Sticky - Telegram Input| Sticky Note                      | Notes on Telegram input handling                    | None                        | None                       | ## Telegram Input\n\nReceives incoming Telegram messages.\nRoutes text vs. voice messages to the correct processing path.         |
| Sticky - Voice to Text | Sticky Note                      | Notes on voice message transcription                | None                        | None                       | ## Voice to Text\n\nDownloads Telegram voice messages and converts them into text using OpenAI.                                   |
| Sticky - Conversation Memory | Sticky Note                | Notes on conversation memory                         | None                        | None                       | ## Conversation Memory\n\nStores recent messages so the assistant can stay on topic during short conversations.                   |
| Sticky - AI Assistant  | Sticky Note                      | Notes on AI Assistant focus and behavior             | None                        | None                       | ## AI Assistant\n\nUnderstands user requests and decides when to search or create tasks in Notion.\nThe agent is strictly focused on Todo management. |
| Sticky - Notion Integration | Sticky Note                 | Notes on Notion operations                           | None                        | None                       | ## Notion Integration\n\nCreates and searches tasks and pages inside Notion based on the user request.                            |
| Sticky - Reply to User | Sticky Note                      | Notes on replying to Telegram                        | None                        | None                       | ## Reply to User\n\nSends the assistant’s response back to Telegram in a clean, readable format.                                  |
| Sticky - Optional Integrations | Sticky Note               | Notes on optional Gmail and Calendar nodes           | None                        | None                       | ## Optional Integrations\n\nGmail and Google Calendar nodes are disabled by default.\nEnable and configure them only if you want to extend the workflow beyond Notion. |
| Sticky - Credit       | Sticky Note                      | Credit to workflow creator                           | None                        | None                       | ### Credit\nBuilt / customized by **weblane.co.il**                                                                                 |
| Sticky - Overview1    | Sticky Note                      | Setup instructions                                  | None                        | None                       | ## Setup steps\n\n1. Create a Telegram bot and connect it to the Telegram Trigger node\n2. Add your OpenAI API key (used for voice transcription and the AI agent)\n3. Connect your Notion account and update the Page/Database URL\n4. (Optional) Enable Gmail or Google Calendar nodes if you plan to extend the workflow\n5. Activate the workflow and start chatting with the bot\n |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure with Telegram API credentials (Telegram Bot token)  
   - Set updates to listen for: `message`  
   - Connect webhook for incoming Telegram messages  

2. **Create Set node "Voice or Text"**  
   - Type: Set  
   - Add field `text` with expression: `{{$json?.message?.text || ""}}`  
   - Connect input from Telegram Trigger node  

3. **Create If node**  
   - Type: If  
   - Condition: Check if `text` is empty string  
   - If TRUE (voice message), route to voice processing path  
   - If FALSE (text message), route directly to AI assistant  

4. **Create Telegram node "Get Voice File"**  
   - Type: Telegram  
   - Operation: Get file  
   - File ID: `{{$json.message.voice.file_id}}`  
   - Use Telegram API credentials  
   - Connect from If node (TRUE branch)  

5. **Create OpenAI node "Transcribe a recording"**  
   - Type: OpenAI Audio Transcription  
   - Operation: Transcribe  
   - Credentials: OpenAI API key  
   - Connect from "Get Voice File" node  

6. **Create Langchain Memory Buffer Window node "Window Buffer Memory"**  
   - Type: Langchain Memory Buffer Window  
   - Session Key: `{{$json.message.from.id}}` (Telegram user ID)  
   - Session ID Type: customKey  

7. **Create Langchain OpenAI Chat Model node "OpenAI Chat Model"**  
   - Type: Langchain OpenAI Chat Model  
   - Model: GPT-4.1-mini  
   - Credentials: OpenAI API key  

8. **Create Langchain Agent node "Tard, AI Assistant"**  
   - Type: Langchain Agent  
   - Text input: `{{$json.text}}` (from transcription or direct text)  
   - System message (assistant prompt) to focus strictly on Notion Todo management, natural tone, etc. (refer to system message content in workflow)  
   - Connect AI language model input to "OpenAI Chat Model"  
   - Connect AI memory input to "Window Buffer Memory"  
   - Connect AI tools input to the Notion nodes (below)  

9. **Create Notion Tool node "Search a page in Notion"**  
   - Operation: Search  
   - Text: From AI assistant parameter `"Search_Text"`  
   - Return all results  
   - Connect credentials to Notion API account  
   - Connect as AI tool input to "Tard, AI Assistant"  

10. **Create Notion Tool node "Create a page in Notion"**  
    - Operation: Create page  
    - Page ID: Link to your Notion Todo List page or database URL  
    - Title: From AI assistant parameter `"Title"`  
    - Simple flag: From AI assistant parameter `"Simplify"` (boolean)  
    - Connect credentials to Notion API account  
    - Connect as AI tool input to "Tard, AI Assistant"  

11. **Connect "If" node FALSE branch (text path) and "Transcribe a recording" node output to "Tard, AI Assistant"**  

12. **Create Telegram node "Telegram" for replies**  
    - Operation: Send message  
    - Chat ID: `{{$json.message.from.id}}` (Telegram user ID)  
    - Text: `{{$json.output}}` (AI assistant response)  
    - Parse mode: Markdown  
    - Disable attribution appending  
    - Connect input from "Tard, AI Assistant" output  

13. **Disable or optionally add Gmail and Google Calendar nodes** for future extensions, configuring credentials and parameters as needed.  

14. **Add Sticky Notes** (optional) to document the workflow visually, following the content provided for context and setup instructions.  

15. **Activate the workflow** and test by sending text or voice messages to your Telegram bot.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                      |
|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow uses a Langchain AI agent named "Tard" focused exclusively on Notion Todo management and general AI questions.       | Internal workflow design                             |
| Voice messages are transcribed using OpenAI's audio transcription capabilities to convert speech to text seamlessly.               | OpenAI API documentation                            |
| Telegram bot must be created and connected correctly to receive messages; use Telegram BotFather to create the bot and get token. | Telegram Bot API documentation                       |
| Notion integration requires a Notion API key and the URL of the target Todo List page or database.                                 | Notion API docs and developer portal                |
| Gmail and Google Calendar nodes are disabled by default for optional extensions; enable only if required and configure OAuth2.     | Gmail and Google Calendar API documentation          |
| Workflow credit: Built and customized by **weblane.co.il**                                                                         | https://weblane.co.il                                |
| Setup steps note: Create Telegram bot, add OpenAI key, connect Notion, optionally enable Gmail/Calendar, then activate workflow.   | Sticky Note in workflow                              |

---

**Disclaimer:** The provided information is exclusively extracted and compiled from an automated n8n workflow implementation. All data and integrations comply with current legal and platform policies.