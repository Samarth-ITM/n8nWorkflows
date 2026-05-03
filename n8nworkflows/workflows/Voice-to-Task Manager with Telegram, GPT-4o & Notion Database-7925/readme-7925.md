Voice-to-Task Manager with Telegram, GPT-4o & Notion Database

https://n8nworkflows.xyz/workflows/voice-to-task-manager-with-telegram--gpt-4o---notion-database-7925


# Voice-to-Task Manager with Telegram, GPT-4o & Notion Database

### 1. Workflow Overview

This workflow, titled **"Voice-to-Task Manager with Telegram, GPT-4o & Notion Database"**, is designed to allow users to create, update, or analyze tasks in a Notion database via Telegram messages, including both voice and text inputs. Using AI (OpenAI GPT-4o-mini) for natural language understanding and task processing, it automates task management by transforming user instructions into structured task entries or insights.

**Target Use Cases:**
- Receiving task-related instructions via Telegram (voice or text)
- Transcribing voice messages to text automatically
- Detecting user intent: create, update, or analyze tasks
- Extracting structured task data or updates from natural language
- Managing tasks in a Notion database (create new pages, update existing ones)
- Providing analytical feedback on current task lists
- Sending confirmation messages back to the user via Telegram

**Logical Blocks:**

1.1 **Input Reception and Preprocessing**  
Receives Telegram messages (voice or text), fetches and transcribes voice messages if needed, and prepares a unified text input for AI processing.

1.2 **Intent Detection**  
Uses GPT-4o to classify the user’s intent into one of three categories: create, update, or analyze.

1.3 **Intent Routing and Processing**  
Routes the flow based on intent and executes relevant sub-processes:  
- Create: Extract task fields and create a new Notion database page  
- Update: Identify the matching task, generate an updated task JSON, and update the Notion page  
- Analyze: Retrieve current tasks and generate analytical insights

1.4 **Database Integration and Confirmation**  
Interacts with Notion API to create or update tasks and sends confirmation messages back to the user on Telegram.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Preprocessing

**Overview:**  
Handles incoming Telegram messages, distinguishes between voice and text inputs, fetches voice files if necessary, transcribes voice to text, and prepares a unified text output for downstream AI processing.

**Nodes Involved:**  
- Receive Telegram Messages  
- Voice or Text? (Switch)  
- Fetch Voice Message  
- Transcribe Voice to Text  
- Prepare for LLM  
- Prepare Unified Text  

**Node Details:**  

- **Receive Telegram Messages**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (voice or text)  
  - Config: Listens to "message" updates using Telegram API credentials  
  - Input: External Telegram webhook  
  - Output: Raw Telegram message JSON  
  - Failures: Telegram auth errors, webhook downtime  

- **Voice or Text?**  
  - Type: Switch  
  - Role: Routes input based on message content type (voice, text, or error)  
  - Config: Checks existence of `message.voice.file_id` for audio, `message.text` for text, or "error"  
  - Output connections:  
    - Audio → Fetch Voice Message  
    - Text → Prepare for LLM  
    - Error → (not connected)  
  - Edge cases: Messages without voice or text cause routing to error  

- **Fetch Voice Message**  
  - Type: Telegram node (file fetch)  
  - Role: Downloads voice file from Telegram server by file ID  
  - Config: Uses file_id from Telegram message voice field  
  - Output: Audio file for transcription  
  - Failures: File not found, Telegram API errors  

- **Transcribe Voice to Text**  
  - Type: OpenAI audio resource (translate)  
  - Role: Converts voice audio file to text transcription using OpenAI Whisper model  
  - Config: Audio translate operation, OpenAI API credentials  
  - Output: Transcribed text JSON  
  - Failures: API rate limits, audio format issues  

- **Prepare for LLM**  
  - Type: Set node  
  - Role: Assigns text from Telegram message directly for text inputs  
  - Config: Sets JSON property `text` to `message.text`  
  - Output: Unified text to feed AI models  

- **Prepare Unified Text**  
  - Type: Set node  
  - Role: Ensures unified key `text` for downstream processing, from either transcription or direct text  
  - Config: Sets `text` field from the input JSON  
  - Output: Standardized JSON with `text` field  

---

#### 1.2 Intent Detection

**Overview:**  
Analyzes the unified user message text to classify the intent as one of: create, update, or analyze.

**Nodes Involved:**  
- Detect Intent  
- Wrap Intent  

**Node Details:**  

- **Detect Intent**  
  - Type: LangChain LLM Chain (OpenAI GPT-4o-mini)  
  - Role: Classifies user's natural language message into task-related intent  
  - Config: Prompt instructs model to respond with a single word: create, update, or analyze  
  - Input: `text` from Prepare Unified Text  
  - Output: Plain text intent response  
  - Failures: Model errors, ambiguous input leading to incorrect classification  

- **Wrap Intent**  
  - Type: Set node  
  - Role: Wraps detected intent into a JSON object `{ intent: <intent> }` for routing  
  - Config: Outputs JSON with key `intent` equal to detected text  
  - Output: `{ intent: "create" | "update" | "analyze" }`  

---

#### 1.3 Intent Routing and Processing

**Overview:**  
Routes the workflow based on detected intent and processes each accordingly: creating new tasks, updating existing tasks, or analyzing task list insights.

**Nodes Involved:**  
- Wrap Intent + Text  
- Route by Intent (Switch)  

**Create Task Path:**  
- Create Task (Prompt)  
- Create Task Prompt (Model)  
- Extract Create Task Fields  
- Create a database page  
- Send Confirmation  

**Update Task Path:**  
- Get Search String  
- Get Search String Prompt (Model)  
- Get Notion Tasks (update)  
- Prepare Match Prompt  
- Find Matching Task  
- Find Matching Task Prompt (Model)  
- Prepare Update Prompt  
- Update Task Prompt  
- Update Task Prompt (Model)  
- to task object  
- Update a database page  
- Send Confirmation (update)  

**Analyze Task Path:**  
- Get Notion Tasks (analyze)  
- Prepare Analysis Prompt  
- AI Analyze Tasks  
- Alalyze Task Prompt (Model)  
- Send Analysis Result  

**Node Details:**  

- **Wrap Intent + Text**  
  - Type: Code  
  - Role: Combines detected intent and original text for routing  
  - Output: JSON with properties `intent` and `text`  

- **Route by Intent**  
  - Type: Switch  
  - Role: Routes flow to create, update, or analyze sub-branches based on `intent` property  
  - Output connections: create, update, analyze  

---

**Create Task Sub-Flow**

- **Create Task (Prompt)**  
  - Type: LangChain Chain LLM  
  - Role: Extracts structured task fields from user text for new task creation  
  - Config: Prompt instructs extraction of fields like title, priority, description, due date, tags  
  - Input: User text  
  - Output: JSON task object  
  - Failures: Parsing errors, incomplete data  

- **Create Task Prompt (Model)**  
  - Type: OpenAI Chat Model (gpt-4o-mini)  
  - Role: Executes the LLM prompt for task creation extraction  
  - Input: User text  
  - Output: JSON structured task data  

- **Extract Create Task Fields**  
  - Type: Output Parser Structured  
  - Role: Parses the JSON output from LLM prompt to ensure valid task fields  
  - Input: Raw LLM output  
  - Output: Validated JSON task object  

- **Create a database page**  
  - Type: Notion node  
  - Role: Creates new page in the Notion database with extracted task properties  
  - Config: Maps extracted fields to Notion properties (title, description, due date, priority, tags)  
  - Failures: Notion API errors, invalid data formats  

- **Send Confirmation**  
  - Type: Telegram node  
  - Role: Sends confirmation message with task link back to Telegram user  
  - Config: Uses chat ID from original message to send reply  
  - Failures: Telegram errors, message formatting issues  

---

**Update Task Sub-Flow**

- **Get Search String**  
  - Type: LangChain Chain LLM  
  - Role: Extracts a fuzzy search string from user text to identify which task to update  
  - Input: User text  
  - Output: Search string (plain text)  

- **Get Search String Prompt (Model)**  
  - Type: OpenAI Chat Model  
  - Role: Executes LLM prompt to identify search string from input text  

- **Get Notion Tasks (update)**  
  - Type: Notion node  
  - Role: Retrieves all existing tasks from Notion for matching  
  - Output: List of tasks JSON  

- **Prepare Match Prompt**  
  - Type: Code  
  - Role: Packages search string and tasks list into JSON for AI matching  

- **Find Matching Task**  
  - Type: LangChain Chain LLM  
  - Role: Finds the most relevant existing task matching the search string  
  - Output: Single matched task JSON  

- **Find Matching Task Prompt (Model)**  
  - Type: OpenAI Chat Model  
  - Role: Executes LLM prompt that returns the best matching task JSON object  

- **Prepare Update Prompt**  
  - Type: Code  
  - Role: Combines user message and matched task for update prompt  

- **Update Task Prompt**  
  - Type: LangChain Chain LLM  
  - Role: Generates updated task JSON based on user instructions and original task  
  - Config: Prompt to apply changes, format lists, keep unchanged fields  

- **Update Task Prompt (Model)**  
  - Type: OpenAI Chat Model  
  - Role: Runs the update task LLM prompt  

- **to task object**  
  - Type: Code  
  - Role: Parses LLM output string into JSON object for Notion update  

- **Update a database page**  
  - Type: HTTP Request node  
  - Role: PATCH request to Notion API to update existing page properties with new data  
  - Config: Maps updated task fields to Notion properties  
  - Failures: API auth errors, invalid data, network issues  

- **Send Confirmation (update)**  
  - Type: Telegram node  
  - Role: Sends confirmation message with updated task link to user  

---

**Analyze Task Sub-Flow**

- **Get Notion Tasks (analyze)**  
  - Type: Notion node  
  - Role: Retrieves all current tasks from Notion database for analysis  

- **Prepare Analysis Prompt**  
  - Type: Code  
  - Role: Prepares JSON containing user question and tasks list for AI analysis  

- **AI Analyze Tasks**  
  - Type: LangChain Chain LLM  
  - Role: Produces actionable insights or advice based on user’s natural language question and current tasks  
  - Output: Plain text analysis  

- **Alalyze Task Prompt (Model)**  
  - Type: OpenAI Chat Model  
  - Role: Runs the analysis LLM prompt  

- **Send Analysis Result**  
  - Type: Telegram node  
  - Role: Sends the analysis text back to the user via Telegram  

---

#### 1.4 Database Integration and Confirmation

This block is integrated within the create and update sub-flows, handling Notion API interactions and sending Telegram confirmations.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                          | Input Node(s)                         | Output Node(s)                             | Sticky Note                              |
|----------------------------|--------------------------------|----------------------------------------|-------------------------------------|-------------------------------------------|------------------------------------------|
| Receive Telegram Messages   | Telegram Trigger               | Receives Telegram messages              | -                                   | Voice or Text?                            | **Get Telegram Message**                  |
| Voice or Text?             | Switch                        | Routes by message type (voice/text)    | Receive Telegram Messages           | Fetch Voice Message / Prepare for LLM     | **Get Telegram Message**                  |
| Fetch Voice Message         | Telegram                      | Downloads voice file                    | Voice or Text? (Audio)              | Transcribe Voice to Text                   | **Get Telegram Message**                  |
| Transcribe Voice to Text    | OpenAI Audio (Translate)      | Transcribes voice audio to text         | Fetch Voice Message                 | Prepare Unified Text                       | **Get Telegram Message**                  |
| Prepare for LLM             | Set                          | Prepares text field for LLM input       | Voice or Text? (Text)               | Prepare Unified Text                       | **Get Telegram Message**                  |
| Prepare Unified Text        | Set                          | Normalizes text input                    | Prepare for LLM / Transcribe Voice | Detect Intent                             |                                          |
| Detect Intent               | LangChain LLM Chain           | Classifies user intent                   | Prepare Unified Text                | Wrap Intent                               | **Add Intent (create, update, analyze)** |
| Wrap Intent                | Set                          | Wraps intent into JSON                   | Detect Intent                      | Wrap Intent + Text                         | **Add Intent (create, update, analyze)** |
| Wrap Intent + Text          | Code                         | Combines intent and text                 | Wrap Intent, Prepare Unified Text  | Route by Intent                           | **Add Intent (create, update, analyze)** |
| Route by Intent             | Switch                       | Routes flow by intent                    | Wrap Intent + Text                  | Create Task (Prompt) / Get Search String / Get Notion Tasks (analyze) | **Add Intent (create, update, analyze)** |
| Create Task (Prompt)        | LangChain Chain LLM           | Extracts structured task for creation   | Route by Intent (create)            | Create a database page                    | **Create Task**                           |
| Create Task Prompt (Model)  | OpenAI Chat Model             | Executes LLM prompt for create task     | Create Task (Prompt)                | Extract Create Task Fields                | **Create Task**                           |
| Extract Create Task Fields  | Output Parser Structured     | Parses JSON output of create task       | Create Task Prompt (Model)          | Create a database page                    | **Create Task**                           |
| Create a database page      | Notion                       | Creates Notion task page                 | Extract Create Task Fields          | Send Confirmation                        | **Create Task**                           |
| Send Confirmation          | Telegram                     | Sends creation confirmation message     | Create a database page              | -                                         | **Create Task**                           |
| Get Search String           | LangChain Chain LLM           | Extracts search string for update        | Route by Intent (update)            | Get Notion Tasks (update)                 | **Update Task**                           |
| Get Search String Prompt (Model) | OpenAI Chat Model         | Runs LLM prompt for search string       | Get Search String                  | Get Notion Tasks (update)                 | **Update Task**                           |
| Get Notion Tasks (update)   | Notion                       | Retrieves current tasks for update       | Get Search String                  | Prepare Match Prompt                      | **Update Task**                           |
| Prepare Match Prompt        | Code                         | Combines search string and tasks list   | Get Notion Tasks (update), Get Search String | Find Matching Task                  | **Update Task**                           |
| Find Matching Task          | LangChain Chain LLM           | Finds best matching task JSON            | Prepare Match Prompt               | Prepare Update Prompt                     | **Update Task**                           |
| Find Matching Task Prompt (Model) | OpenAI Chat Model         | Runs LLM prompt to find matching task   | Find Matching Task                | Prepare Update Prompt                     | **Update Task**                           |
| Prepare Update Prompt       | Code                         | Combines user message and matched task  | Find Matching Task, Wrap Intent + Text | Update Task Prompt                  | **Update Task**                           |
| Update Task Prompt          | LangChain Chain LLM           | Generates updated task JSON               | Prepare Update Prompt              | to task object                           | **Update Task**                           |
| Update Task Prompt (Model)  | OpenAI Chat Model             | Runs LLM prompt for task update           | Update Task Prompt                | to task object                           | **Update Task**                           |
| to task object              | Code                         | Parses updated task JSON string           | Update Task Prompt (Model)         | Update a database page                    | **Update Task**                           |
| Update a database page      | HTTP Request                 | Updates existing Notion task page         | to task object                    | Send Confirmation (update)                | **Update Task**                           |
| Send Confirmation (update) | Telegram                     | Sends update confirmation message         | Update a database page             | -                                         | **Update Task**                           |
| Get Notion Tasks (analyze)  | Notion                       | Retrieves tasks for analysis               | Route by Intent (analyze)          | Prepare Analysis Prompt                   | **Analyze Tasks**                         |
| Prepare Analysis Prompt     | Code                         | Prepares question and tasks JSON for AI   | Get Notion Tasks (analyze)         | AI Analyze Tasks                         | **Analyze Tasks**                         |
| AI Analyze Tasks            | LangChain Chain LLM           | Provides insights and advice on tasks      | Prepare Analysis Prompt            | Send Analysis Result                     | **Analyze Tasks**                         |
| Alalyze Task Prompt (Model) | OpenAI Chat Model             | Runs LLM prompt to analyze tasks           | AI Analyze Tasks                  | Send Analysis Result                     | **Analyze Tasks**                         |
| Send Analysis Result        | Telegram                     | Sends task analysis result to user          | AI Analyze Tasks                  | -                                         | **Analyze Tasks**                         |
| Sticky Notes                | StickyNote                   | Visual notes for workflow sections         | -                               | -                                         | Various content as noted above            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node "Receive Telegram Messages"**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates  
   - Credentials: Link to Telegram API OAuth2 credential  
   - Position: [48,144]

2. **Add Switch Node "Voice or Text?"**  
   - Conditions:  
     - Audio: Exists `$json.message.voice.file_id`  
     - Text: Exists `$json.message.text`  
     - Error: Exists `"error"` string (no connection)  
   - Position: [240,128]  
   - Connect output from "Receive Telegram Messages"

3. **Add Telegram Node "Fetch Voice Message"**  
   - Type: Telegram  
   - Resource: File  
   - FileId: Expression `$json.message.voice.file_id`  
   - Credentials: Telegram API  
   - Position: [560,112]  
   - Connect from "Voice or Text?" (Audio output)

4. **Add OpenAI Audio Node "Transcribe Voice to Text"**  
   - Resource: audio  
   - Operation: translate  
   - Credentials: OpenAI API  
   - Position: [768,112]  
   - Connect from "Fetch Voice Message"

5. **Add Set Node "Prepare for LLM"**  
   - Assign `text` = `$json.message.text`  
   - Position: [560,288]  
   - Connect from "Voice or Text?" (Text output)

6. **Add Set Node "Prepare Unified Text"**  
   - Assign `text` = `$json.text` or transcription text  
   - Position: [1040,192]  
   - Connect from both "Prepare for LLM" and "Transcribe Voice to Text"

7. **Add LangChain OpenAI Chat Node "Detect Intent"**  
   - Model: gpt-4o-mini  
   - Prompt: Classification of intent (create, update, analyze)  
   - Input: `text` from "Prepare Unified Text"  
   - Position: [1312,192]  
   - Connect from "Prepare Unified Text"

8. **Add Set Node "Wrap Intent"**  
   - Output JSON: `{ "intent": "{{ $json.text }}" }`  
   - Position: [1584,192]  
   - Connect from "Detect Intent"

9. **Add Code Node "Wrap Intent + Text"**  
   - Output JSON: `{ intent: <intent>, text: <text> }`  
   - Position: [1744,192]  
   - Connect from "Wrap Intent" and "Prepare Unified Text"

10. **Add Switch Node "Route by Intent"**  
    - Conditions based on `intent`: create, update, analyze  
    - Position: [1936,176]  
    - Connect from "Wrap Intent + Text"

---

**Create Task Branch:**

11. **Add LangChain Chain LLM "Create Task (Prompt)"**  
    - Prompt to extract structured task fields from user text  
    - Input: User text from "Route by Intent" (create output)  
    - Position: [2432,-144]  
    - Connect from "Route by Intent" (create)

12. **Add OpenAI Chat Node "Create Task Prompt (Model)"**  
    - Model: gpt-4o-mini  
    - Connect from "Create Task (Prompt)"

13. **Add Output Parser Structured "Extract Create Task Fields"**  
    - JSON schema for task fields (title, priority, description, due_date, tags)  
    - Connect from "Create Task Prompt (Model)"

14. **Add Notion Node "Create a database page"**  
    - DatabaseId: your Notion task database  
    - Map extracted fields to Notion properties (Name, Description, Due Date, Priority, Tags)  
    - Credentials: Notion API  
    - Connect from "Extract Create Task Fields"

15. **Add Telegram Node "Send Confirmation"**  
    - Text: "Task Link :{{ $json.url }}"  
    - ChatId: from original Telegram message  
    - Connect from "Create a database page"

---

**Update Task Branch:**

16. **Add LangChain Chain LLM "Get Search String"**  
    - Prompt to extract fuzzy search string for task matching  
    - Input: User text from "Route by Intent" (update)  
    - Position: [2400,352]  
    - Connect from "Route by Intent" (update)

17. **Add OpenAI Chat Node "Get Search String Prompt (Model)"**  
    - Model: gpt-4o-mini  
    - Connect from "Get Search String"

18. **Add Notion Node "Get Notion Tasks (update)"**  
    - Operation: getAll tasks from database  
    - Credentials: Notion API  
    - Connect from "Get Search String"

19. **Add Code Node "Prepare Match Prompt"**  
    - Combine search string and tasks list into JSON  
    - Connect from "Get Notion Tasks (update)" and "Get Search String"

20. **Add LangChain Chain LLM "Find Matching Task"**  
    - Prompt to select best matching task JSON object  
    - Connect from "Prepare Match Prompt"

21. **Add OpenAI Chat Node "Find Matching Task Prompt (Model)"**  
    - Model: gpt-4o-mini  
    - Connect from "Find Matching Task"

22. **Add Code Node "Prepare Update Prompt"**  
    - Combine user text and matched task into input JSON  
    - Connect from "Find Matching Task" and "Wrap Intent + Text"

23. **Add LangChain Chain LLM "Update Task Prompt"**  
    - Prompt to generate updated task JSON  
    - Connect from "Prepare Update Prompt"

24. **Add OpenAI Chat Node "Update Task Prompt (Model)"**  
    - Model: gpt-4o-mini  
    - Connect from "Update Task Prompt"

25. **Add Code Node "to task object"**  
    - Parses updated task JSON string to object  
    - Connect from "Update Task Prompt (Model)"

26. **Add HTTP Request Node "Update a database page"**  
    - Method: PATCH  
    - URL: `https://api.notion.com/v1/pages/{{ $json["id"] }}`  
    - Body: Map updated fields to Notion page properties  
    - Credentials: Notion API  
    - Connect from "to task object"

27. **Add Telegram Node "Send Confirmation (update)"**  
    - Text: "Task Link :{{ $json.url }}"  
    - ChatId: fixed or from message  
    - Connect from "Update a database page"

---

**Analyze Task Branch:**

28. **Add Notion Node "Get Notion Tasks (analyze)"**  
    - Operation: getAll tasks  
    - Credentials: Notion API  
    - Connect from "Route by Intent" (analyze)

29. **Add Code Node "Prepare Analysis Prompt"**  
    - Combines user question and current tasks into JSON  
    - Connect from "Get Notion Tasks (analyze)" and "Wrap Intent + Text"

30. **Add LangChain Chain LLM "AI Analyze Tasks"**  
    - Prompt to analyze task list and provide advice  
    - Connect from "Prepare Analysis Prompt"

31. **Add OpenAI Chat Node "Alalyze Task Prompt (Model)"**  
    - Model: gpt-4o-mini  
    - Connect from "AI Analyze Tasks"

32. **Add Telegram Node "Send Analysis Result"**  
    - Text: AI analysis result  
    - ChatId: user chat ID  
    - Connect from "AI Analyze Tasks"

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                         |
|------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow uses the GPT-4o-mini OpenAI model for all LLM prompts.          | OpenAI API key required with access to GPT-4o-mini model                                               |
| Telegram API credentials use OAuth2 and require webhook setup.               | Telegram Bot API docs: https://core.telegram.org/bots/api                                              |
| Notion API integration requires database ID of the Tasks database.           | Notion API docs: https://developers.notion.com/reference                                                  |
| Voice messages are transcribed using OpenAI Whisper via the audio translate operation. | Requires OpenAI audio transcription quota                                                              |
| Intent detection prompt is designed to return exactly one of: create, update, analyze. | Critical for routing flow correctly                                                                     |
| Task creation and update prompts enforce strict JSON output for automation.   | Avoids parsing errors downstream                                                                        |
| Update flow includes fuzzy matching to identify tasks from user input.       | May fail if tasks are ambiguous or insufficiently described                                             |
| Analysis flow provides plain-text, user-friendly advice on task load and priorities. | Useful for productivity insights                                                                        |
| Sticky notes in workflow visually mark major blocks: input reception, intent detection, create/update/analyze flows. | Helpful during manual workflow inspection                                                              |

---

**Disclaimer:**  
This documentation is generated based on a complete n8n workflow JSON for the "Voice-to-Task Manager with Telegram, GPT-4o & Notion Database". All nodes and connections have been analyzed and described to allow full reproduction and modification. The workflow complies with legal and ethical content policies.