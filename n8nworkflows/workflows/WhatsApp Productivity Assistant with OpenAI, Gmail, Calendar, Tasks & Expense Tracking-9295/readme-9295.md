WhatsApp Productivity Assistant with OpenAI, Gmail, Calendar, Tasks & Expense Tracking

https://n8nworkflows.xyz/workflows/whatsapp-productivity-assistant-with-openai--gmail--calendar--tasks---expense-tracking-9295


# WhatsApp Productivity Assistant with OpenAI, Gmail, Calendar, Tasks & Expense Tracking

### 1. Workflow Overview

This workflow, titled **"WhatsApp Productivity Assistant with OpenAI, Gmail, Calendar, Tasks & Expense Tracking"**, functions as a comprehensive AI-powered personal productivity assistant ("Jarvis") accessible via WhatsApp. It integrates multiple Google services (Gmail, Calendar, Tasks, Contacts, Sheets) and OpenAI's GPT-4.1-mini model, along with ElevenLabs speech-to-text and text-to-speech capabilities, to automate daily productivity tasks.

**Target Use Cases:**  
- Managing emails: reading, sending, replying, labeling, and drafting via Gmail.  
- Scheduling and managing calendar events in Google Calendar.  
- Creating, updating, completing, and deleting Google Tasks.  
- Tracking expenses through Google Sheets with creation, retrieval, and deletion of expense entries.  
- Accessing and retrieving contact information from Google Contacts.  
- Handling WhatsApp messages including text and audio, with transcription and audio replies.  
- Context-aware AI interaction with multi-modal memory and dynamic tool usage.

**Logical Blocks:**  
- **1.1 WhatsApp Input Reception & Filtering**  
- **1.2 Message Type Detection & Audio Processing**  
- **1.3 AI Core Processing (Jarvis Agent + OpenAI Model + Memory)**  
- **1.4 Gmail Management MCP (Multi-Client Protocol) Subsystem**  
- **1.5 Google Calendar MCP Subsystem**  
- **1.6 Google Tasks MCP Subsystem**  
- **1.7 Finance Tracking via Google Sheets MCP Subsystem**  
- **1.8 Google Contacts MCP Subsystem**  
- **1.9 WhatsApp Response (Text and Audio) Sending**  
- **1.10 Utility Nodes (Memory, Filters, Switches)**  

---

### 2. Block-by-Block Analysis

---

#### 1.1 WhatsApp Input Reception & Filtering

**Overview:**  
Receives incoming WhatsApp messages, filters them to allow only messages from a specific user, and routes them for further processing.

**Nodes Involved:**  
- WhatsApp Trigger  
- Only allow me (Filter)  
- Switch (Check Text or Audio)  

**Node Details:**  

- **WhatsApp Trigger**  
  - Type: WhatsApp message webhook trigger  
  - Configuration: Listens for incoming WhatsApp messages via webhook updates (messages)  
  - Credentials: WhatsApp API  
  - Input: Incoming WhatsApp message JSON  
  - Output: Message data forwarded downstream  
  - Edge cases: Webhook downtime, malformed messages, auth errors  

- **Only allow me (Filter)**  
  - Type: Filter node  
  - Configuration: Passes only messages where sender ID equals "919920842422" (presumably the owner's number)  
  - Input: WhatsApp Trigger output  
  - Output: Messages from authorized user continue, others dropped  
  - Edge cases: Unauthorized messages ignored silently  

- **Switch (Check Text or Audio)**  
  - Type: Switch node  
  - Configuration: Checks whether message type is "text" or "audio"  
  - Input: Filter output  
  - Output: Routes text messages to AI processing and audio messages to media URL extraction  
  - Edge cases: Unsupported message types ignored  

---

#### 1.2 Message Type Detection & Audio Processing

**Overview:**  
Processes audio messages by retrieving media URLs, downloading audio, transcribing audio to text, then forwarding text to the AI engine.

**Nodes Involved:**  
- Get Media URL  
- Download Audio  
- Transcribe audio or video  

**Node Details:**  

- **Get Media URL**  
  - Type: WhatsApp API node  
  - Configuration: Retrieves media URL for the audio message id  
  - Input: Audio message metadata  
  - Output: Media URL for audio file  
  - Edge cases: Media not found, expired URLs, API errors  

- **Download Audio**  
  - Type: HTTP Request  
  - Configuration: Downloads audio media from obtained URL using WhatsApp credentials  
  - Input: Media URL  
  - Output: Binary audio data passed downstream  
  - Edge cases: Download failures, network errors  

- **Transcribe audio or video**  
  - Type: ElevenLabs speech-to-text node  
  - Configuration: Converts audio binary data to text transcription  
  - Credentials: ElevenLabs API  
  - Input: Downloaded audio file  
  - Output: Transcribed text forwarded for AI processing  
  - Edge cases: Transcription inaccuracies, API timeouts  

---

#### 1.3 AI Core Processing (Jarvis Agent + OpenAI Model + Memory)

**Overview:**  
Central AI processing block that interprets user requests, utilizes OpenAI GPT-4.1-mini model, maintains conversation memory, and orchestrates tool usage.

**Nodes Involved:**  
- Simple Memory  
- OpenAI Chat Model  
- Jarvis (Langchain Agent)  
- Think (Tool Think node)  
- Set Reply Message  

**Node Details:**  

- **Simple Memory**  
  - Type: Langchain Memory Buffer (windowed)  
  - Configuration: Maintains conversation state/session with sessionKey "1"  
  - Input: Conversation data  
  - Output: Extended context with memory for agent usage  
  - Edge cases: Memory overflow or loss on restart  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model node  
  - Configuration: Uses "gpt-4.1-mini" model to generate AI responses  
  - Credentials: OpenAI API key  
  - Input: Prompt plus memory context  
  - Output: AI-generated text output  
  - Edge cases: API rate limits, model errors, malformed prompts  

- **Jarvis (Langchain Agent)**  
  - Type: Langchain agent node  
  - Configuration: Custom system prompt defining Jarvis identity, capabilities (email, calendar, tasks, finance, contacts), communication guidelines, operational rules, and privacy instructions  
  - Input: User input text (transcribed or text) plus context  
  - Output: Structured commands or responses for downstream nodes  
  - Edge cases: Misinterpretation of user intents, incomplete parameters  

- **Think (Tool Think node)**  
  - Type: Langchain tool think node  
  - Configuration: Used for internal reasoning or pre-processing before Jarvis action  
  - Input/Output: Intermediate AI thought process  
  - Edge cases: None critical, mainly internal  

- **Set Reply Message**  
  - Type: Set node  
  - Configuration: Sets output message for replying to user, either AI output or error message  
  - Input: Jarvis node output  
  - Output: Message string for WhatsApp reply  

---

#### 1.4 Gmail Management MCP (Multi-Client Protocol) Subsystem

**Overview:**  
Enables AI-powered email management including sending, replying, drafting, labeling, and fetching emails via Gmail.

**Nodes Involved:**  
- Gmail MCP Server (MCP Trigger)  
- Gmail MCP (MCP Client)  
- Send Email  
- Reply to an Email  
- Draft Email  
- Draft Email Reply  
- Get Emails  
- Add Label to Email  
- Get Labels  

**Node Details:**  

- **Gmail MCP Server**  
  - Type: Langchain MCP Trigger node  
  - Configuration: Listens on webhook path for Gmail MCP requests from AI  
  - Output: Routes AI email requests to Gmail tools  
  - Edge cases: Webhook downtime, auth errors  

- **Gmail MCP**  
  - Type: Langchain MCP Client node  
  - Configuration: Invoked by AI to perform Gmail operations  
  - Input: AI commands  
  - Output: Email operation results  

- **Send Email / Reply to an Email / Draft Email / Draft Email Reply**  
  - Type: Gmail Tool nodes  
  - Configuration: Various Gmail operations with AI-configurable parameters for addresses, subject, body, attachments, CC, BCC, thread IDs  
  - Credentials: Gmail OAuth2  
  - Edge cases: Invalid email addresses, API quota, malformed content  

- **Get Emails / Add Label to Email / Get Labels**  
  - Type: Gmail tool nodes for fetching emails, labeling, and retrieving labels  
  - Configuration: Supports filters, label management  
  - Edge cases: Large data sets, permission errors  

---

#### 1.5 Google Calendar MCP Subsystem

**Overview:**  
Manages calendar functionality including event creation, updates, rescheduling, deletion, and availability checks.

**Nodes Involved:**  
- Calendar MCP Server (MCP Trigger)  
- Calendar MCP (MCP Client)  
- Check Availability  
- Get all Events  
- Delete Calendar Event  
- Reschedule Event  
- Get Event  
- Create an event  

**Node Details:**  

- **Calendar MCP Server**  
  - Type: Langchain MCP Trigger node  
  - Configuration: Webhook trigger for calendar-related AI requests  
  - Edge cases: Webhook availability, API errors  

- **Calendar MCP**  
  - Type: Langchain MCP Client node  
  - Configuration: Client interface for calendar operations invoked by AI  

- **Check Availability / Get all Events / Delete Calendar Event / Reschedule Event / Get Event / Create an event**  
  - Type: Google Calendar Tool nodes  
  - Configuration: Operations include checking free slots, fetching events (with filters), deleting, updating event times and attendees, and creating new events  
  - Credentials: Google Calendar OAuth2  
  - Edge cases: Conflicting events, invalid date/times, permission errors  

---

#### 1.6 Google Tasks MCP Subsystem

**Overview:**  
Handles Google Tasks operations: creating, completing, deleting, and retrieving tasks.

**Nodes Involved:**  
- Task Manager MCP (Langchain MCP Trigger)  
- Google Tasks MCP (Langchain MCP Client)  
- Create a Task  
- Complete a Task  
- Delete a Task  
- Get many Tasks  
- Get a Task  

**Node Details:**  

- **Task Manager MCP**  
  - Type: MCP Trigger node for tasks  
  - Configuration: Webhook entry point for AI task commands  

- **Google Tasks MCP**  
  - Type: MCP Client node for tasks  

- **Create a Task / Complete a Task / Delete a Task / Get many Tasks / Get a Task**  
  - Type: Google Tasks Tool nodes  
  - Configuration: Task creation with title, notes, due dates; marking completed with timestamps; deletion; retrieving single or multiple tasks  
  - Credentials: Google Tasks OAuth2  
  - Edge cases: Invalid task IDs, past due dates, API failures  

---

#### 1.7 Finance Tracking via Google Sheets MCP Subsystem

**Overview:**  
Manages expense tracking using Google Sheets: creating, retrieving, and deleting expense records.

**Nodes Involved:**  
- Finance Manager MCP Server (MCP Trigger)  
- Finance Tracker (MCP Client)  
- Get all Expenses  
- Create Expense  
- Delete Expense  

**Node Details:**  

- **Finance Manager MCP Server**  
  - Type: MCP Trigger node  
  - Configuration: Receives AI requests related to finance spreadsheet operations  

- **Finance Tracker**  
  - Type: MCP Client node for finance  

- **Get all Expenses / Create Expense / Delete Expense**  
  - Type: Google Sheets Tool nodes  
  - Configuration: Reading sheet data, appending new rows (expenses), and clearing specific rows for deletions  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Data loss risk on deletion, invalid ranges, API limits  

---

#### 1.8 Google Contacts MCP Subsystem

**Overview:**  
Provides access to Google Contacts for retrieving contact information to assist email and communication tasks.

**Nodes Involved:**  
- Google Contacts MCP (MCP Trigger)  
- Google Contacts (MCP Client)  
- Get Contacts  

**Node Details:**  

- **Google Contacts MCP**  
  - Type: MCP Trigger node  
  - Configuration: Webhook entry point for AI contact queries  

- **Google Contacts**  
  - Type: MCP Client node  

- **Get Contacts**  
  - Type: Google Contacts Tool node  
  - Configuration: Retrieves contacts filtered by query with options for raw data, name, and email fields  
  - Credentials: Google Contacts OAuth2  
  - Edge cases: Large contact lists, query failures  

---

#### 1.9 WhatsApp Response (Text and Audio) Sending

**Overview:**  
Sends AI-generated replies back to the user via WhatsApp in either text or audio format.

**Nodes Involved:**  
- Set Reply Message  
- Send message (WhatsApp)  
- Convert text to speech (ElevenLabs)  
- Send Audio (WhatsApp)  

**Node Details:**  

- **Set Reply Message**  
  - Type: Set node, sets the message text to reply  

- **Send message**  
  - Type: WhatsApp node  
  - Configuration: Sends text response to the sender's WhatsApp ID  
  - Credentials: WhatsApp API  

- **Convert text to speech**  
  - Type: ElevenLabs text-to-speech node  
  - Configuration: Converts reply text to audio with chosen voice ("Devi - Clear Hindi pronunciation")  
  - Credentials: ElevenLabs API  

- **Send Audio**  
  - Type: WhatsApp node  
  - Configuration: Sends audio message to WhatsApp user  
  - Credentials: WhatsApp API  

- Edge cases: WhatsApp API limits, audio format compatibility, message delivery failures  

---

#### 1.10 Utility Nodes (Memory, Filters, Switches)

**Overview:**  
Supporting nodes for session memory, filtering users, and routing messages based on type.

**Nodes Involved:**  
- Simple Memory  
- Only allow me  
- Switch (Check Text or Audio)  
- Think (Tool Think)  

These nodes have been described above in their respective blocks.

---

### 3. Summary Table

| Node Name              | Node Type                                 | Functional Role                             | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                  |
|------------------------|-------------------------------------------|--------------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| WhatsApp Trigger       | WhatsApp Trigger                          | Entry webhook for incoming WhatsApp messages | -                            | Only allow me                 |                                                                                              |
| Only allow me          | Filter                                   | Allows only messages from authorized user   | WhatsApp Trigger             | Switch (Check Text or Audio)  |                                                                                              |
| Switch (Check Text or Audio) | Switch                             | Routes messages based on type (text/audio) | Only allow me                | Jarvis (Text), Get Media URL (Audio) |                                                                                          |
| Get Media URL          | WhatsApp API                             | Gets media URL for audio messages            | Switch (Audio branch)        | Download Audio                |                                                                                              |
| Download Audio         | HTTP Request                            | Downloads audio from media URL                | Get Media URL                | Transcribe audio or video     |                                                                                              |
| Transcribe audio or video | ElevenLabs speech-to-text             | Converts audio to text                        | Download Audio               | Jarvis                       |                                                                                              |
| Simple Memory          | Langchain Memory Buffer                  | Maintains conversational context            | Jarvis                       | Jarvis                       |                                                                                              |
| OpenAI Chat Model      | Langchain OpenAI Model                   | Generates AI responses                        | Jarvis                       | Jarvis                       |                                                                                              |
| Jarvis                 | Langchain Agent                         | AI assistant core logic                       | Simple Memory, OpenAI Model  | Set Reply Message            | ## Jarvis 🤖 Your AI-powered personal assistant. Orchestrates tasks, calendar, emails, contacts & expenses. Uses memory + OpenAI model for smart decisions. Sends results back to Telegram. |
| Set Reply Message      | Set                                     | Prepares reply message to send                | Jarvis                       | Check Text or Audio           |                                                                                              |
| Send message           | WhatsApp                                 | Sends text reply to user                      | Check Text or Audio (Text)   | -                            |                                                                                              |
| Convert text to speech | ElevenLabs text-to-speech                 | Converts reply text to audio                   | Check Text or Audio (Audio)  | Send Audio                   |                                                                                              |
| Send Audio             | WhatsApp                                 | Sends audio reply to user                      | Convert text to speech       | -                            |                                                                                              |
| Gmail MCP Server       | Langchain MCP Trigger                    | Handles AI Gmail requests                      | Get Emails, Send Email, etc. | Gmail MCP                    | ## Gmail MCP 📧 Full email management. Send & draft messages, Reply, label, and fetch emails. |
| Gmail MCP              | Langchain MCP Client                     | Executes Gmail operations for AI              | Gmail MCP Server             | Jarvis                       |                                                                                              |
| Send Email             | Gmail Tool                              | Sends new emails                              | Gmail MCP                   | -                            |                                                                                              |
| Reply to an Email      | Gmail Tool                              | Sends email replies                           | Gmail MCP                   | -                            |                                                                                              |
| Draft Email            | Gmail Tool                              | Creates email drafts                          | Gmail MCP                   | -                            |                                                                                              |
| Draft Email Reply      | Gmail Tool                              | Creates draft replies                         | Gmail MCP                   | -                            |                                                                                              |
| Get Emails             | Gmail Tool                              | Fetches emails                               | Gmail MCP                   | -                            |                                                                                              |
| Add Label to Email     | Gmail Tool                              | Adds labels to emails                         | Gmail MCP                   | -                            |                                                                                              |
| Get Labels             | Gmail Tool                              | Gets label list                              | Gmail MCP                   | -                            |                                                                                              |
| Calendar MCP Server    | Langchain MCP Trigger                    | Handles AI calendar requests                  | Get all Events, Create Event | Calendar MCP                 | ## Calendar MCP 📅 Your scheduling hub. Check availability, Create, reschedule, or delete events. |
| Calendar MCP           | Langchain MCP Client                     | Executes calendar operations                   | Calendar MCP Server          | Jarvis                       |                                                                                              |
| Check Availability     | Google Calendar Tool                    | Checks free/busy slots                        | Calendar MCP                | -                            |                                                                                              |
| Get all Events         | Google Calendar Tool                    | Retrieves events                              | Calendar MCP                | -                            |                                                                                              |
| Delete Calendar Event  | Google Calendar Tool                    | Deletes events                                | Calendar MCP                | -                            |                                                                                              |
| Reschedule Event       | Google Calendar Tool                    | Updates event times and attendees             | Calendar MCP                | -                            |                                                                                              |
| Get Event              | Google Calendar Tool                    | Retrieves a single event                       | Calendar MCP                | -                            |                                                                                              |
| Create an event        | Google Calendar Tool                    | Creates new events                            | Calendar MCP                | -                            |                                                                                              |
| Task Manager MCP       | Langchain MCP Trigger                    | Handles AI task management requests           | Create Task, Delete Task, etc. | Google Tasks MCP            | ## Task Manager MCP ✅ Manages to-dos with ease: Create / Complete / Delete tasks, Retrieve individual or bulk tasks. |
| Google Tasks MCP       | Langchain MCP Client                     | Executes Google Tasks operations               | Task Manager MCP            | Jarvis                       |                                                                                              |
| Create a Task          | Google Tasks Tool                      | Creates new tasks                             | Google Tasks MCP            | -                            |                                                                                              |
| Complete a Task        | Google Tasks Tool                      | Marks tasks as completed                       | Google Tasks MCP            | -                            |                                                                                              |
| Delete a Task          | Google Tasks Tool                      | Deletes tasks                                | Google Tasks MCP            | -                            |                                                                                              |
| Get many Tasks         | Google Tasks Tool                      | Retrieves multiple tasks                        | Google Tasks MCP            | -                            |                                                                                              |
| Get a Task             | Google Tasks Tool                      | Retrieves a single task                         | Google Tasks MCP            | -                            |                                                                                              |
| Finance Manager MCP Server | Langchain MCP Trigger               | Handles AI finance spreadsheet requests       | Get all Expenses, Create Expense, Delete Expense | Finance Tracker       | ## Finance Manager MCP 💵 Track personal or business expenses. Create new expenses, Get expense reports, Delete outdated entries. |
| Finance Tracker        | Langchain MCP Client                     | Executes finance Google Sheets operations      | Finance Manager MCP Server  | Jarvis                       |                                                                                              |
| Get all Expenses       | Google Sheets Tool                    | Reads expense data from sheet                   | Finance Tracker             | -                            |                                                                                              |
| Create Expense         | Google Sheets Tool                    | Appends new expense entries                     | Finance Tracker             | -                            |                                                                                              |
| Delete Expense         | Google Sheets Tool                    | Deletes rows from expense sheet                  | Finance Tracker             | -                            |                                                                                              |
| Google Contacts MCP    | Langchain MCP Trigger                    | Handles AI contacts requests                    | Get Contacts                | Google Contacts             | ## Google Contacts MCP 👥 Access and manage your contact list. Fetch contacts for quick communication. |
| Google Contacts        | Langchain MCP Client                     | Executes Google Contacts operations             | Google Contacts MCP         | Jarvis                       |                                                                                              |
| Get Contacts           | Google Contacts Tool                  | Retrieves contact info from Google Contacts     | Google Contacts MCP         | -                            |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node:**  
   - Type: WhatsApp Trigger  
   - Configure webhook for incoming messages, enable messages updates  
   - Connect to "Only allow me" filter node  
   - Set WhatsApp API credentials  

2. **Create "Only allow me" Filter Node:**  
   - Type: Filter  
   - Condition: `$json.messages[0].from` equals `"919920842422"` (your phone number)  
   - Connect output to "Switch (Check Text or Audio)" node  

3. **Create "Switch (Check Text or Audio)" Node:**  
   - Type: Switch  
   - Condition: `$json.messages[0].type` equals "text" or "audio"  
   - Connect "Text" output to "Jarvis" AI processing chain  
   - Connect "Audio" output to "Get Media URL" node  

4. **Create Media Processing Chain:**  
   - **Get Media URL** (WhatsApp API node)  
     - Operation: mediaUrlGet  
     - Parameter: `$json.messages[0].audio.id`  
     - Connect to "Download Audio" node  
   - **Download Audio** (HTTP Request node)  
     - URL from previous node output  
     - Use WhatsApp credentials for authentication  
     - Connect to "Transcribe audio or video" node  
   - **Transcribe audio or video** (ElevenLabs speech-to-text)  
     - Speech to text operation  
     - Use ElevenLabs credentials  
     - Output transcribed text to "Jarvis" node  

5. **Create AI Core Block:**  
   - **Simple Memory** (Langchain memory buffer)  
     - SessionKey: "1"  
     - Connect to "Jarvis" agent  
   - **OpenAI Chat Model** (Langchain OpenAI node)  
     - Model: "gpt-4.1-mini"  
     - Use OpenAI API credentials  
     - Connect to "Jarvis" agent  
   - **Jarvis** (Langchain agent)  
     - Configure with system prompt defining Jarvis personality, capabilities, and operational rules (as per description)  
     - Input: user message (text or transcribed) + memory + OpenAI model  
     - Output to "Set Reply Message" node  

6. **Create "Set Reply Message" Node:**  
   - Type: Set  
   - Assign variable `message` with AI output or error message  
   - Connect to "Check Text or Audio" switch for reply type  

7. **Create Reply Sending Nodes:**  
   - **Send message** (WhatsApp node)  
     - Operation: send text message  
     - Send to `$json.contacts[0].wa_id` from WhatsApp Trigger  
     - Connect from "Check Text or Audio" (Text output)  
   - **Convert text to speech** (ElevenLabs text-to-speech)  
     - Text from "Set Reply Message" node  
     - Voice: "Devi - Clear Hindi pronunciation"  
     - Connect to "Send Audio" node  
   - **Send Audio** (WhatsApp node)  
     - Operation: send audio message  
     - Recipient: `$json.messages[0].from` from WhatsApp Trigger  
     - Connect from "Convert text to speech" node  

8. **Create Gmail MCP Subsystem:**  
   - **Gmail MCP Server** (Langchain MCP Trigger)  
     - Webhook path: unique identifier  
     - Connect to Gmail tools nodes  
   - **Gmail MCP** (Langchain MCP Client)  
     - Endpoint URL: same as Gmail MCP Server webhook URL + "/sse"  
     - Connect to "Jarvis" node  
   - Add Gmail Tool nodes: Send Email, Reply to an Email, Draft Email, Draft Email Reply, Get Emails, Add Label to Email, Get Labels  
   - Configure Gmail OAuth2 credentials for all Gmail Tool nodes  

9. **Create Google Calendar MCP Subsystem:**  
   - **Calendar MCP Server** (MCP Trigger)  
     - Webhook path: unique identifier  
     - Connect to calendar tool nodes  
   - **Calendar MCP** (MCP Client)  
     - Endpoint URL: server webhook + "/sse"  
     - Connect to "Jarvis" node  
   - Add Google Calendar nodes: Check Availability, Get all Events, Delete Calendar Event, Reschedule Event, Get Event, Create an event  
   - Configure Google Calendar OAuth2 credentials  

10. **Create Google Tasks MCP Subsystem:**  
    - **Task Manager MCP** (MCP Trigger)  
      - Webhook path: unique identifier  
      - Connect to Google Tasks nodes  
    - **Google Tasks MCP** (MCP Client)  
      - Endpoint URL: server webhook + "/sse"  
      - Connect to "Jarvis" node  
    - Add Google Tasks nodes: Create a Task, Complete a Task, Delete a Task, Get many Tasks, Get a Task  
    - Configure Google Tasks OAuth2 credentials  

11. **Create Finance MCP Subsystem:**  
    - **Finance Manager MCP Server** (MCP Trigger)  
      - Webhook path: unique identifier  
      - Connect to Google Sheets nodes  
    - **Finance Tracker** (MCP Client)  
      - Endpoint URL: server webhook + "/sse"  
      - Connect to "Jarvis" node  
    - Add Google Sheets nodes: Get all Expenses, Create Expense, Delete Expense  
    - Configure Google Sheets OAuth2 credentials for your Expense Tracker spreadsheet  

12. **Create Google Contacts MCP Subsystem:**  
    - **Google Contacts MCP** (MCP Trigger)  
      - Webhook path: unique identifier  
      - Connect to Google Contacts node  
    - **Google Contacts** (MCP Client)  
      - Endpoint URL: server webhook + "/sse"  
      - Connect to "Jarvis" node  
    - Add Google Contacts node: Get Contacts  
    - Configure Google Contacts OAuth2 credentials  

13. **Link all MCP Clients to Jarvis node as AI tools:**  
    - Connect Gmail MCP, Calendar MCP, Google Tasks MCP, Finance Tracker, Google Contacts MCP client nodes as AI tools input to Jarvis agent  

14. **Test end-to-end with WhatsApp messages:**  
    - Send text or audio messages from authorized WhatsApp number  
    - Confirm AI responses and actions on Gmail, Calendar, Tasks, Finance Sheets, and Contacts  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                             | Context or Link                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Jarvis 🤖 Your AI-powered personal assistant. Orchestrates tasks, calendar, emails, contacts & expenses. Uses memory + OpenAI model for smart decisions. Sends results back to Telegram.                                                                   | Sticky Note1                                                   |
| Gmail MCP 📧 Full email management. Send & draft messages, Reply, label, and fetch emails.                                                                                                                                                                 | Sticky Note4                                                   |
| Finance Manager MCP 💵 Track personal or business expenses. Create new expenses, Get expense reports, Delete outdated entries.                                                                                                                            | Sticky Note5                                                   |
| Google Contacts MCP 👥 Access and manage your contact list. Fetch contacts for quick communication.                                                                                                                                                       | Sticky Note6                                                   |
| Calendar MCP 📅 Your scheduling hub. Check availability, Create, reschedule, or delete events.                                                                                                                                                            | Sticky Note3                                                   |
| Task Manager MCP ✅ Manages to-dos with ease: Create / Complete / Delete tasks, Retrieve individual or bulk tasks.                                                                                                                                         | Sticky Note2                                                   |
| The workflow integrates ElevenLabs API for speech-to-text (transcription) and text-to-speech to enable audio communication over WhatsApp.                                                                                                               | ElevenLabs API integration                                     |
| OpenAI GPT-4.1-mini model powers the AI understanding and response generation with a customized system prompt specifying Jarvis's personality and capabilities.                                                                                            | OpenAI API credentials required                                |
| WhatsApp API nodes require proper credential setup with phone number ID and API keys.                                                                                                                                                                     | WhatsApp API credentials                                       |
| Gmail, Calendar, Tasks, Sheets, and Contacts integrations require OAuth2 credentials with respective scopes enabled.                                                                                                                                       | Google OAuth2 credentials                                      |
| The MCP (Model Context Protocol) Trigger and Client nodes facilitate AI-driven tool invocation and context-aware multi-step workflows across Google services and Sheets.                                                                                 | MCP architecture                                              |
| Ensure all webhook URLs for MCP triggers are publicly accessible and secured.                                                                                                                                                                            | Deployment note                                               |
| Date/time inputs for events and tasks must comply with RFC3339 format and use future dates for scheduling or task deadlines unless otherwise specified.                                                                                                   | Operational rules                                             |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.