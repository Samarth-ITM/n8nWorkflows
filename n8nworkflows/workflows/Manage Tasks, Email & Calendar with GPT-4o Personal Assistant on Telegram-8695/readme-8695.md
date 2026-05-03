Manage Tasks, Email & Calendar with GPT-4o Personal Assistant on Telegram

https://n8nworkflows.xyz/workflows/manage-tasks--email---calendar-with-gpt-4o-personal-assistant-on-telegram-8695


# Manage Tasks, Email & Calendar with GPT-4o Personal Assistant on Telegram

### 1. Workflow Overview

This workflow implements a comprehensive AI Personal Assistant that manages tasks, emails, and calendar events through Telegram, leveraging GPT-4o for natural language understanding and generation. It supports both text and voice inputs from users, processes requests via AI models, and interacts with Google Sheets, Gmail, Google Calendar, and Telegram APIs to perform task management, email handling, and calendar operations. Additionally, it includes a scheduled reminder system for due tasks with notifications sent via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Preprocessing:** Handles incoming Telegram messages, distinguishing text from voice input, and processes voice messages into transcribed text.

- **1.2 AI Processing Core:** Uses GPT-4o-based language models and LangChain agents to interpret user requests and generate appropriate responses or actions.

- **1.3 Task Management:** Manages task data stored in Google Sheets, including reading, adding, updating tasks, and filtering for reminders.

- **1.4 Email & Calendar Management:** Reads and sends Gmail messages; creates, reads, and deletes Google Calendar events based on user commands.

- **1.5 Response Delivery:** Sends the AI’s replies back to users on Telegram.

- **1.6 Reminder System:** Periodically checks for due tasks and sends task reminders via Telegram, marking tasks as reminded.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preprocessing

- **Overview:**  
Captures Telegram messages, distinguishes between text and voice inputs, extracts or downloads message content, and prepares unified text input for AI processing.

- **Nodes Involved:**  
  - Telegram Bot Trigger  
  - Store User Chat ID (Code)  
  - Route Text or Voice Messages (Switch)  
  - Extract Text Message (Set)  
  - Get Voice File from Telegram (Telegram)  
  - Download Voice File (HTTP Request)  
  - Transcribe Voice to Text (OpenAI LangChain)  
  - Merge Text and Voice Input (Set)  
  - Sticky Note: Voice & Text Processing

- **Node Details:**

  - **Telegram Bot Trigger**  
    - Type: Trigger node (Telegram)  
    - Role: Entry point for all incoming Telegram updates (messages)  
    - Configuration: Configured to listen for messages from users  
    - Inputs: None (trigger)  
    - Outputs: Passes message data downstream  
    - Failures: Possible Telegram API downtime or invalid webhook setup

  - **Store User Chat ID (Code)**  
    - Type: Code node (JavaScript)  
    - Role: Extracts and stores the user chat ID for response routing  
    - Configuration: Reads chat ID from incoming message JSON, saves in workflow data  
    - Inputs: Telegram Bot Trigger output  
    - Outputs: Passes enriched message data  
    - Edge Cases: Missing chat ID or malformed message data

  - **Route Text or Voice Messages (Switch)**  
    - Type: Switch node  
    - Role: Determines if the incoming message is text or voice  
    - Configuration: Condition checks message payload for text presence or voice message attachment  
    - Inputs: From Store User Chat ID  
    - Outputs: Two branches—text messages and voice messages  
    - Failures: Unrecognized input types or empty messages

  - **Extract Text Message (Set)**  
    - Type: Set node  
    - Role: Extracts text message content for processing  
    - Configuration: Sets variable with the text message content from input  
    - Inputs: Text message branch from Switch  
    - Outputs: Passes extracted text downstream  
    - Edge Cases: Empty or null text messages

  - **Get Voice File from Telegram (Telegram node)**  
    - Type: Telegram node  
    - Role: Retrieves voice file metadata from Telegram API  
    - Configuration: Uses voice message file ID from incoming message to request download URL  
    - Inputs: Voice message branch from Switch  
    - Outputs: Provides file metadata for download  
    - Failures: Invalid file ID or Telegram API errors

  - **Download Voice File (HTTP Request)**  
    - Type: HTTP Request node  
    - Role: Downloads actual voice file using URL from previous node  
    - Configuration: HTTP GET request to Telegram file URL with proper authentication  
    - Inputs: Output from Get Voice File node  
    - Outputs: Voice file binary data  
    - Failures: Network errors, authorization issues, file not found

  - **Transcribe Voice to Text (LangChain OpenAI node)**  
    - Type: OpenAI LangChain node  
    - Role: Transcribes voice audio into text using OpenAI’s speech-to-text capabilities  
    - Configuration: Uses OpenAI credentials, configured for transcription  
    - Inputs: Voice file binary data from Download Voice File  
    - Outputs: Transcribed text  
    - Failures: API rate limits, transcription errors, unsupported audio formats

  - **Merge Text and Voice Input (Set)**  
    - Type: Set node  
    - Role: Consolidates the extracted text message or transcribed voice text into a single text field for AI processing  
    - Configuration: Sets unified text variable based on whichever branch was active  
    - Inputs: From Extract Text Message or Transcribe Voice to Text  
    - Outputs: Unified text for AI  
    - Failures: Missing input from either branch

  - **Sticky Note: Voice & Text Processing**  
    - Role: Denotes the logical grouping of these nodes for input preprocessing

---

#### 1.2 AI Processing Core

- **Overview:**  
Processes unified user text input via GPT-4o powered LangChain agents, maintains conversation memory, and prevents duplicate responses.

- **Nodes Involved:**  
  - AI Personal Assistant (LangChain Agent)  
  - OpenAI Language Model (LangChain LM)  
  - Conversation Memory (LangChain Memory Buffer)  
  - Prevent Duplicate Responses (Code)  
  - Sticky Note: AI Assistant Core

- **Node Details:**

  - **AI Personal Assistant (LangChain Agent)**  
    - Type: LangChain Agent node  
    - Role: Core AI logic handling natural language understanding and decision making  
    - Configuration: Linked with OpenAI LM and memory buffer, configured for task, email, calendar commands  
    - Inputs: Unified text from Merge Text and Voice Input; AI language model; conversation memory  
    - Outputs: AI-generated responses and commands for downstream nodes  
    - Failures: API errors, invalid prompt formatting, memory buffer overflow

  - **OpenAI Language Model (LangChain LM)**  
    - Type: LangChain LM Chat node  
    - Role: Provides GPT-4o text generation capabilities for the agent  
    - Configuration: Uses OpenAI API key with chat completion model  
    - Inputs: Receives prompts from AI Personal Assistant  
    - Outputs: Generated text replies or command instructions  
    - Failures: Rate limits, invalid credentials

  - **Conversation Memory (LangChain Memory Buffer Window)**  
    - Type: LangChain Memory Buffer  
    - Role: Maintains recent conversation context to provide continuity  
    - Configuration: Sliding window buffer, limited size for recent exchanges  
    - Inputs: Conversation history and new user inputs  
    - Outputs: Contextualized prompts for AI model  
    - Failures: Memory overflow, data corruption

  - **Prevent Duplicate Responses (Code)**  
    - Type: Code node  
    - Role: Checks if the AI response was already sent to avoid repetitive replies  
    - Configuration: Compares current response with stored previous responses  
    - Inputs: AI Personal Assistant output  
    - Outputs: Either blocks or allows message forwarding  
    - Edge Cases: False positives blocking legitimate responses

  - **Sticky Note: AI Assistant Core**  
    - Role: Marks the AI logic processing block

---

#### 1.3 Task Management

- **Overview:**  
Manages tasks stored in Google Sheets, including reading all tasks, adding new tasks, updating statuses, filtering due tasks, and tracking reminder statuses.

- **Nodes Involved:**  
  - Read Task Sheet (Google Sheets Tool)  
  - Add New Task (Google Sheets Tool)  
  - Update Task Status (Google Sheets Tool)  
  - Read Task Data (Google Sheets)  
  - Filter Due Tasks (Code)  
  - Format Reminder Message (Code)  
  - Send Task Reminder (Telegram)  
  - Mark Reminder Sent (Google Sheets)  
  - Sticky Notes: Task Management, Reminder System, Data Processing, Smart Filtering, Message Formatting, Reminder Delivery, Status Tracking

- **Node Details:**

  - **Read Task Sheet (Google Sheets Tool)**  
    - Type: Google Sheets Tool (Read)  
    - Role: Retrieves all task entries for AI processing or reminders  
    - Configuration: Reads from specific spreadsheet and sheet holding tasks  
    - Inputs: Triggered by AI assistant or scheduler  
    - Outputs: Task data array  
    - Failures: Authentication errors, sheet access denied

  - **Add New Task (Google Sheets Tool)**  
    - Type: Google Sheets Tool (Append)  
    - Role: Adds new task rows as requested by AI commands  
    - Configuration: Spreadsheet and sheet specified; columns mapped to task fields  
    - Inputs: AI assistant instructions  
    - Outputs: Confirmation of task addition  
    - Failures: Write permission issues

  - **Update Task Status (Google Sheets Tool)**  
    - Type: Google Sheets Tool (Update)  
    - Role: Updates status fields of tasks (e.g., completed, reminded)  
    - Configuration: Identifies rows by task ID or criteria; updates status column  
    - Inputs: AI assistant commands or reminder marking  
    - Outputs: Update confirmation  
    - Edge Cases: Concurrent edits, missing row IDs

  - **Read Task Data (Google Sheets)**  
    - Type: Google Sheets (Read)  
    - Role: Reads task data for the reminder system scheduler  
    - Configuration: Reads from task spreadsheet  
    - Inputs: 30-Minute Reminder Timer  
    - Outputs: Task list for filtering  
    - Failures: Same as Read Task Sheet

  - **Filter Due Tasks (Code)**  
    - Type: Code node  
    - Role: Filters tasks that are due and have not been reminded yet  
    - Configuration: Uses due date comparison, reminder status flags  
    - Inputs: Task data from Read Task Data  
    - Outputs: List of due tasks to remind  
    - Edge Cases: Date parsing errors, timezone mismatches

  - **Format Reminder Message (Code)**  
    - Type: Code node  
    - Role: Builds user-friendly reminder messages from task details  
    - Configuration: Formats task title, due date, and instructions into text  
    - Inputs: Filtered due tasks  
    - Outputs: Formatted reminder text  
    - Edge Cases: Missing task fields

  - **Send Task Reminder (Telegram)**  
    - Type: Telegram node  
    - Role: Sends reminder messages to users via Telegram  
    - Configuration: Uses stored chat IDs to send messages  
    - Inputs: Formatted reminder message  
    - Outputs: Message delivery confirmation  
    - Failures: Telegram API limits, invalid chat IDs

  - **Mark Reminder Sent (Google Sheets)**  
    - Type: Google Sheets Tool (Update)  
    - Role: Marks tasks as reminded to avoid duplicate notifications  
    - Configuration: Updates reminder sent status column  
    - Inputs: After successful reminder send  
    - Outputs: Update confirmation

  - **Sticky Notes:**  
    - Task Management: groups task data read/write nodes  
    - Reminder System: groups reminder logic and scheduling  
    - Data Processing, Smart Filtering, Message Formatting, Reminder Delivery, Status Tracking: denote sub-functions within reminder system

---

#### 1.4 Email & Calendar Management

- **Overview:**  
Reads unread emails, sends emails, and manages Google Calendar events (create, read, delete) as per AI assistant instructions.

- **Nodes Involved:**  
  - Read Unread Emails (Gmail Tool)  
  - Send Email (Gmail Tool)  
  - Create Calendar Event (Google Calendar Tool)  
  - Read Calendar Events (Google Calendar Tool)  
  - Delete Calendar Event (Google Calendar Tool)  
  - Sticky Note: Email & Calendar

- **Node Details:**

  - **Read Unread Emails (Gmail Tool)**  
    - Type: Gmail Tool (Read)  
    - Role: Fetches unread emails for processing or summarization  
    - Configuration: Uses Gmail OAuth2 credentials, queries unread emails  
    - Inputs: Triggered by AI assistant  
    - Outputs: Email message data  
    - Failures: OAuth token expiration, Gmail API limits

  - **Send Email (Gmail Tool)**  
    - Type: Gmail Tool (Send)  
    - Role: Sends emails composed by AI assistant  
    - Configuration: Configured with Gmail OAuth2 credentials, email fields mapped  
    - Inputs: AI assistant generated email content  
    - Outputs: Send confirmation  
    - Failures: Invalid recipient, quota limits

  - **Create Calendar Event (Google Calendar Tool)**  
    - Type: Google Calendar Tool (Create)  
    - Role: Adds events to user’s Google Calendar  
    - Configuration: Event details mapped (title, time, attendees)  
    - Inputs: AI assistant commands  
    - Outputs: Event creation confirmation  
    - Failures: Calendar access denied, invalid event data

  - **Read Calendar Events (Google Calendar Tool)**  
    - Type: Google Calendar Tool (Read)  
    - Role: Retrieves events for given date ranges or queries  
    - Configuration: Time window parameters  
    - Inputs: AI assistant queries  
    - Outputs: Event data  
    - Failures: API errors, permission issues

  - **Delete Calendar Event (Google Calendar Tool)**  
    - Type: Google Calendar Tool (Delete)  
    - Role: Removes calendar events as requested  
    - Configuration: Event ID required  
    - Inputs: AI assistant commands  
    - Outputs: Deletion confirmation  
    - Edge Cases: Event not found, insufficient permissions

  - **Sticky Note: Email & Calendar**  
    - Groups these nodes logically for email and calendar integration

---

#### 1.5 Response Delivery

- **Overview:**  
Handles sending AI-generated responses back to the Telegram user.

- **Nodes Involved:**  
  - Prevent Duplicate Responses (Code)  
  - Send Response to User (Telegram)  
  - Sticky Note: Response Delivery

- **Node Details:**

  - **Prevent Duplicate Responses (Code)**  
    - See details above in AI Processing Core.

  - **Send Response to User (Telegram)**  
    - Type: Telegram node  
    - Role: Sends textual response messages back to the user chat  
    - Configuration: Uses stored chat ID and text message from AI output  
    - Inputs: Filtered AI response (non-duplicate)  
    - Outputs: Confirmation of message sent  
    - Failures: Telegram API rate limits, invalid chat ID

  - **Sticky Note: Response Delivery**  
    - Marks nodes responsible for message output

---

#### 1.6 Reminder System

- **Overview:**  
Implements scheduled checks every 30 minutes to identify due tasks, format reminders, send notifications, and track reminder status.

- **Nodes Involved:**  
  - 30-Minute Reminder Timer (Schedule Trigger)  
  - Read Task Data (Google Sheets)  
  - Filter Due Tasks (Code)  
  - Format Reminder Message (Code)  
  - Send Task Reminder (Telegram)  
  - Mark Reminder Sent (Google Sheets)  
  - Sticky Notes: Reminder System and related sub-functional notes as above

- **Node Details:**

  - **30-Minute Reminder Timer (Schedule Trigger)**  
    - Type: Schedule Trigger node  
    - Role: Fires workflow execution every 30 minutes  
    - Configuration: Fixed interval schedule  
    - Inputs: None (trigger)  
    - Outputs: Triggers Read Task Data node

  - **Read Task Data to Mark Reminder Sent**  
    - See details above in Task Management

  - **Filter Due Tasks, Format Reminder Message, Send Task Reminder, Mark Reminder Sent**  
    - See details above in Task Management

  - **Sticky Notes:**  
    - Reminder System and subnotes demarcate this scheduled reminder functionality

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                      | Input Node(s)                    | Output Node(s)                   | Sticky Note            |
|----------------------------|------------------------------------|-----------------------------------|---------------------------------|---------------------------------|------------------------|
| Telegram Bot Trigger        | Telegram Trigger                   | Entry point for Telegram messages  | None                            | Store User Chat ID               |                        |
| Store User Chat ID          | Code                              | Extracts and stores chat ID        | Telegram Bot Trigger            | Route Text or Voice Messages     |                        |
| Route Text or Voice Messages| Switch                           | Routes text vs. voice messages     | Store User Chat ID              | Extract Text Message, Get Voice File from Telegram |                        |
| Extract Text Message        | Set                               | Extracts text from messages        | Route Text or Voice Messages (text branch) | Merge Text and Voice Input        | Voice & Text Processing |
| Get Voice File from Telegram| Telegram                          | Gets voice file metadata           | Route Text or Voice Messages (voice branch) | Download Voice File              | Voice & Text Processing |
| Download Voice File         | HTTP Request                      | Downloads voice file binary        | Get Voice File from Telegram    | Transcribe Voice to Text         | Voice & Text Processing |
| Transcribe Voice to Text    | LangChain OpenAI                  | Transcribes voice to text          | Download Voice File             | Merge Text and Voice Input       | Voice & Text Processing |
| Merge Text and Voice Input | Set                               | Combines text and transcribed text| Extract Text Message, Transcribe Voice to Text | AI Personal Assistant            | Voice & Text Processing |
| AI Personal Assistant       | LangChain Agent                  | Processes AI commands & responses  | Merge Text and Voice Input, Conversation Memory, OpenAI LM | Prevent Duplicate Responses      | AI Assistant Core       |
| OpenAI Language Model       | LangChain LM Chat                | Generates AI text responses        | AI Personal Assistant           | AI Personal Assistant            | AI Assistant Core       |
| Conversation Memory         | LangChain Memory Buffer          | Maintains conversation context     | User inputs, AI responses       | AI Personal Assistant            | AI Assistant Core       |
| Prevent Duplicate Responses | Code                             | Avoids repeated AI responses       | AI Personal Assistant           | Send Response to User            | AI Assistant Core, Response Delivery |
| Send Response to User       | Telegram                         | Sends AI reply to Telegram user    | Prevent Duplicate Responses     | None                            | Response Delivery       |
| Read Unread Emails          | Gmail Tool                      | Reads unread emails                | AI Personal Assistant           | AI Personal Assistant            | Email & Calendar        |
| Send Email                  | Gmail Tool                      | Sends emails                      | AI Personal Assistant           | AI Personal Assistant            | Email & Calendar        |
| Create Calendar Event       | Google Calendar Tool            | Creates calendar events            | AI Personal Assistant           | AI Personal Assistant            | Email & Calendar        |
| Read Calendar Events        | Google Calendar Tool            | Reads calendar events              | AI Personal Assistant           | AI Personal Assistant            | Email & Calendar        |
| Delete Calendar Event       | Google Calendar Tool            | Deletes calendar events            | AI Personal Assistant           | AI Personal Assistant            | Email & Calendar        |
| Read Task Sheet             | Google Sheets Tool              | Reads tasks from sheet             | AI Personal Assistant           | AI Personal Assistant            | Task Management         |
| Add New Task                | Google Sheets Tool              | Adds new task                     | AI Personal Assistant           | AI Personal Assistant            | Task Management         |
| Update Task Status          | Google Sheets Tool              | Updates task status                | AI Personal Assistant           | AI Personal Assistant            | Task Management         |
| Read Task Data              | Google Sheets                  | Reads tasks for reminders          | 30-Minute Reminder Timer        | Filter Due Tasks                 | Task Management, Reminder System |
| Filter Due Tasks            | Code                           | Filters tasks due for reminder     | Read Task Data                  | Format Reminder Message          | Reminder System, Smart Filtering |
| Format Reminder Message     | Code                           | Creates reminder text              | Filter Due Tasks                | Send Task Reminder               | Reminder System, Message Formatting |
| Send Task Reminder          | Telegram                       | Sends reminder message             | Format Reminder Message         | Mark Reminder Sent               | Reminder System, Reminder Delivery |
| Mark Reminder Sent          | Google Sheets                  | Marks tasks as reminded            | Send Task Reminder              | None                            | Reminder System, Status Tracking |
| 30-Minute Reminder Timer   | Schedule Trigger               | Triggers reminder workflow         | None                          | Read Task Data                  | Reminder System         |
| Setup Instructions          | Sticky Note                   | Workflow setup instructions        | None                          | None                          |                        |
| Template Overview          | Sticky Note                   | Workflow overview and notes        | None                          | None                          |                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot Trigger**  
   - Node Type: Telegram Trigger  
   - Setup: Connect to your Telegram bot via token, configure to trigger on new messages.

2. **Add Code Node "Store User Chat ID"**  
   - Extract chat ID from incoming Telegram message JSON (e.g., `{{$json["message"]["chat"]["id"]}}`)  
   - Store chat ID in workflow data for reply routing.

3. **Add Switch Node "Route Text or Voice Messages"**  
   - Condition 1: Check if message has text (`{{$json["message"]["text"]}}` exists)  
   - Condition 2: Check if message has voice attachment (`{{$json["message"]["voice"]}}` exists)

4. **For Text Branch:**  
   - Add Set Node "Extract Text Message"  
   - Extract text message content (`{{$json["message"]["text"]}}`) into a variable for AI input.

5. **For Voice Branch:**  
   - Add Telegram Node "Get Voice File from Telegram"  
     - Configure with file_id from voice message (`{{$json["message"]["voice"]["file_id"]}}`) to get download URL.  
   - Add HTTP Request Node "Download Voice File"  
     - GET request to Telegram file URL with Telegram bot token authentication.  
   - Add LangChain OpenAI Node "Transcribe Voice to Text"  
     - Configure with OpenAI credentials, set to transcribe audio input.  
   
6. **Add Set Node "Merge Text and Voice Input"**  
   - Combine the text from text or transcribed voice into a unified variable for AI processing.

7. **Add LangChain Memory Buffer Node "Conversation Memory"**  
   - Configure sliding window buffer with appropriate size to maintain conversation context.

8. **Add OpenAI Language Model Node "OpenAI Language Model"**  
   - Configure with OpenAI API key, select GPT-4o or equivalent chat completion model.

9. **Add LangChain Agent Node "AI Personal Assistant"**  
   - Connect inputs: unified user text, conversation memory, and OpenAI LM.  
   - Configure agent with prompts for task, email, calendar management commands.

10. **Add Code Node "Prevent Duplicate Responses"**  
    - Implement logic to compare current AI output with previous responses to avoid repetition.

11. **Add Telegram Node "Send Response to User"**  
    - Use stored chat ID and AI response text to send message back to user.

12. **Add Google Sheets Tool Nodes ("Read Task Sheet", "Add New Task", "Update Task Status")**  
    - Configure with Google Sheets credentials and target spreadsheet/sheet for task data.  
    - Map fields for reading, adding, and updating task entries.

13. **Add Gmail Tool Nodes ("Read Unread Emails", "Send Email")**  
    - Configure OAuth2 credentials for Gmail access.  
    - Set parameters for reading unread emails and sending email messages.

14. **Add Google Calendar Tool Nodes ("Create Calendar Event", "Read Calendar Events", "Delete Calendar Event")**  
    - Configure with Google Calendar credentials and target calendar.  
    - Map event details for create/read/delete operations.

15. **Add Schedule Trigger Node "30-Minute Reminder Timer"**  
    - Set to trigger every 30 minutes.

16. **Add Google Sheets Node "Read Task Data"**  
    - Reads task list for reminder processing on schedule trigger.

17. **Add Code Node "Filter Due Tasks"**  
    - Implement logic to identify due tasks not yet reminded.

18. **Add Code Node "Format Reminder Message"**  
    - Format task details into user-friendly reminder messages.

19. **Add Telegram Node "Send Task Reminder"**  
    - Sends reminder messages to users’ Telegram chat IDs.

20. **Add Google Sheets Node "Mark Reminder Sent"**  
    - Updates task entries to mark reminders as sent.

21. **Connect all nodes according to the data flow described above.**

22. **Add Sticky Notes for logical grouping and documentation if desired.**

23. **Test the workflow end-to-end with both text and voice inputs and verify task, email, calendar commands and reminders.**

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                           |
|---------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow uses GPT-4o model via LangChain integration for advanced AI capabilities.                   | OpenAI GPT-4o via LangChain nodes                         |
| Telegram Bot API is used both for receiving user inputs and sending responses and reminders.            | Telegram Bot API documentation: https://core.telegram.org/bots/api |
| Google Sheets, Gmail, and Google Calendar integrations require proper OAuth2 credentials setup.         | Google Cloud Console and API credentials setup            |
| The reminder system uses a scheduled trigger every 30 minutes to check for due tasks and notify users.  | n8n Schedule Trigger node documentation                   |
| Conversation memory is managed with a sliding window buffer to maintain context without exceeding limits.| LangChain Memory Buffer Window node                        |
| Prevent Duplicate Responses node avoids spamming users with repeated answers.                           | Custom Code node with logic to detect repeated messages   |
| Recommended to monitor API usage and handle rate limits for Telegram and OpenAI APIs.                   | API quotas and rate limits documentation                   |
| Workflow tags for easy search and categorization: AI, Personal Assistant, Task Management, Email, Calendar, Telegram, Automation |                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.