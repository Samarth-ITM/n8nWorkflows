Personal Assistant Bot with Multi-Agent System using Telegram & Google Gemini

https://n8nworkflows.xyz/workflows/personal-assistant-bot-with-multi-agent-system-using-telegram---google-gemini-8582


# Personal Assistant Bot with Multi-Agent System using Telegram & Google Gemini

### 1. Workflow Overview

This workflow implements a sophisticated **Personal Assistant Bot** that uses a multi-agent system architecture, integrating Telegram messaging with AI capabilities powered by Google Gemini and other services. It targets personal productivity and team/project management use cases, acting as a centralized AI assistant for Salman (referred to as "akil" in prompts), handling tasks, calendar events, emails, project tracking, research, and memory management.

**Logical blocks** in the workflow:

- **1.1 Input Reception and Message Processing**  
  Receives Telegram messages (text, voice, documents, images), routes and preprocesses them based on content type.

- **1.2 AI Agent Management**  
  Main conversational AI agent ("Manager Agent") processes input text, leveraging memory and multiple specialized agents for tasks, email, research, calendar, and project management.

- **1.3 Memory Management**  
  Handles storing and retrieving personal data, notifications, contacts, and past conversations from an Airtable-based knowledge base.

- **1.4 Todo and Task Management**  
  Manages creation, updating, deletion, and status tracking of tasks and todos via Todoist integration.

- **1.5 Email Management**  
  Manages sending, drafting, replying, labeling, and marking Gmail emails, integrating with contacts for recipient details.

- **1.6 Research Agent**  
  Performs web and knowledge research using Wolfram Alpha, Wikipedia, and SerpAPI to provide updated and relevant information.

- **1.7 Calendar Management**  
  Manages Google Calendar events including creation, updates, deletions, availability checks, and reminders.

- **1.8 Project Management**  
  Interfaces with Google Sheets to manage company and client projects and tasks, including creation, update, deletion, and progress analysis.

- **1.9 Scheduled Automated Queries**  
  Triggers scheduled queries at specific times (daily and weekly) to prompt updates, summaries, and follow-ups.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Message Processing

**Overview:**  
Handles incoming Telegram messages, distinguishing message types (text, voice, photo, document). Prepares data accordingly for further AI processing.

**Nodes Involved:**  
- Telegram Trigger  
- Switch  
- Edit Fields, Edit Fields1  
- Telegram1 (for voice message files)  
- Telegram2 (for photo files)  
- Code (to convert image binary to base64)  
- Analyze image (Google Gemini image analysis)  
- Transcribe a recording (Google Gemini audio transcription)  
- Merge (merges outputs from various pre-processing branches)

**Node Details:**

- **Telegram Trigger**: Listens for all incoming Telegram messages (text, voice, documents, photos). Uses webhook for real-time updates.

- **Switch**: Routes messages by type:
  - Document messages → pass to Edit Fields (extracts caption text)
  - Voice messages → forward to Telegram1 for file retrieval, then transcribed
  - Text messages → pass to Edit Fields1 (extracts text)
  - Photo messages → forward to Telegram2 for file retrieval and processing

- **Telegram1 & Telegram2**: Fetch voice and photo files respectively from Telegram servers.

- **Code**: Converts retrieved photo binary data to base64 encoded string for AI model consumption.

- **Analyze image**: Uses Google Gemini to analyze and explain photo content.

- **Transcribe a recording**: Uses Google Gemini to transcribe voice recording to text.

- **Edit Fields and Edit Fields1**: Set unified `text` field from various inputs (caption or text message).

- **Merge**: Combines all processed input branches into a single output for the next AI processing step.

**Edge Cases & Failures:**  
- File retrieval failures (Telegram API errors, invalid file IDs).  
- Transcription or image analysis API timeouts or errors.  
- Messages without expected fields may cause expression failures.

---

#### 1.2 AI Agent Management

**Overview:**  
Central AI agent ("Manager Agent") processes the unified text input, interacting with memory, task, email, calendar, research, and project agents to produce responses.

**Nodes Involved:**  
- Manager Agent  
- Window Buffer Memory  
- Telegram (sending replies)

**Node Details:**

- **Window Buffer Memory**: Maintains contextual windowed conversation memory keyed by a fixed session key (`1236574355`). Stores last 10 interactions to provide conversational context.

- **Manager Agent**:  
  - Uses Google Gemini chat model with a defined system prompt characterizing the assistant as a witty, proactive personal assistant ("Main") for Salman.  
  - Integrates with other agents/tools (memory_base, calendar_agent, email_agent, todo_and_task_manager, project_management, research_Agent).  
  - Saves important info to memory base and schedules retry or delegation if tools fail.  
  - Provides conversational and task/project management in a human-like style.  
  - On error, continues output without failing workflow.

- **Telegram**: Sends the final AI-generated text response back to the Telegram chat ID (`1236574355` fixed).

**Edge Cases & Failures:**  
- AI model API failures or timeouts.  
- Memory agent failures preventing context retrieval.  
- Telegram message sending failures (chat ID issues).  
- Unexpected input formats.

---

#### 1.3 Memory Management

**Overview:**  
Agent dedicated to storing, retrieving, and managing personal data, contacts, notifications, and past conversations in Airtable knowledge base tables.

**Nodes Involved:**  
- memory_base (agent tool)  
- Airtable nodes: get many records, search, create, update, delete for "Personal", "Notifications", "Contact details" tables  
- Window Buffer Memory1 (context memory for memory_base agent)

**Node Details:**

- **memory_base Agent**:  
  Role: Salman’s brain storing personal and company information, contacts, notifications.  
  Uses Airtable as backing store.  
  Accesses and manages tables: Notifications, Personal, Contacts.

- **Airtable nodes**:  
  Provide CRUD operations on the knowledge base.  
  Configured with Airtable base and specific table IDs/names.

- **Window Buffer Memory1**: Maintains contextual memory for agent conversations.

**Edge Cases & Failures:**  
- Airtable API rate limits or authentication errors.  
- Missing or malformed data in tables.  
- Conflicts on upsert operations.

---

#### 1.4 Todo and Task Management

**Overview:**  
Manages personal and company tasks using Todoist integration. Supports task lifecycle operations: create, update, delete, close, reopen, move, fetch.

**Nodes Involved:**  
- todo_and_task_manager (agent tool)  
- Todoist nodes: create task, update task, delete task, close task, reopen task, move a task, get many tasks, get a single task  
- Google Gemini Chat Model2  
- Window Buffer Memory2 (context memory for task manager)  
- Sticky Note1 (labeling block)

**Node Details:**

- **todo_and_task_manager Agent**:  
  Task and project manager AI assistant for Salman.  
  Converts user requests into Todoist API operations.  
  Keeps tasks organized and timely.

- **Todoist nodes**:  
  Full coverage of task management API operations with dynamic parameters from AI agent.

- **Window Buffer Memory2**: Context memory for task manager conversations.

**Edge Cases & Failures:**  
- Todoist API rate limits, authentication failures.  
- Task ID not found errors on updates or deletes.  
- Input parsing errors from AI agent instructions.

---

#### 1.5 Email Management

**Overview:**  
Handles email composition, sending, drafting, replying, labeling, and unread marking via Gmail integration, supporting professional email workflows.

**Nodes Involved:**  
- email_agent (agent tool)  
- Gmail nodes: Send Email, Create Draft, Email Reply, Get Labels, Label Emails, Mark Unread  
- Window Buffer Memory2 (shared context with task manager)  
- Google Sheets (fetches Team Members contacts for email recipients)  
- Google Gemini Chat Model4  
- Sticky Note3

**Node Details:**

- **email_agent Agent**:  
  AI email assistant composing plain text professional emails with a signature.  
  Immediately sends emails unless a draft is explicitly requested.  
  Manages labels and email threading.

- **Gmail nodes**:  
  Cover sending, drafting, replying, labeling, unread marking with dynamic data from AI agent.

- **Google Sheets**:  
  Reads team member contacts to retrieve email addresses for recipients.

- **Window Buffer Memory2**: Context for email conversations.

**Edge Cases & Failures:**  
- Gmail API quota or auth errors.  
- Incorrect recipient extraction from contacts.  
- Email sending failures or network issues.

---

#### 1.6 Research Agent

**Overview:**  
Performs web and factual research, gathering updated info from multiple sources and summarizing it for Salman.

**Nodes Involved:**  
- research_Agent (agent tool)  
- SerpAPI, Wikipedia, Wolfram Alpha nodes  
- Google Gemini Chat Model3  
- Sticky Note2

**Node Details:**

- **research_Agent Agent**:  
  Proactively selects appropriate research tools based on query.  
  Summarizes, compares, verifies facts, and suggests next steps.

- **SerpAPI, Wikipedia, Wolfram Alpha**:  
  Provide search, encyclopedia, and computational knowledge respectively.

- **Google Gemini Chat Model3**:  
  Processes and formats research results.

**Edge Cases & Failures:**  
- API failures or rate limits on external knowledge sources.  
- Inconsistent or outdated info returned.  
- Tool selection errors.

---

#### 1.7 Calendar Management

**Overview:**  
AI agent manages Google Calendar events: querying availability, creating, updating, deleting events, and sending reminders.

**Nodes Involved:**  
- calendar_agent (agent tool)  
- Google Calendar nodes: Get all event, Get a single event, Create Event with attendee, Create Event without attendee, Delete event, Update event, Availability operation agent  
- Google Gemini Chat Model5  
- Sticky Note5

**Node Details:**

- **calendar_agent Agent**:  
  Manages scheduling efficiency, availability checks, and reminders.  
  Provides event details to other agents.

- **Google Calendar nodes**:  
  Full CRUD operations on calendar events with attendee management.

- **Google Gemini Chat Model5**:  
  Processes calendar-related AI conversations.

**Edge Cases & Failures:**  
- Google Calendar API errors (auth, quota).  
- Conflicts in scheduling or event overlaps.  
- Missing event IDs or invalid parameters.

---

#### 1.8 Project Management

**Overview:**  
Manages company and client projects and tasks tracked in Google Sheets, including updates, deletes, and detailed analysis.

**Nodes Involved:**  
- project_management (agent tool)  
- Google Sheets nodes: Clients task, Company task, update notes in company/client task, create company/clients task, delete company/client task  
- Google Gemini Chat Model6  
- Window Buffer Memory4  
- Sticky Note4

**Node Details:**

- **project_management Agent**:  
  Oversees project data, task progress, notes, and feedback.  
  Interfaces with Google Sheets as project database.

- **Google Sheets nodes**:  
  Perform append, update, and delete operations on client and company task sheets.

- **Window Buffer Memory4**: Context memory for project management.

**Edge Cases & Failures:**  
- Google Sheets API limits or auth errors.  
- Data consistency on concurrent updates.  
- Missing or invalid row identifiers.

---

#### 1.9 Scheduled Automated Queries

**Overview:**  
Two scheduled triggers prompt the bot daily and weekly to generate summaries, reminders, follow-ups, and research reports.

**Nodes Involved:**  
- Schedule Trigger (daily at 8:00 am)  
- Schedule Trigger1 (weekly on Monday 8:00 am)  
- Edit Fields3 (daily query text)  
- Edit Fields4 (weekly query text)  
- Merge (integrates these queries into the main AI processing flow)

**Node Details:**

- **Schedule Trigger and Schedule Trigger1**: Cron-based triggers for automation.

- **Edit Fields3 & Edit Fields4**: Prepare complex prompt texts requesting schedules, todos, notifications, news, emails (daily), and team follow-ups with analysis (weekly).

- **Merge**: Integrates scheduled queries into main agent processing.

**Edge Cases & Failures:**  
- Cron expression misconfiguration.  
- AI agent unavailability at scheduled times.  
- Delays or failures in processing scheduled prompts.

---

### 3. Summary Table

| Node Name                   | Node Type                              | Functional Role                         | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                     |
|-----------------------------|--------------------------------------|---------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger            | telegramTrigger                      | Receive Telegram messages              | -                                | Switch                          |                                                                                                |
| Switch                     | switch                              | Route messages by type                 | Telegram Trigger                 | Edit Fields, Telegram1, Edit Fields1, Telegram2 |                                                                                                |
| Edit Fields                | set                                 | Extract caption text from document    | Switch                         | Merge                          |                                                                                                |
| Telegram1                  | telegram                            | Retrieve voice file                    | Switch                         | Transcribe a recording          |                                                                                                |
| Edit Fields1               | set                                 | Extract text from text message         | Switch                         | Merge                          |                                                                                                |
| Telegram2                  | telegram                            | Retrieve photo file                    | Switch                         | Code                          |                                                                                                |
| Code                       | code                                | Convert image binary to base64         | Telegram2                      | Analyze image                  |                                                                                                |
| Analyze image              | googleGemini                        | Analyze photo content                  | Code                          | Edit Fields5                   |                                                                                                |
| Edit Fields5               | set                                 | Prepare text from image analysis       | Analyze image                  | Merge                          |                                                                                                |
| Merge                      | merge                               | Combine inputs for AI processing       | Edit Fields, Edit Fields1, Transcribe a recording, Edit Fields5 | Manager Agent                 |                                                                                                |
| Manager Agent              | langchain.agent                     | Main AI assistant processing input     | Merge                         | Telegram                      |                                                                                                |
| Telegram                   | telegram                            | Send AI response back to Telegram      | Manager Agent                 | -                             |                                                                                                |
| Window Buffer Memory       | memoryBufferWindow                  | Conversation context memory for Manager Agent | -                          | Manager Agent                 |                                                                                                |
| Schedule Trigger           | scheduleTrigger                    | Trigger daily scheduled query          | -                              | Edit Fields3                  |                                                                                                |
| Edit Fields3               | set                                 | Create daily schedule query prompt     | Schedule Trigger              | Merge                         |                                                                                                |
| Schedule Trigger1          | scheduleTrigger                    | Trigger weekly scheduled query         | -                              | Edit Fields4                  |                                                                                                |
| Edit Fields4               | set                                 | Create weekly team follow-up prompt    | Schedule Trigger1             | Merge                         |                                                                                                |
| Window Buffer Memory1      | memoryBufferWindow                  | Context memory for memory_base agent   | -                              | memory_base                   | ## Memory Agent                                                                                |
| memory_base                | langchain.agentTool                | Stores and retrieves personal data     | -                              | Manager Agent                 | ## Memory Agent                                                                                |
| get many records           | airtableTool                      | Read records from Airtable knowledge base | -                          | memory_base                   |                                                                                                |
| Search - personal          | airtableTool                      | Search personal data in Airtable       | -                              | memory_base                   |                                                                                                |
| Create a record - personal | airtableTool                      | Create new personal record in Airtable | -                              | memory_base                   |                                                                                                |
| Create or Update Existing - personal | airtableTool              | Upsert personal data record             | -                              | memory_base                   |                                                                                                |
| delete record - personal   | airtableTool                      | Delete personal record                   | -                              | memory_base                   |                                                                                                |
| get record - personal      | airtableTool                      | Get personal record by ID                | -                              | memory_base                   |                                                                                                |
| Search notifications       | airtableTool                      | Search notifications in Airtable        | -                              | memory_base                   |                                                                                                |
| get record - notifications | airtableTool                      | Get notification record by ID            | -                              | memory_base                   |                                                                                                |
| get record - contacts      | airtableTool                      | Get contact details by ID                | -                              | memory_base                   |                                                                                                |
| Search contacts            | airtableTool                      | Search contact details                   | -                              | memory_base                   |                                                                                                |
| Window Buffer Memory2      | memoryBufferWindow                  | Context memory for email_agent and task manager | -                              | email_agent, todo_and_task_manager | ## Todo and task manager                                                                      |
| todo_and_task_manager      | langchain.agentTool                | AI-powered todo/task management          | -                              | Manager Agent                 | ## Todo and task manager                                                                      |
| create task                | todoistTool                      | Create a task in Todoist                  | -                              | todo_and_task_manager         |                                                                                                |
| update task                | todoistTool                      | Update an existing task                    | -                              | todo_and_task_manager         |                                                                                                |
| delete task                | todoistTool                      | Delete a task                             | -                              | todo_and_task_manager         |                                                                                                |
| close task                 | todoistTool                      | Close (complete) a task                   | -                              | todo_and_task_manager         |                                                                                                |
| reopen task                | todoistTool                      | Reopen a closed task                      | -                              | todo_and_task_manager         |                                                                                                |
| move a task                | todoistTool                      | Move task to another project              | -                              | todo_and_task_manager         |                                                                                                |
| Get many tasks             | todoistTool                      | Retrieve multiple tasks                    | -                              | todo_and_task_manager         |                                                                                                |
| Get a single task          | todoistTool                      | Retrieve a single task                     | -                              | todo_and_task_manager         |                                                                                                |
| Google Gemini Chat Model2  | lmChatGoogleGemini               | Language model for todo_and_task_manager  | -                              | todo_and_task_manager         |                                                                                                |
| email_agent                | langchain.agentTool              | AI email assistant                         | -                              | Manager Agent                 | ## Todo and task manager                                                                      |
| Gmail                     | gmailTool                       | Retrieve emails                            | -                              | email_agent                   |                                                                                                |
| Send Email                 | gmailTool                       | Send emails                               | -                              | email_agent                   |                                                                                                |
| Create Draft               | gmailTool                       | Create email drafts                        | -                              | email_agent                   |                                                                                                |
| Email Reply                | gmailTool                       | Reply to emails                            | -                              | email_agent                   |                                                                                                |
| Get Labels                 | gmailTool                       | Get Gmail labels                           | -                              | email_agent                   |                                                                                                |
| Label Emails               | gmailTool                       | Add labels to emails                       | -                              | email_agent                   |                                                                                                |
| Mark Unread               | gmailTool                       | Mark emails as unread                       | -                              | email_agent                   |                                                                                                |
| Google Sheets              | googleSheetsTool                | Retrieve team member contacts for email   | -                              | email_agent                   |                                                                                                |
| Google Gemini Chat Model4  | lmChatGoogleGemini               | Language model for email_agent             | -                              | email_agent                   |                                                                                                |
| research_Agent             | langchain.agentTool              | AI research assistant                       | -                              | Manager Agent                 | ## Research Agent                                                                            |
| SerpAPI                   | toolSerpApi                     | Web search API                              | -                              | research_Agent               |                                                                                                |
| Wikipedia                 | toolWikipedia                   | Encyclopedia lookup                         | -                              | research_Agent               |                                                                                                |
| Wolfram Alpha             | toolWolframAlpha               | Computational knowledge                     | -                              | research_Agent               |                                                                                                |
| Google Gemini Chat Model3  | lmChatGoogleGemini               | Language model for research_Agent          | -                              | research_Agent               |                                                                                                |
| calendar_agent             | langchain.agentTool              | AI calendar assistant                       | -                              | Manager Agent                 | ## Calendar Agent                                                                          |
| Google Calendar nodes (Get all event, Get single event, Create Event with attendee, Create Event without attendee, Delete event, Update event, Availablity operation agent) | googleCalendarTool | Various calendar CRUD and availability checks    | -                              | calendar_agent                 |                                                                                                |
| Google Gemini Chat Model5  | lmChatGoogleGemini               | Language model for calendar_agent           | -                              | calendar_agent               |                                                                                                |
| project_management         | langchain.agentTool              | AI project and task manager                  | -                              | Manager Agent                 | ## Project Manager                                                                        |
| Google Sheets nodes (Clients task, Company task, update notes in company/client task, create company/clients task, delete company/client task) | googleSheetsTool                | Manage project and client tasks data           | -                              | project_management           |                                                                                                |
| Google Gemini Chat Model6  | lmChatGoogleGemini               | Language model for project_management        | -                              | project_management           |                                                                                                |
| Window Buffer Memory4      | memoryBufferWindow              | Context memory for project_management        | -                              | project_management           |                                                                                                |
| Schedule Trigger           | scheduleTrigger                | Daily scheduled query trigger                 | -                              | Edit Fields3                |                                                                                                |
| Schedule Trigger1          | scheduleTrigger                | Weekly scheduled query trigger                | -                              | Edit Fields4                |                                                                                                |
| Edit Fields3               | set                             | Prepare daily query prompt                    | Schedule Trigger               | Merge                       |                                                                                                |
| Edit Fields4               | set                             | Prepare weekly query prompt                   | Schedule Trigger1              | Merge                       |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: `telegramTrigger`  
   - Parameters: Listen to "message" updates, set webhook ID.  
   - Position: near left center.

2. **Create Switch node**  
   - Type: `switch`  
   - Configure rules based on existence of `message.document`, `message.voice`, `message.text`, `message.photo[0]` to route accordingly.  
   - Connect Telegram Trigger output to Switch.

3. **For document messages:**  
   - Add `Set` node named "Edit Fields" to extract `text` from `message.caption`. Connect Switch output to it.

4. **For voice messages:**  
   - Add `Telegram` node "Telegram1" to get voice file by `message.voice.file_id`.  
   - Connect Switch output to Telegram1.  
   - Add `Google Gemini` node "Transcribe a recording" configured for audio transcription, linked from Telegram1.

5. **For text messages:**  
   - Add `Set` node "Edit Fields1" to extract `text` from `message.text`. Connect Switch output.

6. **For photo messages:**  
   - Add `Telegram` node "Telegram2" to get photo file by `message.photo[0].file_id`. Connect Switch output.  
   - Add `Code` node to convert binary to base64 string. Connect from Telegram2.  
   - Add `Google Gemini` node "Analyze image" configured with model "gemini-2.5-pro" for image analysis. Connect from Code.  
   - Add `Set` node "Edit Fields5" to extract `content` text from image analysis output. Connect from Analyze image.

7. **Add Merge node**  
   - Configure for 6 inputs to combine outputs from Edit Fields, Edit Fields1, Transcribe a recording, Edit Fields5, etc.  
   - Connect all branches to Merge.

8. **Add Window Buffer Memory**  
   - For conversation context, configure with session key `"1236574355"`. Connect output to Manager Agent.

9. **Add Manager Agent node**  
   - Type: langchain.agent  
   - Configure system prompt as defined (witty personal assistant "Main").  
   - Set text input from merged `text`.  
   - Add integrations for agents: memory_base, calendar_agent, email_agent, todo_and_task_manager, project_management, research_Agent.  
   - Connect Window Buffer Memory output to Manager Agent input.

10. **Add Telegram node**  
    - Send response text from Manager Agent output to Telegram chat ID `"1236574355"`.

11. **Build Memory Management block:**  
    - Add `memory_base` agent tool node with system message for storing/retrieving Airtable data.  
    - Add Airtable nodes: get many, search, create, update, delete for tables "Personal", "Notifications", "Contact details".  
    - Add Window Buffer Memory1 for memory_base context.

12. **Build Todo and Task Management block:**  
    - Add `todo_and_task_manager` agent tool.  
    - Add Todoist nodes: create, update, delete, close, reopen, move, get many, get single task.  
    - Add Google Gemini Chat Model2 node.  
    - Add Window Buffer Memory2 for context.

13. **Build Email Management block:**  
    - Add `email_agent` agent tool.  
    - Add Gmail nodes: send email, create draft, reply, get labels, label emails, mark unread.  
    - Add Google Sheets node to fetch contacts.  
    - Add Google Gemini Chat Model4 node.  
    - Use Window Buffer Memory2 shared with task manager.

14. **Build Research Agent block:**  
    - Add `research_Agent` agent tool.  
    - Add SerpAPI, Wikipedia, Wolfram Alpha nodes for research.  
    - Add Google Gemini Chat Model3 node.

15. **Build Calendar Management block:**  
    - Add `calendar_agent` agent tool.  
    - Add Google Calendar nodes: get all events, get single event, create event (with/without attendees), delete event, update event, availability.  
    - Add Google Gemini Chat Model5 node.

16. **Build Project Management block:**  
    - Add `project_management` agent tool.  
    - Add Google Sheets nodes for client and company tasks: read, create, update, delete operations.  
    - Add Google Gemini Chat Model6 node.  
    - Add Window Buffer Memory4 for context.

17. **Add Scheduled Triggers:**  
    - Create `Schedule Trigger` cron node for daily 8:00 am trigger.  
    - Create `Schedule Trigger1` cron node for weekly Monday 8:00 am trigger.  
    - Add Edit Fields3 and Edit Fields4 nodes to prepare detailed prompt texts for daily and weekly queries.  
    - Connect these to Merge node feeding Manager Agent.

18. **Establish connections:**  
    - Connect all nodes as per logic: Telegram Trigger → Switch → preprocessing → Merge → Window Memory → Manager Agent → Telegram send.  
    - Connect agents tools with their respective memory and AI models.  
    - Connect Airtable, Todoist, Gmail, Google Sheets, Google Calendar nodes to respective agents.  
    - Connect scheduled triggers to prompt nodes and then to main AI processing.

19. **Configure Credentials:**  
    - Telegram: OAuth2 with bot token and webhook setup.  
    - Google Gemini (PaLM): API keys for language and multimodal models.  
    - Airtable: API key with base and table access.  
    - Todoist: OAuth2 or API token.  
    - Gmail: OAuth2 with sending and label management scopes.  
    - Google Sheets and Google Calendar: OAuth2 with read/write scopes.  
    - SerpAPI: API key.  
    - Wolfram Alpha: API key.

20. **Test each segment independently and end-to-end.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The assistant's personality is modeled after "Donna Paulsen" from Suits — witty, proactive, caring but direct. | System prompt for 'Manager Agent' node defines this character style.                                   |
| Knowledge base Airtable includes tables: "Notifications", "Personal", "Contacts" storing memory and contacts. | Refer to memory_base agent node description.                                                          |
| Uses Google Gemini (PaLM) API for multimodal AI: chat, image analysis, audio transcription.               | Requires Google Palm API credentials.                                                                  |
| Scheduled daily (8 AM) and weekly (Monday 8 AM) queries automate reminders, research, and team follow-ups. | Cron expressions: `0 8 * * *` (daily), `0 08 * * MON` (weekly).                                       |
| Project and client task data managed via Google Sheets document with multiple sheets for tasks and notes. | Google Sheets document IDs and sheet names are configured in respective nodes.                         |
| Email assistant "Jeni, Salman's Assistant" signs off emails automatically.                               | Defined in email_agent system message.                                                                 |
| AI agents communicate using Langchain framework nodes, enabling modular tool integration and memory.    | Ensure n8n v1.3+ for Langchain nodes compatibility.                                                   |
| Telegram chat ID fixed as "1236574355" for sending responses; adapt as needed for other users.           | Hardcoded in Telegram send nodes.                                                                      |
| Sticky Notes label major logical blocks for clarity inside workflow editor.                              | Visible in workflow UI to guide maintenance and enhancements.                                         |

---

This documentation fully captures the workflow's architecture, logic, node configurations, and integration points, enabling expert users and AI agents to understand, reproduce, or modify this Personal Assistant Bot workflow effectively.