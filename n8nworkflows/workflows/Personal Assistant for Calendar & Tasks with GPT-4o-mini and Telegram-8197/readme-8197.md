Personal Assistant for Calendar & Tasks with GPT-4o-mini and Telegram

https://n8nworkflows.xyz/workflows/personal-assistant-for-calendar---tasks-with-gpt-4o-mini-and-telegram-8197


# Personal Assistant for Calendar & Tasks with GPT-4o-mini and Telegram

### 1. Workflow Overview

This workflow implements a **Personal Assistant for Calendar & Tasks** using GPT-4o-mini and Telegram integration. It is designed to manage Google Calendar events and Google Tasks based on natural language commands received via Telegram chat. The workflow handles event creation (single and recurring), event deletion, task creation, updating, completion, deletion, and repositioning.

**Target Use Cases:**
- Scheduling single or recurring calendar events with one or multiple attendees.
- Managing Google Tasks: creating, updating, completing, deleting, and reordering tasks.
- Handling user interactions on Telegram, including greeting, scheduling requests, task management commands, and error reporting.
- Conflict detection for calendar events with suggestions to reschedule or double-book.

**Logical Blocks:**

- **1.1 Input Reception and Preprocessing:** Receives Telegram messages and triggers AI processing.
- **1.2 AI Processing and Decision Making:** Uses OpenAI GPT-4o-mini to interpret user commands and direct actions.
- **1.3 Google Calendar Management:** Creates, updates, retrieves, and deletes calendar events, including conflict checking.
- **1.4 Google Tasks Management:** Handles creation, updating, completion, deletion, and repositioning of tasks.
- **1.5 Output and Error Handling:** Sends responses or error messages back to Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preprocessing

**Overview:**  
This block captures user input from Telegram and initializes the AI agent with session memory to contextualize conversations.

**Nodes Involved:**  
- Telegram Trigger  
- Simple Memory  
- AI Agent

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for incoming Telegram messages (updates of type "message").  
  - *Configuration:* Default webhook with update type message.  
  - *Inputs:* External (Telegram).  
  - *Outputs:* Passes messages to AI Agent.  
  - *Potential Failures:* Telegram API downtime, webhook misconfiguration.

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains conversation context per Telegram chat ID to enable multi-turn dialog.  
  - *Configuration:* Session key set to Telegram chat ID (`{{$('Telegram Trigger').item.json.message.chat.id}}`).  
  - *Inputs:* From Telegram Trigger.  
  - *Outputs:* To AI Agent.  
  - *Edge Cases:* Memory overflow if conversation very long; session key mismatch.

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Processes user text input, uses system prompt to interpret commands, and selects appropriate tools.  
  - *Configuration:*  
    - Text input from Telegram message text.  
    - System prompt defines assistant "Troy" with detailed instructions on task and calendar management, conflict checks, recurrence handling, and output formatting for Telegram.  
    - Timezone set to Asia/Dhaka.  
  - *Inputs:* Text from Telegram Trigger + context from Simple Memory.  
  - *Outputs:* Passes processed output to Send Message node and MCP clients (Google Tasks MCP and Google Calendar MCP).  
  - *Edge Cases:* Model API failures, incorrect parsing, misinterpretation of user input.

---

#### 1.2 AI Processing and Language Model

**Overview:**  
This block integrates the OpenAI GPT-4o-mini chat model to generate intelligent responses and interpret user commands.

**Nodes Involved:**  
- OpenAI Chat Model

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Processes natural language inputs to generate AI responses.  
  - *Configuration:* Uses `gpt-4o-mini` model variant with default options.  
  - *Inputs:* From AI Agent's language model interface.  
  - *Outputs:* Feeds back to AI Agent for final decision and command routing.  
  - *Edge Cases:* API rate limits, network issues, model unavailability.

---

#### 1.3 Google Calendar Management

**Overview:**  
Handles all calendar-related operations, including creating events with different attendee counts, creating recurring events, deleting events, and retrieving event details.

**Nodes Involved:**  
- MCP Server Trigger  
- Create an Event in Google Calendar with 1 Attendee  
- Create an Event in Google Calendar with 2 Attendees  
- Create a weekly recurrent event Event  
- Delete an event Google Calendar  
- Get_Events  
- GetAll_Events  
- Google Calender MCP

**Node Details:**

- **MCP Server Trigger**  
  - *Type:* Langchain MCP Server Trigger  
  - *Role:* Receives AI tool calls related to Google Calendar operations (event creation, deletion, retrieval).  
  - *Configuration:* Webhook path set for calendar MCP calls.  
  - *Inputs:* From AI Agent via MCP Client Tool.  
  - *Outputs:* Dispatches to calendar tool nodes based on AI commands.  
  - *Edge Cases:* Webhook availability, malformed requests.

- **Create an Event in Google Calendar with 1 Attendee**  
  - *Type:* Google Calendar Tool  
  - *Role:* Creates a single calendar event with one attendee.  
  - *Configuration:*  
    - Start and end times extracted from AI variables (e.g. `$fromAI('Start')`, `$fromAI('End')`).  
    - Summary, description, attendee email inferred from AI context.  
    - Event marked transparent (free time).  
  - *Inputs:* MCP Server Trigger.  
  - *Outputs:* Event creation response.  
  - *Edge Cases:* Invalid date/time formats, attendee email missing or invalid, Google API errors.

- **Create an Event in Google Calendar with 2 Attendees**  
  - *Type:* Google Calendar Tool  
  - *Role:* Similar to above but supports two attendees.  
  - *Configuration:* Attendee emails sourced from AI variables for first and second attendee.  
  - *Edge Cases:* Same as one attendee, plus validation of two emails.

- **Create a weekly recurrent event Event**  
  - *Type:* Google Calendar Tool  
  - *Role:* Creates a weekly recurring event with two attendees.  
  - *Configuration:* Recurrence frequency set to "weekly", attendees from AI variables, times from AI.  
  - *Edge Cases:* Conflicts with existing recurring events, recurrence rule errors.

- **Delete an event Google Calendar**  
  - *Type:* Google Calendar Tool (Delete operation)  
  - *Role:* Deletes an existing calendar event identified by event ID from AI.  
  - *Configuration:* Event ID passed via AI variable.  
  - *Edge Cases:* Non-existent event ID, permission errors.

- **Get_Events**  
  - *Type:* Google Calendar Tool (Get operation)  
  - *Role:* Retrieves a specific event by event ID for conflict checking or details.  
  - *Edge Cases:* Invalid ID, API errors.

- **GetAll_Events**  
  - *Type:* Google Calendar Tool (GetAll operation)  
  - *Role:* Retrieves all events in a time range to check for conflicts.  
  - *Inputs:* `timeMin` and `timeMax` from AI variables.  
  - *Edge Cases:* Large result sets, API rate limits.

- **Google Calender MCP**  
  - *Type:* MCP Client Tool  
  - *Role:* Acts as the client interface for AI Agent to call Google Calendar MCP Server Trigger.  
  - *Inputs:* From AI Agent.  
  - *Outputs:* To MCP Server Trigger.  
  - *Edge Cases:* Credential issues, connection errors.

---

#### 1.4 Google Tasks Management

**Overview:**  
Manages Google Tasks operations including creating, updating, completing, deleting, retrieving tasks, and repositioning tasks within a task list.

**Nodes Involved:**  
- MCP Server Trigger1  
- create_task  
- update tasks  
- complete_task  
- delete task  
- get_tasks  
- move_task_position  
- Google Tasks MCP

**Node Details:**

- **MCP Server Trigger1**  
  - *Type:* Langchain MCP Server Trigger  
  - *Role:* Handles AI tool calls related to Google Tasks management.  
  - *Inputs:* From AI Agent via MCP Client Tool.  
  - *Outputs:* Routes to Google Tasks tool nodes.  
  - *Edge Cases:* Same as MCP Server Trigger.

- **create_task**  
  - *Type:* Google Tasks Tool  
  - *Role:* Creates a new task in a specified task list.  
  - *Configuration:*  
    - Task list ID fixed (`MDQ3MTk5NzQzMTc5NDM3NTA4OTU6MDow`).  
    - Title, notes, parent task, due date, completion date sourced from AI variables.  
  - *Edge Cases:* Invalid dates, missing title, invalid parent task ID.

- **update tasks**  
  - *Type:* Google Tasks Tool (Update operation)  
  - *Role:* Updates an existing task’s title, notes, and due date.  
  - *Configuration:* Task ID and fields from AI variables.  
  - *Edge Cases:* Task not found, invalid fields.

- **complete_task**  
  - *Type:* Google Tasks Tool (Update operation)  
  - *Role:* Marks a task as completed by updating status.  
  - *Configuration:* Task ID from AI variables; status set to "completed".  
  - *Edge Cases:* Task already completed or invalid ID.

- **delete task**  
  - *Type:* Google Tasks Tool (Delete operation)  
  - *Role:* Deletes a task by ID.  
  - *Edge Cases:* Task ID invalid or task already deleted.

- **get_tasks**  
  - *Type:* Google Tasks Tool (GetAll operation)  
  - *Role:* Retrieves all tasks (including completed) in a task list for reference.  
  - *Configuration:* Return all controlled by AI variable.  
  - *Edge Cases:* Large task lists, API limits.

- **move_task_position**  
  - *Type:* HTTP Request Tool  
  - *Role:* Moves a task’s position within a task list via Google Tasks API endpoint not directly available in n8n nodes.  
  - *Configuration:*  
    - POST request to `https://tasks.googleapis.com/tasks/v1/lists/{taskListId}/tasks/{taskId}/move`  
    - Query parameters for parent and previous sibling task IDs optionally specified by AI variables.  
    - Uses predefined Google Tasks OAuth2 credentials.  
  - *Edge Cases:* Invalid task IDs, authorization errors, invalid reposition parameters.

- **Google Tasks MCP**  
  - *Type:* MCP Client Tool  
  - *Role:* Client interface for AI Agent to call Google Tasks MCP Server Trigger1.  
  - *Inputs:* From AI Agent.  
  - *Outputs:* To MCP Server Trigger1.  
  - *Edge Cases:* Credential or connection errors.

---

#### 1.5 Output and Error Handling

**Overview:**  
Responsible for sending processed messages or error notifications back to Telegram users.

**Nodes Involved:**  
- Send Message  
- Send Error Message

**Node Details:**

- **Send Message**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends AI-generated responses back to the Telegram chat.  
  - *Configuration:*  
    - Text from AI Agent output JSON field `output`.  
    - Chat ID from Telegram Trigger message.  
    - Attribution disabled.  
  - *Edge Cases:* Telegram API limits, chat ID missing, message formatting errors.  
  - *Error Handling:* On error, workflow continues to Send Error Message node.

- **Send Error Message**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends a generic error message if the main response fails.  
  - *Configuration:*  
    - Fixed text: "We are facing troubles in providing a response right now. Please try again later."  
    - Chat ID from Telegram Trigger message.  
  - *Edge Cases:* Telegram API errors, chat ID missing.

---

### 3. Summary Table

| Node Name                             | Node Type                             | Functional Role                              | Input Node(s)          | Output Node(s)                     | Sticky Note                                                                                                   |
|-------------------------------------|-------------------------------------|----------------------------------------------|------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------|
| Telegram Trigger                    | Telegram Trigger                    | Receives Telegram messages                    | External                | AI Agent                          |                                                                                                               |
| Simple Memory                      | Langchain Memory Buffer Window      | Maintains conversation context                | Telegram Trigger        | AI Agent                          |                                                                                                               |
| AI Agent                          | Langchain Agent                    | Processes input, routes commands               | Telegram Trigger, Simple Memory | Send Message, MCP Client Tools    |                                                                                                               |
| OpenAI Chat Model                 | Langchain OpenAI Chat Model          | AI language processing                         | AI Agent                | AI Agent                          |                                                                                                               |
| Google Calender MCP                  | MCP Client Tool                    | Client interface for Google Calendar MCP      | AI Agent                | MCP Server Trigger                |                                                                                                               |
| MCP Server Trigger                 | MCP Server Trigger                 | Handles Google Calendar tool calls             | Google Calender MCP     | Calendar Tool Nodes               | ## Google Calendar MCP Server<br>1. Add Google Calendar Credentials<br>2. Select the Calendar from the list (your mail) |
| Create an Event in Google Calendar with 1 Attendee | Google Calendar Tool               | Creates single event with 1 attendee           | MCP Server Trigger      |                                   |                                                                                                               |
| Create an Event in Google Calendar with 2 Attendees | Google Calendar Tool               | Creates single event with 2 attendees          | MCP Server Trigger      |                                   |                                                                                                               |
| Create a weekly recurrent event Event | Google Calendar Tool               | Creates weekly recurring event with attendees | MCP Server Trigger      |                                   |                                                                                                               |
| Delete an event Google Calendar    | Google Calendar Tool (Delete)        | Deletes a calendar event                        | MCP Server Trigger      |                                   |                                                                                                               |
| Get_Events                        | Google Calendar Tool (Get)            | Retrieves specific calendar event              | MCP Server Trigger      |                                   |                                                                                                               |
| GetAll_Events                    | Google Calendar Tool (GetAll)          | Retrieves multiple calendar events             | MCP Server Trigger      |                                   |                                                                                                               |
| MCP Server Trigger1               | MCP Server Trigger                 | Handles Google Tasks tool calls                 | Google Tasks MCP        | Tasks Tool Nodes                 | ## Google Tasks MCP Server<br>1. Add your credentials                                                        |
| Google Tasks MCP                  | MCP Client Tool                    | Client interface for Google Tasks MCP           | AI Agent                | MCP Server Trigger1              |                                                                                                               |
| create_task                      | Google Tasks Tool                   | Creates a new task                              | MCP Server Trigger1     |                                   |                                                                                                               |
| update tasks                    | Google Tasks Tool (Update)             | Updates task details                            | MCP Server Trigger1     |                                   |                                                                                                               |
| complete_task                   | Google Tasks Tool (Update)             | Marks task as completed                         | MCP Server Trigger1     |                                   |                                                                                                               |
| delete task                    | Google Tasks Tool (Delete)             | Deletes a task                                 | MCP Server Trigger1     |                                   |                                                                                                               |
| get_tasks                      | Google Tasks Tool (GetAll)             | Retrieves tasks list                            | MCP Server Trigger1     |                                   |                                                                                                               |
| move_task_position             | HTTP Request Tool                   | Moves task position in a list                   | MCP Server Trigger1     |                                   | ## Custom tool<br>Using http tool for using other APIs which are not directly allowed via available tool nodes.|
| Send Message                   | Telegram Node (Send Message)           | Sends AI response to Telegram                   | AI Agent                | Send Error Message (on error)    |                                                                                                               |
| Send Error Message            | Telegram Node (Send Message)           | Sends generic error message                      | Send Message (on error) |                                   |                                                                                                               |
| Sticky Note1                  | Sticky Note                        | Instruction: Add Google Tasks credentials       |                        |                                   | ## Google Tasks MCP Server<br>1. Add your credentials                                                        |
| Sticky Note2                  | Sticky Note                        | Instruction: Add Google Calendar Credentials    |                        |                                   | ## Google Calendar MCP Server<br>1. Add Google Calendar Credentials<br>2. Select the Calendar from the list (your mail) |
| Sticky Note3                  | Sticky Note                        | General instructions for running workflows      |                        |                                   | ## Personal Assistant<br>1. Run all three workflows separately<br>2. Add the MCP server SSE endpoints in the MCP client<br>3. Setup telegram credentials |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Listen to message updates only.  
   - Set webhook ID (auto-generated).  
   - Connect output to Simple Memory and AI Agent.

2. **Create Simple Memory node**  
   - Type: Langchain Memory Buffer Window  
   - Set sessionKey to `{{$('Telegram Trigger').item.json.message.chat.id}}`  
   - Connect input from Telegram Trigger, output to AI Agent.

3. **Create AI Agent node**  
   - Type: Langchain Agent  
   - Set text input to `{{$json.message.text}}` from Telegram Trigger.  
   - Configure the system prompt with detailed assistant instructions (see 2.1 AI Agent details).  
   - Set promptType to "define".  
   - Connect inputs from Telegram Trigger and Simple Memory.  
   - Connect outputs to Send Message, Google Tasks MCP, and Google Calendar MCP.

4. **Create OpenAI Chat Model node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Connect as AI language model input of AI Agent.

5. **Create Google Tasks MCP Client Tool node**  
   - Type: MCP Client Tool  
   - Configure with Google Tasks MCP server SSE endpoint (to be created).  
   - Connect output to MCP Server Trigger1.

6. **Create MCP Server Trigger1 node (Google Tasks MCP Server)**  
   - Type: MCP Server Trigger  
   - Set webhook path (auto-generated).  
   - Connect input from Google Tasks MCP Client Tool.  
   - Connect output to Google Tasks tool nodes (create_task, update tasks, complete_task, delete task, get_tasks, move_task_position).

7. **Create Google Tasks Tool nodes**  
   - create_task: Set operation to create, specify task list ID, map title, notes, parent, dueDate, completed from AI variables.  
   - update tasks: Operation update, specify taskId, update fields (title, notes, dueDate).  
   - complete_task: Operation update, set taskId and status "completed".  
   - delete task: Operation delete, set taskId.  
   - get_tasks: Operation getAll, showCompleted true, returnAll from AI variable.  
   - move_task_position: HTTP Request node, POST method to `https://tasks.googleapis.com/tasks/v1/lists/{taskListId}/tasks/{taskId}/move`, with query parameters for parent and previous task IDs, authenticated with Google Tasks OAuth2 credentials.

8. **Create Google Calendar MCP Client Tool node**  
   - Type: MCP Client Tool  
   - Configure with Google Calendar MCP server SSE endpoint (to be created).  
   - Connect output to MCP Server Trigger.

9. **Create MCP Server Trigger node (Google Calendar MCP Server)**  
   - Type: MCP Server Trigger  
   - Set webhook path (auto-generated).  
   - Connect input from Google Calendar MCP Client Tool.  
   - Connect output to calendar event nodes.

10. **Create Google Calendar Tool nodes**  
    - Create an Event with 1 Attendee: Operation create, map start, end, summary, description, attendee from AI variables.  
    - Create an Event with 2 Attendees: Same as above, but with two attendees.  
    - Create a weekly recurrent event: Same as above but set recurrence to weekly.  
    - Delete an event: Operation delete, eventId from AI variables.  
    - Get_Events: Operation get, eventId from AI variables.  
    - GetAll_Events: Operation getAll, timeMin, timeMax from AI variables, returnAll flag.

11. **Create Send Message node**  
    - Type: Telegram node (Send Message)  
    - Text from `{{$json.output}}` from AI Agent.  
    - Chat ID from Telegram Trigger message chat ID.  
    - Connect input from AI Agent output.  
    - On error, connect to Send Error Message.

12. **Create Send Error Message node**  
    - Type: Telegram node (Send Message)  
    - Fixed text: "We are facing troubles in providing a response right now. Please try again later."  
    - Chat ID from Telegram Trigger message chat ID.

13. **Add Sticky Notes**  
    - Sticky Note1: Instructions for Google Tasks MCP credentials.  
    - Sticky Note2: Instructions for Google Calendar MCP credentials and calendar selection.  
    - Sticky Note3: General instructions for running workflows and setup.

14. **Credential Setup**  
    - Add Google Tasks OAuth2 credentials with appropriate scopes for tasks API.  
    - Add Google Calendar OAuth2 credentials with calendar API scopes.  
    - Add Telegram API credentials with bot token.

15. **Final Connections**  
    - Connect Telegram Trigger main output to AI Agent main input.  
    - AI Agent connects to MCP Client Tools for Google Calendar and Google Tasks.  
    - MCP Client Tools connect to respective MCP Server Triggers.  
    - MCP Server Triggers connect to their respective API tool nodes.  
    - AI Agent output connects to Send Message node, which on error connects to Send Error Message.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Personal Assistant "Troy" uses Bangladesh Standard Time (Asia/Dhaka) for all scheduling and time-related operations.    | System prompt in AI Agent node.                                                                  |
| MCP (Modular Chat Plugin) architecture allows decoupling AI decision-making and API tool executions via SSE endpoints. | See MCP Client Tool and MCP Server Trigger nodes.                                                |
| Telegram Bot setup requires valid bot token and webhook configuration for receiving messages and sending responses.    | Telegram Trigger and Telegram Send nodes.                                                        |
| Custom HTTP Request node used to access Google Tasks API endpoint for task repositioning, not natively supported by n8n.| Sticky Note on move_task_position node.                                                          |
| Conflict detection logic for calendar events is implemented in AI instructions, mandating pre-checks before event creation. | System prompt in AI Agent node.                                                                  |
| Workflow designed for multi-turn conversations with memory buffer to maintain context per user chat session.           | Simple Memory node configuration.                                                                |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.