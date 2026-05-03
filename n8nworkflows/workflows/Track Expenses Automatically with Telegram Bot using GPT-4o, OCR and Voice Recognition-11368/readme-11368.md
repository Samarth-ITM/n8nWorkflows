Track Expenses Automatically with Telegram Bot using GPT-4o, OCR and Voice Recognition

https://n8nworkflows.xyz/workflows/track-expenses-automatically-with-telegram-bot-using-gpt-4o--ocr-and-voice-recognition-11368


# Track Expenses Automatically with Telegram Bot using GPT-4o, OCR and Voice Recognition

---

# Reference Document for Workflow:  
**Track Expenses Automatically with Telegram Bot using GPT-4o, OCR and Voice Recognition**

---

### 1. Workflow Overview

This workflow implements a **Personal Expense Tracker Telegram Bot** leveraging GPT-4o AI models, OCR, and voice recognition to automatically process, store, and retrieve expense data from user inputs. Users can send expenses as text, voice messages, photos, or documents (PDFs, images), and the bot extracts structured expense information, saves it categorized by month, and provides summaries or detailed queries on expenses.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Authorization:**  
  Captures Telegram messages (text, audio, photo, file), verifies user authorization (single-user lock), and routes input by type.

- **1.2 Preprocessing & File Extraction:**  
  Downloads attached files/photos/audio, applies OCR or transcription services to extract text from documents and voice notes.

- **1.3 AI Processing (Root Agent & Expense Assistant):**  
  Routes textual input to an AI agent that manages expense extraction, validation, storage, and retrieval using specialized AI assistant tools (ExpenseAssistant) and calculators.

- **1.4 Storage & State Management:**  
  Uses Ainoflow MCP JSON storage for persistent expense data and app settings, supports cleanup and deletion operations.

- **1.5 Telegram Response Handling:**  
  Sends replies, welcome messages, authorization denials, and processing notifications back to the user on Telegram.

- **1.6 Manual Cleanup (Optional):**  
  Provides a manual trigger to clear all stored expense data in Ainoflow.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Authorization

**Overview:**  
Receives incoming Telegram messages, ensures only a single authorized user interacts with the bot, and directs messages by type.

**Nodes Involved:**  
- Trigger  
- Input  
- GetAppSettings  
- IfFirstRun  
- SaveAppSettings  
- IfAccessAllowed  
- NotAuthorizedMessage  
- Switch  
- SendProcessing  
- GetExecutionState  
- IfStateExist  
- DeleteProcessing  

**Node Details:**

- **Trigger (Telegram Trigger):**  
  - Listens for Telegram updates with message content including photos and voice messages, downloads media.  
  - Uses Telegram credentials named "AinoflowExpenses".  
  - Triggers workflow on any message event.

- **Input (Set node):**  
  - Extracts user's language code from Telegram message (`message.from.language_code`), defaults to "en".  
  - Passes this language info downstream for OCR/transcription.

- **GetAppSettings (HTTP Request):**  
  - Fetches stored configuration (including authorized chat_id) from Ainoflow JSON storage under category `config/expense-app-settings`.  
  - Uses Bearer token authentication.

- **IfFirstRun (If node):**  
  - Checks if this is the first run by checking if stored config returned 404 error (no settings).  
  - If first run, proceeds to save user chat_id and related info.

- **SaveAppSettings (HTTP Request):**  
  - Stores user chat_id, first_name, and start_date for authorization locking.  
  - Uses PUT to Ainoflow JSON storage under `config/expense-app-settings`.

- **IfAccessAllowed (If node):**  
  - Authorizes user by comparing current Telegram chat_id with stored chat_id in config or allows if no config (error 404).  
  - If unauthorized, routes to NotAuthorizedMessage.

- **NotAuthorizedMessage (Telegram node):**  
  - Sends "You are not authorized to use this bot." message to unauthorized users.

- **Switch (Switch node):**  
  - Routes incoming messages by type: `/start` command, text message, audio message, file upload, or photo upload.  
  - Uses presence and content of various Telegram message properties to detect type.

- **SendProcessing (Telegram node):**  
  - Sends a "..." message to notify the user processing has started.

- **GetExecutionState (HTTP Request):**  
  - Gets current processing state for the update_id from Ainoflow storage to manage message deletion later.

- **IfStateExist (If node):**  
  - Checks if message_id and chat_id exist in stored execution state to delete "processing" message.

- **DeleteProcessing (Telegram node):**  
  - Deletes the temporary "processing" message to clean up user chat.

**Edge Cases & Failure Types:**  
- Unauthorized access attempts.  
- Missing or corrupted config data.  
- Telegram API failures (message send/delete).  
- Network or authentication errors with Ainoflow storage.

---

#### 1.2 Preprocessing & File Extraction

**Overview:**  
Processes media files from Telegram messages—downloads voice, photo, or document files, extracts text using OCR or audio transcription.

**Nodes Involved:**  
- GetAudioFile  
- TranscribeAudio  
- GetAttachedFile  
- ExtractFileText  
- GetAttachedPhoto  
- ExtractImageText  
- FileOutput  
- ImageOutput  
- AudioOutput  

**Node Details:**

- **GetAudioFile (Telegram node):**  
  - Downloads voice message file from Telegram using `voice.file_id`.

- **TranscribeAudio (HTTP Request):**  
  - Sends audio file to Ainoflow API for speech-to-text transcription.  
  - Uses document language from Input node for language parameter.

- **GetAttachedFile (Telegram node):**  
  - Downloads attached document (e.g., PDF invoice) using file_id.

- **ExtractFileText (HTTP Request):**  
  - Sends document file to Ainoflow API to extract text (OCR or PDF parsing).  
  - Returns extracted text or error message.

- **GetAttachedPhoto (Telegram node):**  
  - Downloads the highest resolution photo attached to the message.

- **ExtractImageText (HTTP Request):**  
  - Sends photo to Ainoflow OCR service (PaddleOCR model) to extract text.

- **FileOutput, ImageOutput, AudioOutput (Set nodes):**  
  - Format extracted text with prefix:  
    `"User uploaded document / photo:\n\n[caption]\n[extracted text or error message]"`  
  - Attach chat_id for downstream AI processing.

**Edge Cases & Failure Types:**  
- File download errors from Telegram.  
- OCR/transcription service failures or timeouts.  
- Unsupported file types or unreadable content.  
- Missing captions or language metadata.

---

#### 1.3 AI Processing (Root Agent & Expense Assistant)

**Overview:**  
The core logic where user text (direct or extracted) is processed by a root AI agent which forwards all expense-related queries to a dedicated Expense Assistant AI. The assistant extracts, validates, stores, and summarizes expenses using embedded MCP storage tools and calculator utilities.

**Nodes Involved:**  
- AgentInput  
- AIAgent  
- ExpenseAssistant  
- Gpt4o  
- Sonnet45  
- JsonStorageMcp  
- Calculator  
- SimpleMemory  
- AssitantMemory  
- Think  
- Think1  
- ReplyText  

**Node Details:**

- **AgentInput (Set node):**  
  - Prepares the input text and chat_id for the AI agent.

- **AIAgent (Langchain Agent node):**  
  - Root AI agent receives user messages and always forwards to ExpenseAssistant without modification.  
  - System message defines its role as a relay only, not to parse or summarize itself.  
  - Uses GPT-4o model (OpenRouter credentials).  
  - Uses SimpleMemory buffer for context (last 30 interactions).

- **ExpenseAssistant (Langchain Agent Tool node):**  
  - Performs expense extraction, validation, storage, retrieval, and deletion.  
  - Uses JsonStorageMcp tool (for Ainoflow JSON storage), Calculator tool (for sums/statistics), and AssitantMemory for session context.  
  - System prompt defines allowed categories, storage data structure, input formats, error handling, and response formatting.  
  - Critical rules: always add expense immediately without confirmation, handle multilingual input, parse all input types.

- **Gpt4o and Sonnet45 (Language Model nodes):**  
  - Linked to ExpenseAssistant for language model calls; Gpt4o uses GPT-4o, Sonnet45 uses Claude 4.5.  
  - Both authenticate via OpenRouter API key.

- **JsonStorageMcp (MCP Client Tool):**  
  - Provides interface to store, retrieve, update, delete JSON expense data in Ainoflow storage.

- **Calculator (ToolCalculator):**  
  - Performs numeric operations like sums and averages for expense statistics.

- **SimpleMemory and AssitantMemory (MemoryBufferWindow):**  
  - Manage conversational context for AI agents keyed by chat_id.

- **Think and Think1 (ToolThink):**  
  - Support reasoning steps internally for AI orchestration.

- **ReplyText (Telegram node):**  
  - Sends AI assistant's textual reply back to user on Telegram.  
  - Uses Markdown format (single asterisk for bold).

**Edge Cases & Failure Types:**  
- AI model or API errors (timeouts, quota limits).  
- Failure to extract valid expense data (missing amount, invalid date).  
- Storage failures (network, auth).  
- Session memory overflows or context loss.  
- Incorrect formatting or user input ambiguity.

---

#### 1.4 Storage & State Management

**Overview:**  
Handles persistent storage of expenses and configuration, manages execution state to track processing messages, and supports manual data cleanup.

**Nodes Involved:**  
- SaveExecutionState  
- GetExecutionState  
- ForEachCategory  
- ForEachCategoryItem  
- DeleteItem  
- Cleanup (Manual Trigger)  

**Node Details:**

- **SaveExecutionState (HTTP Request):**  
  - Saves message_id and chat_id of the "processing" message to Ainoflow JSON storage keyed by Telegram update_id for later deletion.

- **GetExecutionState (HTTP Request):**  
  - Retrieves saved processing message info to enable deletion.

- **ForEachCategory (HTTP Request):**  
  - Iterates over all categories in JSON storage for cleanup purposes.

- **ForEachCategoryItem (HTTP Request):**  
  - Iterates over all items (expenses) in a given category.

- **DeleteItem (HTTP Request):**  
  - Deletes a specific expense item by category and key.

- **Cleanup (Manual Trigger):**  
  - Manual node to trigger full deletion of all stored expenses (dangerous operation).

**Edge Cases & Failure Types:**  
- Partial cleanup failures.  
- Concurrent access conflicts.  
- API rate limits or auth failures.

---

#### 1.5 Telegram Response Handling

**Overview:**  
Sends various reply messages to the user including welcome, authorization errors, processing notifications, and final expense tracking responses.

**Nodes Involved:**  
- WelcomeMessage  
- NotAuthorizedMessage  
- SendProcessing  
- ReplyText  
- DeleteProcessing  

**Node Details:**

- **WelcomeMessage (Telegram node):**  
  - Sends introductory text on `/start` command with instructions for usage.

- **NotAuthorizedMessage (Telegram node):**  
  - Alerts unauthorized users.

- **SendProcessing (Telegram node):**  
  - Shows "..." during processing.

- **ReplyText (Telegram node):**  
  - Sends formatted AI-generated replies.

- **DeleteProcessing (Telegram node):**  
  - Deletes temporary processing message to keep chat clean.

**Edge Cases & Failure Types:**  
- Telegram API rate limits.  
- Message deletion failures (message already deleted or missing).

---

#### 1.6 Manual Cleanup (Optional)

**Overview:**  
Provides a manual trigger node to erase all stored expense data in Ainoflow, used for resetting or testing.

**Nodes Involved:**  
- Cleanup (Manual Trigger)  
- ForEachCategory  
- ForEachCategoryItem  
- DeleteItem  

**Node Details:**

- **Cleanup (Manual Trigger):**  
  - Operator-triggered node to start bulk deletion process.

- **ForEachCategory, ForEachCategoryItem, DeleteItem:**  
  - Iterates all categories and items, deletes every stored record.

**Edge Cases & Failure Types:**  
- Irreversible data loss risk.  
- Partial deletion if errors occur midway.

---

### 3. Summary Table

| Node Name         | Node Type                               | Functional Role                                | Input Node(s)            | Output Node(s)                | Sticky Note                                                                                                         |
|-------------------|---------------------------------------|------------------------------------------------|--------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Trigger           | Telegram Trigger                      | Entry point for Telegram messages               |                          | Input                        | ## Message Trigger                                                                                                   |
| Input             | Set                                  | Extract language code and enrich input          | Trigger                  | GetAppSettings               |                                                                                                                     |
| GetAppSettings    | HTTP Request                        | Load stored app/user config from Ainoflow       | Input                    | IfFirstRun, IfAccessAllowed  |                                                                                                                     |
| IfFirstRun        | If                                   | Detect first-time user                            | GetAppSettings           | SaveAppSettings              | ## First Run - Save Config                                                                                           |
| SaveAppSettings   | HTTP Request                        | Store user chat_id and info                       | IfFirstRun               | IfAccessAllowed              |                                                                                                                     |
| IfAccessAllowed   | If                                   | Check if user is authorized                       | GetAppSettings, SaveAppSettings | Switch, SendProcessing, GetExecutionState | ## Ensure Bot Privacy - locks access to one user                                                                      |
| NotAuthorizedMessage | Telegram                            | Reply unauthorized access message                 | IfAccessAllowed (no)      |                              |                                                                                                                     |
| Switch            | Switch                              | Route message types (/start, text, audio, file, photo) | IfAccessAllowed (yes)     | WelcomeMessage, TextOutput, GetAudioFile, GetAttachedFile, GetAttachedPhoto | ## Get Chat Message / Audio Text                                                                                        |
| WelcomeMessage    | Telegram                            | Send welcome text on /start                        | Switch                   |                              | ## Start Message Reply                                                                                               |
| TextOutput        | Set                                  | Prepare text message for AI agent                 | Switch                   | AgentInput                   |                                                                                                                     |
| GetAudioFile      | Telegram                            | Download voice file                               | Switch                   | TranscribeAudio              |                                                                                                                     |
| TranscribeAudio   | HTTP Request                        | Convert voice audio to text                        | GetAudioFile             | AudioOutput                  |                                                                                                                     |
| AudioOutput       | Set                                  | Prepare transcribed audio text for AI agent       | TranscribeAudio          | AgentInput                   |                                                                                                                     |
| GetAttachedFile   | Telegram                            | Download attached document                         | Switch                   | ExtractFileText              |                                                                                                                     |
| ExtractFileText   | HTTP Request                        | Extract text from file (OCR/PDF parsing)           | GetAttachedFile          | FileOutput                   |                                                                                                                     |
| FileOutput        | Set                                  | Format extracted file text for AI agent            | ExtractFileText          | AgentInput                   |                                                                                                                     |
| GetAttachedPhoto  | Telegram                            | Download attached photo                            | Switch                   | ExtractImageText             |                                                                                                                     |
| ExtractImageText  | HTTP Request                        | OCR photo to extract text                          | GetAttachedPhoto         | ImageOutput                  |                                                                                                                     |
| ImageOutput       | Set                                  | Format photo text for AI agent                      | ExtractImageText         | AgentInput                   |                                                                                                                     |
| AgentInput        | Set                                  | Prepare final input text and chat_id for AI agent | TextOutput, AudioOutput, FileOutput, ImageOutput | AIAgent                    |                                                                                                                     |
| AIAgent           | Langchain Agent                    | Root AI agent routing messages to ExpenseAssistant | AgentInput               | ReplyText                    | ## Root Agent Processing                                                                                             |
| ExpenseAssistant  | Langchain Agent Tool              | Expense extraction, validation, storage, retrieval | Gpt4o, Sonnet45, Calculator, JsonStorageMcp | AIAgent (via ExpenseAssistant) | ## Expense Assistant                                                                                                |
| Gpt4o             | LM Chat OpenRouter                | GPT-4o model for ExpenseAssistant                  | ExpenseAssistant         | ExpenseAssistant             |                                                                                                                     |
| Sonnet45          | LM Chat OpenRouter                | Claude Sonnet 4.5 model for ExpenseAssistant       | ExpenseAssistant         | ExpenseAssistant             |                                                                                                                     |
| JsonStorageMcp    | MCP Client Tool                  | JSON storage tool for Ainoflow                      | ExpenseAssistant         | ExpenseAssistant             |                                                                                                                     |
| Calculator        | Calculator Tool                 | Performs sums and statistics                        | ExpenseAssistant         | ExpenseAssistant             |                                                                                                                     |
| SimpleMemory      | Memory Buffer Window             | Context memory for root AI agent                    | AIAgent                  | AIAgent                      |                                                                                                                     |
| AssitantMemory    | Memory Buffer Window             | Context memory for ExpenseAssistant                  | ExpenseAssistant         | ExpenseAssistant             |                                                                                                                     |
| Think             | ToolThink                      | Internal reasoning tool for ExpenseAssistant        | ExpenseAssistant         | ExpenseAssistant             |                                                                                                                     |
| Think1            | ToolThink                      | Internal reasoning tool for AIAgent                  | AIAgent                  | AIAgent                      |                                                                                                                     |
| ReplyText         | Telegram                        | Sends AI-generated reply to user                     | AIAgent                  |                              | ## Result / Reply Message                                                                                            |
| SendProcessing    | Telegram                        | Sends "..." processing message                        | Switch, IfAccessAllowed   | SaveExecutionState           | ## Add processing started message                                                                                    |
| SaveExecutionState | HTTP Request                  | Saves processing message info for later deletion     | SendProcessing           | GetExecutionState            |                                                                                                                     |
| GetExecutionState | HTTP Request                  | Retrieves processing message info                      | IfAccessAllowed          | IfStateExist                 |                                                                                                                     |
| IfStateExist      | If                             | Checks if processing message info exists              | GetExecutionState        | DeleteProcessing             |                                                                                                                     |
| DeleteProcessing  | Telegram                       | Deletes temporary processing message                   | IfStateExist             |                              | ## Remove processing message                                                                                         |
| Cleanup           | Manual Trigger                | Manual trigger to delete all stored expense data       |                          | ForEachCategory              | ## Cleanup / Reset                                                                                                   |
| ForEachCategory   | HTTP Request                  | List all categories for cleanup                          | Cleanup                  | ForEachCategoryItem          |                                                                                                                     |
| ForEachCategoryItem | HTTP Request                | List all items in category for cleanup                   | ForEachCategory          | DeleteItem                   |                                                                                                                     |
| DeleteItem        | HTTP Request                  | Delete single expense item                                | ForEachCategoryItem      |                              |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials:**  
   - Create Telegram Bot via BotFather, obtain API token.  
   - Add Telegram API credentials in n8n named "AinoflowExpenses".

2. **Setup OpenRouter Credentials:**  
   - Create API key for OpenRouter (https://docs.n8n.io/integrations/builtin/credentials/openrouter/)  
   - Add credentials in n8n named "Openrouter".

3. **Setup Ainoflow Credentials:**  
   - Sign up at https://www.ainoflow.io/signup  
   - Obtain Bearer token for Ainoflow API.  
   - Add HTTP Bearer Auth credentials named "AinoflowPrimary".

4. **Add Telegram Trigger Node:**  
   - Configure to listen to "message" updates, download media, extra large images.  
   - Use "AinoflowExpenses" credentials.  
   - Save webhook ID for integration.

5. **Add Input Set Node:**  
   - Extract `document_language` from incoming Telegram message language code or default "en".

6. **Add HTTP Request Node: GetAppSettings:**  
   - GET https://api.ainoflow.io/api/v1/storage/json/config/expense-app-settings  
   - Use "AinoflowPrimary" Bearer token.  
   - On error continue output.

7. **Add If Node: IfFirstRun:**  
   - Condition: If GetAppSettings returns 404 error (no settings), true branch.

8. **Add HTTP Request Node: SaveAppSettings:**  
   - PUT https://api.ainoflow.io/api/v1/storage/json/config/expense-app-settings  
   - Body parameters: chat_id, first_name, start_date from Trigger message.  
   - Use "AinoflowPrimary" credentials.

9. **Add If Node: IfAccessAllowed:**  
   - Condition: chat_id from Trigger equals saved chat_id or GetAppSettings returns 404 (first run).  
   - True branch continues, false branch to NotAuthorizedMessage.

10. **Add Telegram Node: NotAuthorizedMessage:**  
    - Send "You are not authorized to use this bot." to chat_id.

11. **Add Switch Node:**  
    - Route messages by type: `/start`, text, audio, file, photo.  
    - Use conditions on message text, document.file_id, voice.file_id, photo array.

12. **Add Telegram Node: WelcomeMessage:**  
    - On `/start` command, send welcome message with instructions.

13. **Add nodes for media processing:**  
    - GetAudioFile → TranscribeAudio → AudioOutput (Set text + chat_id)  
    - GetAttachedFile → ExtractFileText → FileOutput (format text + chat_id)  
    - GetAttachedPhoto → ExtractImageText → ImageOutput (format text + chat_id)

14. **Add TextOutput Set Node:**  
    - For plain text messages, set text and chat_id.

15. **Add AgentInput Set Node:**  
    - Receives formatted text and chat_id from media or text outputs.

16. **Add Langchain Agent Node: AIAgent:**  
    - Set model to GPT-4o (OpenRouter credentials).  
    - System prompt: relay all expense messages to ExpenseAssistant without modification.  
    - Use SimpleMemory buffer with context window 30.

17. **Add Langchain Agent Tool Node: ExpenseAssistant:**  
    - System prompt defines expense extraction, validation, storage rules, categories.  
    - Connect JsonStorageMcp (for Ainoflow JSON storage), Calculator, and AssitantMemory nodes.  
    - Use Gpt4o and Sonnet45 LM nodes for AI calls.

18. **Add JsonStorageMcp Tool Node:**  
    - Configure endpoint https://mcp.ainoflow.io/mcp/v1/storage/json  
    - Use "AinoflowPrimary" Bearer credentials.

19. **Add Calculator Tool Node:**  
    - No special config, used for sums and statistics.

20. **Add MemoryBufferWindow Nodes:**  
    - SimpleMemory for AIAgent with sessionKey `agent_{{chat_id}}`  
    - AssitantMemory for ExpenseAssistant with sessionKey `expense_assitant_{{chat_id}}`

21. **Add Telegram ReplyText Node:**  
    - Sends AI assistant output to user in Markdown format (single asterisk for bold).  
    - Uses chat_id from AgentInput.

22. **Add SendProcessing Telegram Node:**  
    - Sends "..." message at processing start.

23. **Add SaveExecutionState HTTP Request:**  
    - PUT to https://api.ainoflow.io/api/v1/storage/json/expense-app-processing/{{update_id}}  
    - Stores chat_id and message_id of processing message.  
    - Expires in 3600000 ms (1 hour).

24. **Add GetExecutionState HTTP Request:**  
    - GET stored processing message info for update_id.

25. **Add IfStateExist Node:**  
    - Checks if chat_id and message_id exist in execution state.

26. **Add DeleteProcessing Telegram Node:**  
    - Deletes processing message from chat.

27. **Add Manual Trigger Node: Cleanup:**  
    - For manual clearing all data.

28. **Add ForEachCategory HTTP Request:**  
    - Lists all categories in Ainoflow storage.

29. **Add ForEachCategoryItem HTTP Request:**  
    - Lists all items in a category.

30. **Add DeleteItem HTTP Request:**  
    - Deletes specific expense by category and key.

31. **Link all nodes according to workflow logic and connection map.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Personal Expense Tracker bot supports multiple input formats: text, voice, photo receipts, PDF invoices.                                                                                                                                           | Bot capability overview                                                                         |
| Setup guide for Telegram bot creation: https://blog.n8n.io/create-telegram-bot/                                                                                                                                                                   | Telegram bot setup instructions                                                                |
| OpenRouter API key setup: https://docs.n8n.io/integrations/builtin/credentials/openrouter/                                                                                                                                                        | OpenRouter credential guidance                                                                 |
| Ainoflow.io account needed for storage and conversion APIs: https://www.ainoflow.io/signup                                                                                                                                                        | Ainoflow API platform for storage, OCR, transcription                                          |
| Expense categories are fixed and must be used exactly as specified to ensure data consistency.                                                                                                                                                     | Expense assistant system prompt                                                                |
| Bot locks to first user to ensure privacy and security. Unauthorized users receive a denial message.                                                                                                                                                | Security feature                                                                               |
| ExpenseAssistant AI adds expenses immediately without confirmation to streamline user experience.                                                                                                                                                  | Business logic rule                                                                            |
| All messages starting with "User uploaded document / photo:" are treated as expense data from OCR or extracted text.                                                                                                                                | Input processing rule                                                                         |
| Use single asterisk (*) for Markdown bold formatting in Telegram responses, not double asterisk or underscores.                                                                                                                                     | Telegram formatting guideline                                                                 |
| Manual Cleanup deletes **all** stored data, use with caution.                                                                                                                                                                                       | Data management warning                                                                       |
| For customization or support, contact developer at https://ainovasystems.com/                                                                                                                                                                       | Project credits and contact                                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---