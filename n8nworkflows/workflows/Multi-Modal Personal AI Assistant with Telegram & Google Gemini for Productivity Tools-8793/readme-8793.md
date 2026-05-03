Multi-Modal Personal AI Assistant with Telegram & Google Gemini for Productivity Tools

https://n8nworkflows.xyz/workflows/multi-modal-personal-ai-assistant-with-telegram---google-gemini-for-productivity-tools-8793


# Multi-Modal Personal AI Assistant with Telegram & Google Gemini for Productivity Tools

### 1. Workflow Overview

This workflow implements a sophisticated multi-modal personal AI assistant integrated primarily with Telegram and Google Gemini (PaLM API) to enhance productivity through task, email, calendar, project, and knowledge management. Its core function is to interact with a user (Adam) via Telegram, process queries and commands using AI agents, and manage data across various productivity tools including Airtable (as knowledge base), Todoist (task management), Google Sheets (team/project data), Gmail (email), and Google Calendar (schedule).

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Preprocessing:** Handling Telegram updates (text, voice, image, document), parsing inputs, and merging them into a unified stream.
- **1.2 AI Processing & Memory Management:** Passing inputs to a sophisticated AI agent “Manager Agent” that uses Google Gemini language models with integrated memory buffers and external knowledge base (Airtable).
- **1.3 Multi-Agent Integration:** Dedicated AI agents for specialized domains:
  - Memory Base Agent (personal and contact info)
  - Todo and Task Manager Agent (Todoist integration)
  - Research Agent (web and Wikipedia research)
  - Email Agent (Gmail operations)
  - Project Management Agent (Google Sheets-based projects)
  - Calendar Agent (Google Calendar scheduling)
- **1.4 Scheduled Notifications:** Automatic daily and weekly triggers that prompt the assistant to provide summaries, follow-ups, and research updates.
- **1.5 Data Storage & Synchronization:** Airtable operations for personal data, notifications, and contacts; Google Sheets for team and project tasks; Todoist for task lifecycle management; Gmail and Google Calendar API interactions.
- **1.6 Response Delivery:** Sending processed AI responses back to the user via Telegram.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Preprocessing

**Overview:**  
This block captures user input from Telegram, discriminates message types (text, voice, photo, document), and prepares the data for downstream AI processing.

**Nodes Involved:**  
- Telegram Trigger  
- Switch  
- Edit Fields (2 variants)  
- Telegram1 (voice file fetch)  
- Telegram2 (photo file fetch)  
- Code (image base64 conversion)  
- Merge

**Node Details:**

- **Telegram Trigger**  
  - Type: Trigger node for Telegram updates  
  - Config: Watches for "message" updates  
  - Inputs: Telegram webhook  
  - Outputs: JSON containing Telegram message data  
  - Potential failures: Telegram API errors, webhook misconfiguration, rate limits

- **Switch**  
  - Type: Conditional routing node  
  - Config: Routes messages by type: document, voice, text, photo  
  - Inputs: Telegram Trigger output  
  - Outputs: Branches to relevant Edit Fields or Telegram nodes  
  - Edge cases: Missing fields, unexpected message types

- **Edit Fields** (for document caption)  
  - Type: Set node  
  - Config: Extracts caption text from document message  
  - Inputs: Switch document branch  
  - Outputs: JSON with `text` property for AI input

- **Telegram1**  
  - Type: Telegram node  
  - Config: Fetches voice file by `file_id`  
  - Inputs: Switch voice branch  
  - Outputs: Binary audio data for transcription

- **Edit Fields1** (for text message)  
  - Type: Set node  
  - Config: Extracts plain text from message  
  - Inputs: Switch text branch  
  - Outputs: JSON with `text` property

- **Telegram2**  
  - Type: Telegram node  
  - Config: Fetches photo file binary by `file_id`  
  - Inputs: Switch photo branch  
  - Outputs: Binary image data

- **Code**  
  - Type: Function node  
  - Config: Converts image binary data to base64 encoded string with correct MIME type  
  - Inputs: Telegram2 output (binary image)  
  - Outputs: JSON with base64 image data in `json.base64Image`

- **Merge**  
  - Type: Merge node  
  - Config: Combines all branches from different message types into a single stream  
  - Inputs: Outputs from Edit Fields, Telegram1, Edit Fields1, Telegram2(Code)  
  - Outputs: Unified stream for AI processing

---

#### 2.2 AI Processing & Memory Management

**Overview:**  
This block receives preprocessed input, manages conversational memory, and uses the central AI agent “Manager Agent” with a custom system prompt to generate personalized and context-aware responses.

**Nodes Involved:**  
- Window Buffer Memory  
- Manager Agent  
- Telegram (response sender)

**Node Details:**

- **Window Buffer Memory**  
  - Type: LangChain memory buffer node  
  - Config: Uses a fixed session key `"1235543129"`, keeps last 10 conversational turns for context  
  - Inputs: Preprocessed input from Merge  
  - Outputs: Context-enriched input for Manager Agent  
  - Edge cases: Memory overflow, session key conflicts

- **Manager Agent**  
  - Type: LangChain agent node with Google Gemini LLM  
  - Config: Injects a detailed, personality-rich system prompt defining the assistant “Main” with witty, proactive traits; manages multiple AI tools (memory, calendar, email, todo, project, research) and fallback strategies  
  - Inputs: Text from Window Buffer Memory  
  - Outputs: AI-generated textual response  
  - Failure types: API limits, prompt parsing errors, tool integration failures  
  - Error handling: Falls back to regular output on errors

- **Telegram**  
  - Type: Telegram node  
  - Config: Sends AI output text back to user chat ID `"1235543129"`  
  - Inputs: Manager Agent output  
  - Outputs: Telegram message sent confirmation

---

#### 2.3 Multi-Agent Integration

**Overview:**  
Specialized AI agents handle domain-specific tasks, interfacing with external services and databases.

**Nodes Involved:**

- Memory Base Agent related: memory_base, Airtable nodes (get many records, search personal, create/update/delete records, search notifications, search contacts, get record nodes), Window Buffer Memory1  
- Todo and Task Manager Agent related: todo_and_task_manager, Todoist nodes (create, update, delete, close, reopen task, get tasks), Window Buffer Memory2  
- Research Agent related: research_Agent, Wikipedia, Google Gemini Chat Model3  
- Email Agent related: email_agent, Gmail nodes (Send Email, Create Draft, Email Reply, Get Labels, Label Emails, Mark Unread, Gmail getAll), Google Sheets (team members), Window Buffer Memory2  
- Project Management Agent related: project_management, Google Sheets (Clients task, Company task, update, create, delete tasks), Window Buffer Memory4  
- Calendar Agent related: calendar_agent, Google Calendar nodes (Get all event, Get a single event, Create Event with/without attendee, Update event, Delete event), Window Buffer Memory4

**Node Details (Summary for key nodes):**

- **memory_base (Agent Tool)**  
  - Role: Manages knowledge base and personal data stored in Airtable  
  - System prompt defines storage and retrieval of notifications, personal info, contacts  
  - Interfaces with Airtable tables "Personal", "Notifications", "Contact details"

- **todo_and_task_manager (Agent Tool)**  
  - Role: Full lifecycle task manager using Todoist APIs  
  - Supports creating, updating, deleting, moving, closing, reopening tasks  
  - Uses Todoist OAuth2 credentials  

- **research_Agent (Agent Tool)**  
  - Role: Web and Wikipedia research assistant  
  - Uses Wikipedia API and Google Gemini LLM for summarization and fact verification  

- **email_agent (Agent Tool)**  
  - Role: Email composition, sending, replying, labeling via Gmail  
  - Enforces plain text email body and signature rules  
  - Uses Gmail OAuth2 credentials  

- **project_management (Agent Tool)**  
  - Role: Manages company and client project data in Google Sheets  
  - Supports creating, updating, deleting project tasks and notes  

- **calendar_agent (Agent Tool)**  
  - Role: Manages Google Calendar events (create, update, delete, availability checks)  
  - Sends reminders and manages attendees  

---

#### 2.4 Scheduled Notifications & Commands

**Overview:**  
Two schedule triggers initiate daily and weekly workflows to proactively provide summaries and follow-ups.

**Nodes Involved:**  
- Schedule Trigger (Daily 8 AM)  
- Schedule Trigger1 (Weekly Monday 8 AM)  
- Edit Fields3 (daily summary prompt)  
- Edit Fields4 (weekly follow-up prompt)  
- Merge (to unify scheduled prompts)  
- Manager Agent (to process prompts)

**Node Details:**

- **Schedule Trigger**  
  - Cron expression: 0 8 * * * (every day at 8 AM)  
  - Triggers Edit Fields3 with a detailed prompt requesting tasks, calendar events, notifications, latest tech/business news, and important emails for the day

- **Schedule Trigger1**  
  - Cron expression: 0 08 * * MON (every Monday at 8 AM)  
  - Triggers Edit Fields4 with a prompt for weekly follow-ups to team members on incomplete and completed tasks, including analysis

---

#### 2.5 Data Storage & Synchronization

**Overview:**  
This block handles syncing data to/from Airtable, Google Sheets, and Todoist for knowledge base, project, and task management.

**Nodes Involved:**  
- Airtable nodes for personal data and notifications management (create, update, delete, search)  
- Google Sheets nodes for team members, client tasks, company tasks (create, update, delete)  
- Todoist nodes for task lifecycle operations  
- Google Gemini Chat Models and Buffer Memory nodes supporting these agents

**Node Details:**

- Airtable nodes: Used by memory_base agent to persist and retrieve knowledge base info including personal data, notifications, and contacts  
- Google Sheets nodes: Manage project-related data for company and clients, including tasks, owners, status, notes  
- Todoist nodes: Provide task operations including create, update, delete, close, reopen, move, and get tasks  
- Buffer Memory nodes: Maintain contextual conversation memory for AI agents  

---

#### 2.6 Response Delivery

**Overview:**  
Final AI-generated outputs are sent back to the user via Telegram.

**Nodes Involved:**  
- Telegram (response sender)

**Node Details:**  
- Sends the textual response generated by Manager Agent to the Telegram chat ID `"1235543129"` without additional attribution or formatting to keep responses personal and natural.

---

### 3. Summary Table

| Node Name                 | Node Type                                  | Functional Role                              | Input Node(s)                              | Output Node(s)                   | Sticky Note                                  |
|---------------------------|--------------------------------------------|----------------------------------------------|--------------------------------------------|----------------------------------|----------------------------------------------|
| Telegram Trigger           | Telegram Trigger                           | Incoming Telegram messages                    | -                                          | Switch                           |                                              |
| Switch                    | Switch                                    | Routes message by type                        | Telegram Trigger                           | Edit Fields, Telegram1, Edit Fields1, Telegram2 |                                              |
| Edit Fields               | Set                                       | Extract caption text                          | Switch (document branch)                    | Merge                            |                                              |
| Telegram1                 | Telegram                                  | Fetch voice file                              | Switch (voice branch)                       | Transcribe a recording           |                                              |
| Edit Fields1              | Set                                       | Extract text message                          | Switch (text branch)                        | Merge                            |                                              |
| Telegram2                 | Telegram                                  | Fetch photo file                              | Switch (photo branch)                       | Code                            |                                              |
| Code                      | Function                                  | Convert image binary to base64                | Telegram2                                   | Edit Fields5                    |                                              |
| Merge                     | Merge                                     | Combine all message type branches             | Edit Fields, Telegram1, Edit Fields1, Edit Fields5 | Window Buffer Memory             |                                              |
| Window Buffer Memory      | LangChain Memory Buffer                    | Maintain recent conversational context       | Merge                                       | Manager Agent                   |                                              |
| Manager Agent             | LangChain Agent (Google Gemini)            | Central AI processing and response generation| Window Buffer Memory                        | Telegram                       |                                              |
| Telegram                  | Telegram                                  | Send AI response to user                      | Manager Agent                               | -                                |                                              |
| Schedule Trigger          | Schedule Trigger                          | Trigger daily summary prompt                  | -                                          | Edit Fields3                    |                                              |
| Edit Fields3              | Set                                       | Prepare daily summary prompt                   | Schedule Trigger                           | Merge                            |                                              |
| Schedule Trigger1         | Schedule Trigger                          | Trigger weekly follow-up prompt               | -                                          | Edit Fields4                    |                                              |
| Edit Fields4              | Set                                       | Prepare weekly follow-up prompt                | Schedule Trigger1                          | Merge                            |                                              |
| Window Buffer Memory1     | LangChain Memory Buffer                    | Contextual memory for memory_base agent       | -                                          | memory_base                     | ## Memory Agent                              |
| memory_base               | LangChain Agent Tool                       | Manage knowledge base (Airtable)              | Window Buffer Memory1                       | Manager Agent                   |                                              |
| Airtable nodes            | AirtableTool                              | CRUD operations on personal data, contacts, notifications | memory_base, others                       | memory_base                     |                                              |
| Window Buffer Memory2     | LangChain Memory Buffer                    | Context for email_agent and todo_and_task_manager | -                                          | email_agent, todo_and_task_manager | ## Todo and task manager                    |
| todo_and_task_manager     | LangChain Agent Tool                       | Manage Todoist tasks                           | Window Buffer Memory2                       | Manager Agent                   |                                              |
| Todoist nodes             | TodoistTool                              | Task operations: create, update, delete, etc. | todo_and_task_manager                      | todo_and_task_manager            |                                              |
| research_Agent            | LangChain Agent Tool                       | Research and web/Wikipedia lookup             | Wikipedia, Google Gemini Chat Model3       | Manager Agent                   | ## Research Agent                            |
| email_agent               | LangChain Agent Tool                       | Email composition and management (Gmail)     | Gmail nodes, Google Sheets                  | Manager Agent                   |                                              |
| Gmail nodes               | GmailTool                                | Sending, draft, reply, labeling emails       | email_agent                                | email_agent                     |                                              |
| Window Buffer Memory4     | LangChain Memory Buffer                    | Context for project_management and calendar_agent | -                                          | project_management, calendar_agent | ## Project Manager, ## Calendar Agent       |
| project_management        | LangChain Agent Tool                       | Project and task management (Google Sheets)  | Google Sheets nodes, Window Buffer Memory4 | Manager Agent                   |                                              |
| Google Sheets nodes       | GoogleSheetsTool                         | Manage team member data, client/company tasks | project_management, email_agent            | project_management, email_agent  |                                              |
| calendar_agent            | LangChain Agent Tool                       | Manage Google Calendar events                  | Google Calendar nodes, Window Buffer Memory4 | Manager Agent                   |                                              |
| Google Calendar nodes     | GoogleCalendarTool                       | Calendar operations: create, update, delete  | calendar_agent                             | calendar_agent                  |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure webhook for receiving “message” updates  
   - Connect credentials for Telegram API  
   - Position: Input node  

2. **Add Switch Node:**  
   - Type: Switch  
   - Add rules to detect message types: document exists, voice exists, text exists, photo exists  
   - Connect Telegram Trigger output to Switch input  

3. **Create Edit Fields Nodes:**  
   - For document caption: Extract `message.caption` into `text`  
   - For text messages: Extract `message.text` into `text`  
   - Connect respective Switch outputs to these nodes  

4. **Add Telegram Nodes for Media Fetch:**  
   - Telegram1: Fetch voice file using `message.voice.file_id`  
   - Telegram2: Fetch photo file using `message.photo[0].file_id`  
   - Connect respective Switch outputs  

5. **Add Code Node (for images):**  
   - Convert binary image to base64 with MIME “image/jpeg”  
   - Connect Telegram2 output to Code node  

6. **Merge All Input Branches:**  
   - Merge node with 6 inputs (text from document caption, transcribed voice, text, photo base64)  
   - Connect all Edit Fields, Transcribe recording, Code outputs to Merge  

7. **Add Window Buffer Memory Node:**  
   - Type: LangChain Memory Buffer  
   - Configure with sessionKey `"1235543129"` and context window length 10  
   - Connect Merge output  

8. **Create Manager Agent Node:**  
   - Type: LangChain Agent (Google Gemini)  
   - Provide system prompt as per specification (detailed personality, tools usage, fallback)  
   - Connect Window Buffer Memory output  
   - Provide Google Gemini API credentials  

9. **Add Telegram Node (Response Sender):**  
   - Type: Telegram  
   - Send message to chat ID `"1235543129"` with output from Manager Agent  
   - Connect Manager Agent output  

10. **Set up Schedule Triggers:**  
    - Daily 8 AM: Schedule Trigger triggers Edit Fields3 with daily summary prompt  
    - Weekly Monday 8 AM: Schedule Trigger1 triggers Edit Fields4 with weekly follow-up prompt  
    - Connect both Edit Fields3 and Edit Fields4 outputs to Merge node to unify with main stream  

11. **Add Sub-Workflows / Agent Tools:**  
    - memory_base: LangChain agent with Airtable operations for knowledge base  
    - todo_and_task_manager: LangChain agent with Todoist nodes for task management  
    - research_Agent: LangChain agent with Wikipedia node and Gemini LLM for research  
    - email_agent: LangChain agent with Gmail nodes for email management  
    - project_management: LangChain agent with Google Sheets nodes for project tasks  
    - calendar_agent: LangChain agent with Google Calendar nodes for schedule management  

12. **Configure Airtable Nodes:**  
    - Set up tables: Personal, Notifications, Contact details  
    - CRUD operations for knowledge base management  

13. **Configure Google Sheets Nodes:**  
    - Connect to team member sheet, client tasks, company tasks  
    - Enable append, update, delete operations  

14. **Configure Todoist Nodes:**  
    - OAuth2 credentials  
    - Nodes for create, update, delete, close, reopen, move, get tasks  

15. **Configure Gmail Nodes:**  
    - OAuth2 credentials  
    - Nodes for send email, create draft, reply, label, mark unread, get labels, get emails  

16. **Configure Google Calendar Nodes:**  
    - OAuth2 credentials  
    - Nodes for get all events, get single event, create event (with/without attendees), update, delete, availability check  

17. **Add Window Buffer Memory Nodes for All Agents:**  
    - Ensure each major agent has context memory node with appropriate session keys  

18. **Test Workflow End-to-End:**  
    - Send test messages to Telegram  
    - Validate AI responses, task creation, email sending, calendar event creation  

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| The personal assistant “Main” is designed with a witty, human-like personality inspired by Donna Paulsen from *Suits*.       | System prompt in Manager Agent node                                                                                       |
| Memory base knowledge stored in Airtable includes Notifications, Personal details, and Contacts tables.                      | Airtable setup                                                                                                            |
| Todoist OAuth2 credentials required for task management operations.                                                          | Todoist nodes                                                                                                             |
| Google Gemini (PaLM API) is used for all language model processing; ensure API quota and credentials are valid.               | LangChain nodes with Google Gemini                                                                                         |
| Gmail OAuth2 credentials required for email sending and management.                                                          | Gmail nodes                                                                                                               |
| Google Calendar OAuth2 credentials required for scheduling and calendar event management.                                     | Google Calendar nodes                                                                                                     |
| Google Sheets OAuth2 credentials required for project and client task management.                                             | Google Sheets nodes                                                                                                       |
| This workflow relies on multiple LangChain agent tools; prompt engineering is critical for correct AI behavior.              | Agent tool nodes with custom system messages                                                                               |
| The workflow contains error handling to continue on agent errors but may require monitoring for API rate limits and failures.| Manager Agent configured with `onError: continueRegularOutput`                                                            |
| Sticky notes in workflow visually mark major functional blocks: Memory Agent, Todo and Task Manager, Research Agent, Project Manager, Calendar Agent | Visual workflow annotations                                                                                               |
| For further understanding of LangChain and Google Gemini integration, refer to official n8n and Google PaLM documentation.  | https://docs.n8n.io/nodes/n8n-nodes-langchain/ and Google PaLM API docs                                                   |

---

**Disclaimer:**  
The provided analysis and documentation are based solely on the given n8n workflow JSON export. This workflow operates within legal and ethical boundaries, handling only public and authorized data via integrated APIs and services.