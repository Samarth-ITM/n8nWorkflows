Jarvis: Productivity AI Agent for Tasks, Calendar, Email & Expense using MCPs

https://n8nworkflows.xyz/workflows/jarvis--productivity-ai-agent-for-tasks--calendar--email---expense-using-mcps-8500


# Jarvis: Productivity AI Agent for Tasks, Calendar, Email & Expense using MCPs

### 1. Workflow Overview

This workflow, titled **Jarvis: Productivity AI Agent for Tasks, Calendar, Email & Expense using MCPs**, is designed as a comprehensive AI-driven personal productivity assistant named "Jarvis." It integrates multiple productivity domains including task management, calendar scheduling, email handling, contact management, and expense tracking. The workflow leverages n8n’s Langchain MCP (Model Context Protocol) tools, OpenAI GPT-4, Telegram as a user interface, and several Google services (Gmail, Calendar, Tasks, Contacts, Sheets) to provide a seamless and intelligent assistant experience.

**Target Use Cases:**  
- Manage daily tasks with Google Tasks: create, complete, delete, and retrieve tasks.  
- Handle calendar events on Google Calendar: check availability, create, update, delete, and fetch events.  
- Manage emails via Gmail: send, reply, draft, label, and retrieve emails.  
- Track expenses with Google Sheets integration: add, retrieve, and delete expense records.  
- Access and retrieve Google Contacts for communication.  
- Provide conversational AI assistance through Telegram, supporting both text and audio inputs with speech recognition and synthesis.

---

**Logical Blocks:**

- **1.1 Input Reception and User Authentication**  
  Receives user input from Telegram; filters authorized users; differentiates between text and audio input.

- **1.2 AI Processing and Memory Management**  
  Processes user input with Jarvis agent, backed by OpenAI GPT-4 and a simple memory buffer keyed by Telegram username.

- **1.3 Email Management (Gmail MCP)**  
  Full Gmail integration for sending, replying, drafting, labeling, and searching emails, via MCP protocol.

- **1.4 Calendar Management (Calendar MCP)**  
  Google Calendar operations: availability checks, event creation, retrieval, rescheduling, and deletion through MCP.

- **1.5 Task Management (Google Tasks MCP)**  
  Google Tasks operations: create, get, update (complete), delete tasks managed via MCP.

- **1.6 Finance Tracking (Finance Manager MCP)**  
  Expense tracking using Google Sheets to add, retrieve, and delete expense data.

- **1.7 Contact Management (Google Contacts MCP)**  
  Fetch and manage Google Contacts to facilitate communication.

- **1.8 Audio Processing (Speech to Text and Text to Speech)**  
  Handles audio input transcription and audio response generation via ElevenLabs API.

- **1.9 Output Delivery**  
  Sends back text or audio responses to Telegram users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and User Authentication

- **Overview:**  
  This block receives user messages from Telegram, checks if the sender is authorized, and routes the input based on whether it's text or audio.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Only allow me (Filter)  
  - Switch (Check Text or Audio)  
  - Switch (inside Only allow me for text/audio existence check)  
  - Get a file (for audio inputs)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Entry point capturing all incoming Telegram messages (text/audio).  
    - Configuration: Listens for "message" updates; uses Telegram Bot credentials.  
    - Outputs: Raw Telegram message JSON.  
    - Failure cases: Telegram API downtime, invalid webhook.

  - **Only allow me (Filter)**  
    - Type: Filter  
    - Role: Authorizes only user with username "jackman8" to proceed.  
    - Configuration: Checks if `message.chat.username` equals "jackman8".  
    - Output: Routes authorized users to Switch node; blocks others.  
    - Edge cases: Unauthorized users get no response; consider adding rejection message.

  - **Switch (Check Text or Audio)**  
    - Type: Switch  
    - Role: Differentiates between text and audio inputs by checking if `message.text` exists.  
    - Configuration: Routes text messages to Jarvis AI, audio messages to "Get a file".  
    - Edge cases: Messages without text or audio might be dropped silently.

  - **Get a file**  
    - Type: Telegram Node  
    - Role: Retrieves the audio file from Telegram for transcription.  
    - Configuration: Uses `message.voice.file_id` from Telegram message.  
    - Output: Passes file data to transcription node.  
    - Failure: Missing or expired file IDs, Telegram file API errors.

---

#### 1.2 AI Processing and Memory Management

- **Overview:**  
  Uses a Langchain agent named Jarvis empowered by GPT-4, supported by a simple memory buffer that maintains session context per Telegram username.

- **Nodes Involved:**  
  - Simple Memory  
  - OpenAI Chat Model  
  - Jarvis (Langchain Agent)  
  - Think (Tool Think)  
  - Set Reply Message

- **Node Details:**

  - **Simple Memory**  
    - Type: Memory Buffer Window (Langchain)  
    - Role: Maintains conversation history keyed by Telegram username.  
    - Configuration: Session key as username from Telegram message; custom session ID type.  
    - Edge cases: Memory overflow or reset, user switching.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4.1-mini model for language generation.  
    - Configuration: Model set to GPT-4.1-mini; credentials for OpenAI API provided.  
    - Failures: API key quota exceeded, rate limits, network errors.

  - **Jarvis**  
    - Type: Langchain Agent  
    - Role: Main AI agent that interprets user requests, decides tool usage, and composes responses.  
    - Configuration: System prompt defines Jarvis’s identity, capabilities, operational rules, and tone.  
    - Uses memory and OpenAI Chat Model nodes.  
    - OnError: Continue regular output to avoid halting on errors.  
    - Edge cases: Incorrect or ambiguous inputs, missing parameters, API failures.

  - **Think**  
    - Type: Langchain Tool Think  
    - Role: Internal thinking step for the agent to process multi-tool decisions.  
    - Configuration: Default.  
    - Connected to Jarvis for enhanced reasoning.

  - **Set Reply Message**  
    - Type: Set  
    - Role: Prepares the final text message output from Jarvis’s response or error messages.  
    - Configuration: Assigns `message` field from Jarvis output or error.  
    - Outputs: Feeds into output delivery nodes.

---

#### 1.3 Email Management (Gmail MCP)

- **Overview:**  
  Full Gmail integration through MCP enables sending, replying, drafting, labeling, and fetching emails.

- **Nodes Involved:**  
  - Gmail MCP (MCP Client Tool)  
  - Gmail MCP Server (MCP Trigger)  
  - Send Email  
  - Reply to an Email  
  - Draft Email  
  - Draft Email Reply  
  - Get Emails  
  - Add Label to Email  
  - Get Labels

- **Node Details:**

  - **Gmail MCP Server**  
    - Type: MCP Trigger  
    - Role: Entry point for AI requests related to Gmail operations.  
    - Configuration: Webhook with a unique path.  
    - Edge cases: Webhook downtime, invalid requests.

  - **Gmail MCP**  
    - Type: MCP Client Tool  
    - Role: Client tool to communicate with Gmail MCP Server via SSE endpoint.  
    - Configuration: Endpoint URL specified.  
    - Used by Jarvis to perform email operations.

  - **Send Email**  
    - Type: Gmail Tool  
    - Role: Sends new emails.  
    - Configuration: Uses AI-provided To, Subject, Message fields; disables attribution.  
    - Credentials: Gmail OAuth2.  
    - Edge cases: Invalid email addresses, quota exhaustion.

  - **Reply to an Email**  
    - Type: Gmail Tool  
    - Role: Replies to existing emails with optional CC, BCC, attachments.  
    - Configuration: Uses AI variables for message content, message ID, recipients.  
    - Edge cases: Missing message ID, attachment errors.

  - **Draft Email**  
    - Type: Gmail Tool  
    - Role: Creates email drafts with HTML content and recipients.  
    - Configuration: Uses AI-provided Subject, Message, CC, BCC, To fields.  
    - Edge cases: Missing subject/message.

  - **Draft Email Reply**  
    - Type: Gmail Tool  
    - Role: Drafts replies linked to a thread ID.  
    - Configuration: Uses AI variables for message, thread ID, recipients.  
    - Edge cases: Invalid thread ID.

  - **Get Emails**  
    - Type: Gmail Tool  
    - Role: Retrieves emails based on AI-supplied filters (search query, date range).  
    - Configuration: Returns all or limited emails per AI input.  
    - Edge cases: Large data volume, malformed queries.

  - **Add Label to Email**  
    - Type: Gmail Tool  
    - Role: Adds labels to specific emails.  
    - Configuration: Uses message ID and comma-separated labels.  
    - Edge cases: Invalid label names/IDs.

  - **Get Labels**  
    - Type: Gmail Tool  
    - Role: Fetches list of labels in Gmail account.  
    - Configuration: Optionally returns all labels.  
    - Edge cases: API failures.

---

#### 1.4 Calendar Management (Calendar MCP)

- **Overview:**  
  Manages Google Calendar events: checks availability, creates, updates, reschedules, retrieves, and deletes events.

- **Nodes Involved:**  
  - Calendar MCP Server (MCP Trigger)  
  - Calendar MCP (MCP Client Tool)  
  - Check Availability  
  - Get all Events  
  - Create an event  
  - Get Event  
  - Reschedule Event  
  - Delete Calendar Event

- **Node Details:**

  - **Calendar MCP Server**  
    - Type: MCP Trigger  
    - Role: Receives AI requests for calendar operations.  
    - Configuration: Unique webhook path.  
    - Edge cases: Downtime, malformed requests.

  - **Calendar MCP**  
    - Type: MCP Client Tool  
    - Role: Sends AI requests to calendar MCP server.  
    - Configuration: SSE endpoint URL specified.

  - **Check Availability**  
    - Type: Google Calendar Tool  
    - Role: Checks if a time slot is free on the calendar.  
    - Configuration: Uses AI inputs for start/end times, calendar selection, timezone.  
    - Edge cases: Invalid dates, timezone mismatches.

  - **Get all Events**  
    - Type: Google Calendar Tool  
    - Role: Retrieves all calendar events within a time window.  
    - Configuration: Uses AI inputs for date filters; expands recurring events.  
    - Edge cases: Large event sets, API limits.

  - **Create an event**  
    - Type: Google Calendar Tool  
    - Role: Creates a new calendar event with optional summary and description.  
    - Configuration: Accepts start/end times, summary, description, reminders, calendar ID.  
    - Edge cases: Past dates, invalid attendees.

  - **Get Event**  
    - Type: Google Calendar Tool  
    - Role: Retrieves details of a single event by event ID.  
    - Edge cases: Nonexistent event ID.

  - **Reschedule Event**  
    - Type: Google Calendar Tool  
    - Role: Updates event timing and attendees.  
    - Configuration: Uses AI inputs for new start/end times and attendees.  
    - Edge cases: Conflicting schedules.

  - **Delete Calendar Event**  
    - Type: Google Calendar Tool  
    - Role: Deletes an event by ID.  
    - Edge cases: Invalid or already deleted events.

---

#### 1.5 Task Management (Google Tasks MCP)

- **Overview:**  
  Manages Google Tasks including creating, retrieving, updating (completing), and deleting tasks.

- **Nodes Involved:**  
  - Task Manager MCP (MCP Trigger)  
  - Google Tasks MCP (MCP Client Tool)  
  - Create a Task  
  - Get a Task  
  - Get many Tasks  
  - Complete a Task  
  - Delete a Task

- **Node Details:**

  - **Task Manager MCP**  
    - Type: MCP Trigger  
    - Role: Entry point for AI task management commands.  
    - Configuration: Webhook path.  
    - Edge cases: Request validation.

  - **Google Tasks MCP**  
    - Type: MCP Client Tool  
    - Role: Connects to task MCP server endpoint.  
    - Configuration: SSE endpoint URL.

  - **Create a Task**  
    - Type: Google Tasks Tool  
    - Role: Creates a new task with title, notes, and due date.  
    - Edge cases: Past due dates, invalid formats.

  - **Get a Task**  
    - Retrieves a specific task by ID.

  - **Get many Tasks**  
    - Retrieves multiple tasks with filtering options like showing completed tasks.

  - **Complete a Task**  
    - Marks a task as completed with a completion date.

  - **Delete a Task**  
    - Deletes a task by ID.

---

#### 1.6 Finance Tracking (Finance Manager MCP)

- **Overview:**  
  Tracks expenses using Google Sheets: adding new expenses, retrieving all expenses, and deleting expense entries.

- **Nodes Involved:**  
  - Finance Manager MCP Server (MCP Trigger)  
  - Finance Tracker (MCP Client Tool)  
  - Get all Expenses  
  - Create Expense  
  - Delete Expense

- **Node Details:**

  - **Finance Manager MCP Server**  
    - MCP Trigger receiving AI requests for finance operations.  
    - Includes a sticky note describing its capabilities and features.

  - **Finance Tracker**  
    - MCP Client Tool communicating with the Finance Manager MCP server via SSE.

  - **Get all Expenses**  
    - Reads rows from a specified Google Sheets document and sheet (Expense Tracker).  
    - AI-configurable parameters: Document ID, sheet name, range.

  - **Create Expense**  
    - Appends new expense entries to the sheet with columns Date, Amount, Category, Description.

  - **Delete Expense**  
    - Clears specified rows from the expense sheet.  
    - Edge cases: Data loss risk, invalid row numbers.

---

#### 1.7 Contact Management (Google Contacts MCP)

- **Overview:**  
  Provides access to Google Contacts to fetch contact details for communication purposes.

- **Nodes Involved:**  
  - Google Contacts MCP (MCP Trigger)  
  - Google Contacts (MCP Client Tool)  
  - Get Contacts

- **Node Details:**

  - **Google Contacts MCP**  
    - MCP Trigger for contact-related AI requests.

  - **Google Contacts**  
    - MCP Client Tool connected to contacts MCP server via SSE.

  - **Get Contacts**  
    - Retrieves contacts matching AI query parameters, with fields like names and email addresses.  
    - Edge cases: Permission restrictions, large contact lists.

---

#### 1.8 Audio Processing (Speech to Text and Text to Speech)

- **Overview:**  
  Handles conversion of audio messages from Telegram into text for AI processing, and converts AI text responses into audio files to send back.

- **Nodes Involved:**  
  - Transcribe audio or video (ElevenLabs speech-to-text)  
  - Convert text to speech (ElevenLabs text-to-speech)  
  - Send an audio file (Telegram)  
  - Send a text message (Telegram)

- **Node Details:**

  - **Transcribe audio or video**  
    - Converts Telegram audio file into transcribed text for Jarvis.  
    - Uses ElevenLabs API.  
    - Failures: Noisy audio, transcription errors.

  - **Convert text to speech**  
    - Synthesizes Jarvis’s response text into audio.  
    - Voice preset: "Devi - Clear Hindi pronunciation".  
    - Failures: API quota, text length limits.

  - **Send an audio file**  
    - Sends synthesized audio back to user's Telegram chat with caption from reply message.  
    - Ensures special characters in caption are escaped for MarkdownV2.

  - **Send a text message**  
    - Sends plain text reply messages via Telegram with MarkdownV2 formatting.

---

#### 1.9 Output Delivery

- **Overview:**  
  Depending on input type (text or audio), delivers Jarvis’s response back to the user via Telegram either as text or as an audio file.

- **Nodes Involved:**  
  - Send a text message  
  - Convert text to speech  
  - Send an audio file

- **Node Details:**

  - Text inputs → Send a text message directly.  
  - Audio inputs → Convert text to speech → Send audio file.  
  - Proper escaping in captions for MarkdownV2 compliance.

---

### 3. Summary Table

| Node Name              | Node Type                        | Functional Role                                 | Input Node(s)               | Output Node(s)                  | Sticky Note                                                   |
|------------------------|---------------------------------|------------------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger                | Entry point for Telegram messages               | -                           | Only allow me                 |                                                               |
| Only allow me          | Filter                         | Authorizes only user "jackman8"                  | Telegram Trigger            | Switch                       |                                                               |
| Switch (text/audio)    | Switch                         | Routes based on message text presence            | Only allow me               | Jarvis, Get a file            |                                                               |
| Get a file             | Telegram                       | Retrieves audio file from Telegram                | Switch                      | Transcribe audio or video     |                                                               |
| Transcribe audio or video | ElevenLabs Speech-to-Text      | Converts audio to text                            | Get a file                  | Jarvis                       |                                                               |
| Jarvis                 | Langchain Agent                | AI assistant core processing                      | Switch, Transcribe audio... | Set Reply Message             |                                                               |
| Simple Memory          | Langchain Memory Buffer        | Maintains session memory keyed by username       | Telegram Trigger            | Jarvis                       |                                                               |
| OpenAI Chat Model      | Langchain OpenAI Model         | GPT-4 language model                              | Jarvis                      | Jarvis                       |                                                               |
| Think                  | Langchain Tool Think           | Internal agent reasoning                          | Jarvis                      | Jarvis                       |                                                               |
| Set Reply Message      | Set                           | Prepares reply message for output                 | Jarvis                      | Check Text or Audio           |                                                               |
| Check Text or Audio    | Switch                        | Routes output to text or audio response           | Set Reply Message            | Send a text message, Convert text to speech |                                                               |
| Send a text message    | Telegram                      | Sends reply as text message                        | Check Text or Audio          | -                            |                                                               |
| Convert text to speech | ElevenLabs Text-to-Speech      | Converts text reply to audio                       | Check Text or Audio          | Send an audio file            |                                                               |
| Send an audio file     | Telegram                      | Sends audio reply to Telegram                      | Convert text to speech       | -                            |                                                               |
| Gmail MCP Server       | MCP Trigger                   | Entry for Gmail-based AI email operations         | Get Emails, Send Email, ... | -                            | Gmail MCP 📧 Full email management. Send & draft messages, reply, label, fetch emails |
| Gmail MCP              | MCP Client Tool               | Client for Gmail MCP Server                        | Jarvis                      | Jarvis                       | Gmail MCP 📧 Full email management. Send & draft messages, reply, label, fetch emails |
| Send Email             | Gmail Tool                   | Sends email                                       | Gmail MCP                   | Gmail MCP Server             |                                                               |
| Reply to an Email      | Gmail Tool                   | Replies to email                                  | Gmail MCP                   | Gmail MCP Server             |                                                               |
| Draft Email            | Gmail Tool                   | Creates email drafts                              | Gmail MCP                   | Gmail MCP Server             |                                                               |
| Draft Email Reply      | Gmail Tool                   | Drafts email replies                              | Gmail MCP                   | Gmail MCP Server             |                                                               |
| Get Emails             | Gmail Tool                   | Retrieves emails                                 | Gmail MCP                   | Gmail MCP Server             |                                                               |
| Add Label to Email     | Gmail Tool                   | Adds labels to emails                             | Gmail MCP                   | Gmail MCP Server             |                                                               |
| Get Labels             | Gmail Tool                   | Retrieves email labels                            | Gmail MCP                   | Gmail MCP Server             |                                                               |
| Calendar MCP Server    | MCP Trigger                   | Entry point for calendar AI operations            | Get Event, Create an event...| -                            | Calendar MCP 📅 Your scheduling hub. Check availability, create, reschedule, delete events |
| Calendar MCP           | MCP Client Tool               | Client to calendar MCP Server                      | Jarvis                      | Jarvis                       | Calendar MCP 📅 Your scheduling hub. Check availability, create, reschedule, delete events |
| Check Availability     | Google Calendar Tool          | Checks free/busy slots                            | Calendar MCP                | Calendar MCP Server          |                                                               |
| Get all Events         | Google Calendar Tool          | Retrieves calendar events                         | Calendar MCP                | Calendar MCP Server          |                                                               |
| Create an event        | Google Calendar Tool          | Creates new calendar event                        | Calendar MCP                | Calendar MCP Server          |                                                               |
| Get Event              | Google Calendar Tool          | Retrieves specific event                          | Calendar MCP                | Calendar MCP Server          |                                                               |
| Reschedule Event       | Google Calendar Tool          | Updates event timing and attendees                | Calendar MCP                | Calendar MCP Server          |                                                               |
| Delete Calendar Event  | Google Calendar Tool          | Deletes event by ID                               | Calendar MCP                | Calendar MCP Server          |                                                               |
| Task Manager MCP       | MCP Trigger                   | Entry point for task AI operations                 | Get a Task, Create a Task...| -                            | Task Manager MCP ✅ Manages to-dos with ease: create, complete, delete, retrieve tasks |
| Google Tasks MCP       | MCP Client Tool               | Client for task MCP Server                         | Jarvis                      | Jarvis                       | Task Manager MCP ✅ Manages to-dos with ease: create, complete, delete, retrieve tasks |
| Create a Task          | Google Tasks Tool             | Creates a new task                                | Google Tasks MCP            | Task Manager MCP             |                                                               |
| Get a Task             | Google Tasks Tool             | Retrieves a task by ID                            | Google Tasks MCP            | Task Manager MCP             |                                                               |
| Get many Tasks         | Google Tasks Tool             | Retrieves multiple tasks                          | Google Tasks MCP            | Task Manager MCP             |                                                               |
| Complete a Task        | Google Tasks Tool             | Marks task as completed                           | Google Tasks MCP            | Task Manager MCP             |                                                               |
| Delete a Task          | Google Tasks Tool             | Deletes a task                                   | Google Tasks MCP            | Task Manager MCP             |                                                               |
| Finance Manager MCP Server | MCP Trigger                  | Entry for finance AI operations                    | Get all Expenses, Create Expense, Delete Expense | -                            | Finance Manager MCP 💵 Track expenses: create, get, delete entries |
| Finance Tracker        | MCP Client Tool               | Client for finance MCP server                      | Jarvis                      | Jarvis                       | Finance Manager MCP 💵 Track expenses: create, get, delete entries |
| Get all Expenses       | Google Sheets Tool            | Reads expense data from sheet                     | Finance Manager MCP Server  | Finance Manager MCP Server   |                                                               |
| Create Expense         | Google Sheets Tool            | Appends new expense data                          | Finance Manager MCP Server  | Finance Manager MCP Server   |                                                               |
| Delete Expense         | Google Sheets Tool            | Deletes expense data rows                          | Finance Manager MCP Server  | Finance Manager MCP Server   |                                                               |
| Google Contacts MCP    | MCP Trigger                   | Entry for contacts AI operations                   | Get Contacts                | -                            | Google Contacts MCP 👥 Access and manage contacts |
| Google Contacts        | MCP Client Tool               | Client for contacts MCP server                     | Jarvis                      | Jarvis                       | Google Contacts MCP 👥 Access and manage contacts |
| Get Contacts           | Google Contacts Tool          | Retrieves contacts matching query                 | Google Contacts MCP         | Google Contacts MCP          |                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure webhook to listen for "message" updates.  
   - Set credentials with your Telegram Bot token.

2. **Create Filter node "Only allow me":**  
   - Type: Filter  
   - Condition: Allow only if message.chat.username equals "jackman8".  
   - Connect its input from Telegram Trigger.

3. **Create Switch node "Switch" for text/audio detection:**  
   - Type: Switch  
   - Rule "Text": Exists check on `message.text`.  
   - Rule "Audio": Not exists check on `message.text`.  
   - Connect "Only allow me" output to this node.

4. **For audio path:**  
   - Add "Get a file" Telegram node to retrieve voice file by `message.voice.file_id`.  
   - Connect "Switch" Audio output to "Get a file".

5. **Add "Transcribe audio or video" node:**  
   - Type: ElevenLabs Speech-to-Text.  
   - Use ElevenLabs credentials.  
   - Connect input from "Get a file".

6. **Add "Jarvis" Langchain Agent node:**  
   - Type: Langchain Agent.  
   - Configure prompt text to include user message and memory context.  
   - Set system prompt defining Jarvis’s identity, capabilities, and instructions (copy from given system message).  
   - Use OpenAI Chat Model node (GPT-4.1-mini) for language model.  
   - Connect inputs from "Switch" Text output and from "Transcribe audio or video" output.  
   - Connect "Simple Memory" node (Langchain Memory Buffer) keyed on Telegram username as AI memory input.

7. **Add "Simple Memory" node:**  
   - Type: Langchain Memory Buffer Window.  
   - Use Telegram username as session key.  
   - Connect input from Telegram Trigger.  
   - Connect output to Jarvis AI memory input.

8. **Add "OpenAI Chat Model":**  
   - Type: Langchain OpenAI Chat Model.  
   - Set model to "gpt-4.1-mini".  
   - Use OpenAI API credentials.  
   - Connect output to Jarvis AI language model input.

9. **Add "Think" node:**  
   - Type: Langchain Tool Think.  
   - Connect input and output to Jarvis for internal reasoning.

10. **Add "Set Reply Message" node:**  
    - Type: Set.  
    - Assign `message` field from Jarvis’s `output` or `error`.  
    - Connect input from Jarvis.

11. **Add "Check Text or Audio" Switch node:**  
    - Type: Switch.  
    - Rule "Text": Check if Telegram Trigger message has text.  
    - Rule "Audio": Otherwise.  
    - Connect input from "Set Reply Message".

12. **Add "Send a text message" Telegram node:**  
    - Use Telegram Bot credentials.  
    - Set chatId from Telegram Trigger message chat.id.  
    - Text from "Set Reply Message" message.  
    - Connect input from Switch Text output.

13. **Add "Convert text to speech" ElevenLabs node:**  
    - Use ElevenLabs API credentials.  
    - Voice: "Devi - Clear Hindi pronunciation".  
    - Text from "Set Reply Message" message.  
    - Connect input from Switch Audio output.

14. **Add "Send an audio file" Telegram node:**  
    - Use Telegram Bot credentials.  
    - Set chatId from Telegram Trigger message chat.id.  
    - Send audio binary data from ElevenLabs output.  
    - Caption from escaped "Set Reply Message" message.  
    - Connect input from "Convert text to speech".

15. **Set up Gmail MCP Server trigger:**  
    - Type: MCP Trigger.  
    - Create webhook for Gmail MCP operations.

16. **Set up Gmail MCP client tool:**  
    - Connect to Gmail MCP Server SSE endpoint.

17. **Configure Gmail nodes:**  
    - Send Email, Reply to Email, Draft Email, Draft Email Reply, Get Emails, Add Label to Email, Get Labels.  
    - Use Gmail OAuth2 credentials.  
    - Connect outputs to Gmail MCP Server as AI tool.

18. **Set up Calendar MCP Server trigger and client tool:**  
    - Configure webhook and SSE endpoint for calendar operations.

19. **Configure Google Calendar Tool nodes:**  
    - Check Availability, Get all Events, Create an event, Get Event, Reschedule Event, Delete Calendar Event.  
    - Use Google Calendar OAuth2 credentials.

20. **Set up Task Manager MCP trigger and client tool:**  
    - Configure webhook and SSE endpoint for tasks.

21. **Configure Google Tasks Tool nodes:**  
    - Create a Task, Get a Task, Get many Tasks, Complete a Task, Delete a Task.  
    - Use Google Tasks OAuth2 credentials.

22. **Set up Finance Manager MCP Server trigger and client tool:**  
    - Configure webhook and SSE endpoint for finance management.

23. **Configure Google Sheets Tool nodes:**  
    - Get all Expenses, Create Expense, Delete Expense.  
    - Use Google Sheets OAuth2 credentials.  
    - Set Document ID and Sheet Name referencing Expense Tracker spreadsheet.

24. **Set up Google Contacts MCP trigger and client tool:**  
    - Configure webhook and SSE endpoint for contacts.

25. **Configure Google Contacts Tool node:**  
    - Get Contacts node to retrieve contact info.  
    - Use Google Contacts OAuth2 credentials.

26. **Add Sticky Notes:**  
    - Add notes summarizing the blocks for usability and maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| "Jarvis 🤖: Your AI-powered personal assistant orchestrating tasks, calendar, emails, contacts & expenses using MCPs." | General project description in Sticky Note1.                                                          |
| Gmail MCP 📧: Full email management including send, draft, reply, label, and fetch emails.                            | Sticky Note4 in the email management section.                                                         |
| Calendar MCP 📅: Scheduling hub for checking availability, creating, rescheduling, and deleting events.               | Sticky Note3 in the calendar management block.                                                        |
| Task Manager MCP ✅: Manages to-dos with create, complete, delete, retrieve functionality.                            | Sticky Note2 in the task management area.                                                             |
| Finance Manager MCP 💵: Tracks personal/business expenses with create, retrieve, delete operations.                   | Sticky Note5 in the finance tracking block.                                                           |
| Google Contacts MCP 👥: Access and manage contact list for quick communication.                                       | Sticky Note6 in the contacts management block.                                                        |
| Telegram Bot used as primary user interface for conversational input and output including audio support.             | Telegram Trigger and related Telegram nodes.                                                          |
| ElevenLabs API used for speech-to-text and text-to-speech conversion to support audio interactions.                   | Nodes: Transcribe audio or video, Convert text to speech.                                             |
| OpenAI GPT-4.1-mini model powers AI reasoning and natural language processing.                                       | OpenAI Chat Model node with credentials.                                                              |

---

This detailed documentation enables understanding, modification, and full reconstruction of the Jarvis productivity assistant workflow in n8n. It covers all nodes, configurations, integration points, potential failure modes, and logical flow, supporting both human users and AI agents in advanced usage scenarios.