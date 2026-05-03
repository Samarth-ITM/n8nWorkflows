Email-to-Task Automation with Mistral AI (Gmail/Outlook to Google Tasks/Microsoft To Do)

https://n8nworkflows.xyz/workflows/email-to-task-automation-with-mistral-ai--gmail-outlook-to-google-tasks-microsoft-to-do--10186


# Email-to-Task Automation with Mistral AI (Gmail/Outlook to Google Tasks/Microsoft To Do)

### 1. Workflow Overview

This workflow automates the conversion of actionable emails from Gmail and Outlook into structured tasks within Google Tasks or Microsoft To Do, leveraging Mistral AI-powered agents for natural language processing and task orchestration. It is designed for personal productivity enhancement by reading recent emails, extracting task-relevant information, and managing tasks in the appropriate task management system based on the target platform indicated.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Scheduled triggers fetch recent emails from Gmail and Microsoft Outlook, then store and normalize email data into a central Data Table.
- **1.2 Data Preparation:** Processes raw email content (HTML to plain text) and upserts parsed emails into the data table for downstream AI processing.
- **1.3 AI Task Orchestration:** Main orchestration agent analyzes the collected task data rows and delegates task management to specialized AI sub-agents.
- **1.4 AI Sub-Agents for Task Management:** Separate AI agents dedicated to Google Tasks and Microsoft To Do handle specific API operations to create or update tasks based on AI-extracted instructions.
- **1.5 Task API Operations:** Nodes that execute actual task creation, updating, deletion, or retrieval for Google Tasks and Microsoft To Do, called exclusively by the AI sub-agents.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow on a schedule and retrieves emails from Gmail and Outlook received within the last day.

**Nodes Involved:**  
- Schedule Trigger  
- Get many messages (Gmail)  
- Get many messages1 (Microsoft Outlook)  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution on a recurring schedule (default once per day).  
  - Configuration: Interval set to run daily (default).  
  - Inputs: None  
  - Outputs: Triggers both email retrieval nodes in parallel.  
  - Edge cases: Misconfiguration may cause workflow to not trigger or trigger too frequently.  

- **Get many messages (Gmail)**  
  - Type: Gmail node (getAll operation)  
  - Role: Fetch all Gmail messages received after yesterday (last 24 hours).  
  - Configuration: Filter `receivedAfter` set dynamically to one day ago using expression `$today.minus({days: 1}).toISO()`.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: List of Gmail messages with full payload.  
  - Credentials: Gmail OAuth2  
  - Edge cases: Authentication failures, API rate limits, empty inbox or no new emails.  

- **Get many messages1 (Microsoft Outlook)**  
  - Type: Microsoft Outlook node (getAll operation)  
  - Role: Fetch all Outlook messages received after yesterday.  
  - Configuration: Filters to retrieve messages with specified fields; `receivedAfter` set dynamically to one day ago.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: List of Outlook messages with selected fields.  
  - Credentials: Microsoft Outlook OAuth2  
  - Edge cases: Authentication failures, API limits, empty inbox, or missing fields.  

---

#### 1.2 Data Preparation

**Overview:**  
Transforms Outlook email HTML content into plain text and upserts Gmail and Outlook emails into a central Data Table for further AI processing.

**Nodes Involved:**  
- Code in JavaScript  
- Upsert row(s) (for Gmail)  
- Upsert row(s)1 (for Outlook)  
- Merge  

**Node Details:**

- **Code in JavaScript**  
  - Type: Code node (JavaScript)  
  - Role: Converts Outlook email HTML body content into plain text.  
  - Configuration: Runs once per item; uses regex to strip HTML tags from `$json.body.content`.  
  - Inputs: Output from Get many messages1 (Outlook)  
  - Outputs: Modified messages with plain text body content.  
  - Edge cases: Malformed HTML could cause incorrect text extraction; empty body fields.  

- **Upsert row(s) (Gmail)**  
  - Type: Data Table node (upsert operation)  
  - Role: Inserts or updates Gmail email details into the Data Table "To Do Database".  
  - Configuration: Maps email headers and content fields into table columns (To, From, Subject, Body, Date, Email ID). Uses `id` as matching key.  
  - Inputs: Output from Get many messages (Gmail)  
  - Outputs: Upserted data table entries.  
  - Edge cases: Duplicate emails, missing headers, network or API errors.  

- **Upsert row(s)1 (Outlook)**  
  - Type: Data Table node (upsert operation)  
  - Role: Inserts or updates Outlook email details into the Data Table "To Do Database".  
  - Configuration: Maps Outlook message fields like `toRecipients[0].emailAddress.address`, `sender.emailAddress.address`, `subject`, `body.content` (already plain text), `sentDateTime`, and `conversationId` as matching key.  
  - Inputs: Output from Code in JavaScript  
  - Outputs: Upserted data table entries.  
  - Edge cases: Similar to Gmail upsert, plus ensuring conversion from HTML to text succeeded.  

- **Merge**  
  - Type: Merge node (default mode)  
  - Role: Combines Gmail and Outlook data stream outputs after upsert operations.  
  - Inputs: Outputs from Upsert row(s) and Upsert row(s)1  
  - Outputs: Merged data stream for downstream processing.  
  - Edge cases: Mismatched data structures could cause merge errors.  

---

#### 1.3 AI Task Orchestration

**Overview:**  
This block acts as the main AI orchestrator, reading batches of tasks from the Data Table and routing them to the appropriate AI sub-agent based on the target system (Google Tasks or Microsoft To Do).

**Nodes Involved:**  
- Get row(s) in Data table  
- Tasks AI Agent (Main Task Orchestration Agent)  
- Simple Memory  

**Node Details:**

- **Get row(s) in Data table**  
  - Type: Data Table node (get operation)  
  - Role: Retrieves up to 5 rows (tasks) from Data Table "To Do Database" for processing, enforcing batch size.  
  - Configuration: Limit set to 5 to control batch processing size.  
  - Inputs: Output from Merge node  
  - Outputs: Data rows representing actionable tasks.  
  - Edge cases: Empty data table, unexpected data format.  

- **Tasks AI Agent**  
  - Type: LangChain Agent node (agent)  
  - Role: Main orchestration AI agent responsible for parsing task rows and routing each to the correct sub-agent (Google Tasks Agent or Microsoft To Do Agent).  
  - Configuration:  
    - System message defines role, batch size limit (5 tasks), 5-second delay requirement after each tool call, and strict filtering for owner "Jordan Hoyle".  
    - Error handling enabled to continue on errors.  
  - Inputs: Rows from Get row(s) in Data table  
  - Outputs: Aggregated execution logs from sub-agents.  
  - Edge cases: AI prompt mis-interpretation, API rate limiting, empty or malformed task rows.  
  - Notes: This node delegates all direct task API calls to sub-agents; it does not perform API operations itself.  

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains a context window of 10 recent interactions for the main AI agent to improve conversational coherence and state.  
  - Configuration: Uses sessionKey "1" and custom sessionIdType.  
  - Inputs: Connected to Tasks AI Agent as ai_memory.  
  - Outputs: Provides memory context for AI calls.  
  - Edge cases: Memory overflow or state loss in long sessions.  

---

#### 1.4 AI Sub-Agents for Task Management

**Overview:**  
Separate AI agents specialized for Google Tasks and Microsoft To Do handle task-specific API operations, ensuring tasks are created or updated correctly based on the input from the main orchestrator.

**Nodes Involved:**  
- Google Tasks Agent  
- Microsoft To Do Agent  
- Simple Memory1 (Microsoft To Do Agent memory)  
- Simple Memory2 (Google Tasks Agent memory)  
- Mistral Cloud Chat Model1 (Google Tasks Agent LM)  
- Mistral Cloud Chat Model2 (Microsoft To Do Agent LM)  

**Node Details:**

- **Google Tasks Agent**  
  - Type: LangChain Agent Tool  
  - Role: Processes a single task object for Google Tasks; decides whether to GET, UPDATE, or CREATE tasks based on task presence.  
  - Configuration:  
    - Input protocol expects structured task object with Title, Due Date, Notes, and Target System (Google Tasks).  
    - Failsafe returns "No action required" if input is empty or 'no task identified'.  
    - Enforces action only for tasks explicitly for "Jordan Hoyle".  
    - Prohibits DELETE operations to prevent accidental removals.  
    - Returns only logs of CREATE/UPDATE actions or failsafe messages—no conversational output.  
  - Inputs: From Tasks AI Agent tool calls.  
  - Outputs: Logs passed back to main orchestrator.  
  - Edge cases: Task title conflicts, API errors, missing or malformed input data.  

- **Microsoft To Do Agent**  
  - Type: LangChain Agent Tool  
  - Role: Similar to Google Tasks Agent but for Microsoft To Do tasks, handling GET, UPDATE, CREATE operations.  
  - Configuration: Same protocols and constraints as Google Tasks Agent, but targeting Microsoft To Do platform.  
  - Inputs: From Tasks AI Agent tool calls.  
  - Outputs: Logs of actions or failsafe messages.  
  - Edge cases: Same as Google Tasks Agent but for Microsoft To Do API.  

- **Simple Memory1 & Simple Memory2**  
  - Type: LangChain Memory Buffer Window  
  - Role: Provide 10-step context window memory for Microsoft To Do Agent (Simple Memory1) and Google Tasks Agent (Simple Memory2), enhancing conversation and task state retention.  
  - Inputs/Outputs: Connected as ai_memory to respective agents.  

- **Mistral Cloud Chat Model1 & 2**  
  - Type: LangChain Language Model (Mistral Cloud)  
  - Role: Provide large language model capabilities for Google Tasks Agent and Microsoft To Do Agent respectively.  
  - Configuration: Model set to "mistral-large-latest".  
  - Inputs: Connected as ai_languageModel to respective agents.  
  - Edge cases: API rate limits, token size limits, network disruptions.  

---

#### 1.5 Task API Operations

**Overview:**  
These nodes perform the actual task creation, updating, deletion, and retrieval API calls for Google Tasks and Microsoft To Do. They are invoked only by the respective AI sub-agents and never directly by the main workflow.

**Nodes Involved:**  
- Google Tasks: Create task1, Update task1, Delete task1, Get many tasks1, Get completed task1  
- Microsoft To Do: Create task, Update task, Delete task, Get Many Tasks  

**Node Details:**

- **Google Tasks Nodes (Create task1, Update task1, Delete task1, Get many tasks1, Get completed task1)**  
  - Type: Google Tasks Tool nodes  
  - Role: Execute task CRUD operations and retrieval on Google Tasks, targeted at a specific task list (ID: RmhtQ0kzY1ltOTJvdGxNSA).  
  - Configuration:  
    - Create and Update use dynamic fields from AI overrides for title, notes, due date.  
    - Delete uses Task_ID from AI overrides.  
    - Get many tasks retrieves all tasks, including hidden and completed if specified.  
  - Inputs: Invoked internally from Google Tasks Agent.  
  - Outputs: API responses with task data or confirmation.  
  - Credentials: Google Tasks OAuth2  
  - Edge cases: Authorization failure, invalid task IDs, API rate limiting, invalid date formats.  
  - Restrictions: Delete operation is prohibited by AI agent policy but node remains defined.  

- **Microsoft To Do Nodes (Create task, Update task, Delete task, Get Many Tasks)**  
  - Type: Microsoft To Do Tool nodes  
  - Role: Execute task CRUD operations and retrieval on Microsoft To Do, targeted at a specific task list ID (long GUID).  
  - Configuration:  
    - Create and Update use AI override fields for title and notes.  
    - Delete uses Task_ID from AI overrides.  
    - Get Many Tasks returns all tasks with returnAll flag from AI overrides.  
  - Inputs: Invoked internally from Microsoft To Do Agent.  
  - Outputs: API responses with task data or confirmation.  
  - Credentials: Microsoft To Do OAuth2  
  - Edge cases: Permissions issues, invalid task IDs, API errors, rate limits.  
  - Restrictions: Delete operation prohibited by AI agent policy but node remains defined.  

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                                                   | Input Node(s)                  | Output Node(s)                        | Sticky Note                                                                                                                      |
|-------------------------|-----------------------------------|-----------------------------------------------------------------|-------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                  | Starts the workflow on schedule                                  | None                          | Get many messages, Get many messages1 | "Schedule your workflow for whatever timeframe makes sense. The standard set is once per day."                                 |
| Get many messages       | Gmail Node                       | Fetch Gmail emails from last day                                | Schedule Trigger              | Upsert row(s)                       |                                                                                                                                 |
| Get many messages1      | Microsoft Outlook Node           | Fetch Outlook emails from last day                              | Schedule Trigger              | Code in JavaScript                  |                                                                                                                                 |
| Code in JavaScript      | Code Node                       | Convert Outlook email HTML body to plain text                   | Get many messages1            | Upsert row(s)1                     | "The main purpose of this section is to avoid token limits or API rate limits. We will break down the emails to necessary info." |
| Upsert row(s)           | Data Table Node                 | Upsert Gmail email data into Data Table                         | Get many messages             | Merge                             |                                                                                                                                 |
| Upsert row(s)1          | Data Table Node                 | Upsert Outlook email data into Data Table                       | Code in JavaScript            | Merge                             |                                                                                                                                 |
| Merge                   | Merge Node                     | Combine Gmail and Outlook email data streams                    | Upsert row(s), Upsert row(s)1| Get row(s) in Data table           |                                                                                                                                 |
| Get row(s) in Data table| Data Table Node                 | Retrieve batch of tasks (max 5) from Data Table                 | Merge                        | Tasks AI Agent                    |                                                                                                                                 |
| Tasks AI Agent          | LangChain Agent                | Main AI agent orchestrating task routing                        | Get row(s) in Data table      | Google Tasks Agent, Microsoft To Do Agent | "Review the prompt in the Tasks AI Agent. Change Owners Name to your name or the name of the person that tasks will be managed."|
| Simple Memory           | LangChain Memory Buffer Window | Provide memory context for Tasks AI Agent                       |                               | Tasks AI Agent                    |                                                                                                                                 |
| Google Tasks Agent      | LangChain Agent Tool           | AI sub-agent for Google Tasks task management                   | Tasks AI Agent               | Create task1, Update task1, Get many tasks1 |                                                                                                                                 |
| Simple Memory2          | LangChain Memory Buffer Window | Provide memory context for Google Tasks Agent                   |                               | Google Tasks Agent               |                                                                                                                                 |
| Mistral Cloud Chat Model1| LangChain Language Model       | Language model for Google Tasks Agent                           |                               | Google Tasks Agent               |                                                                                                                                 |
| Microsoft To Do Agent   | LangChain Agent Tool           | AI sub-agent for Microsoft To Do task management                | Tasks AI Agent               | Create task, Update task, Get Many Tasks |                                                                                                                                 |
| Simple Memory1          | LangChain Memory Buffer Window | Provide memory context for Microsoft To Do Agent               |                               | Microsoft To Do Agent           |                                                                                                                                 |
| Mistral Cloud Chat Model2| LangChain Language Model       | Language model for Microsoft To Do Agent                        |                               | Microsoft To Do Agent           |                                                                                                                                 |
| Create task1            | Google Tasks Tool              | Create new Google Task                                          | Google Tasks Agent           |                                 | "Change the Google Tasks and Microsoft To Do list to your personal tasks lists."                                               |
| Update task1            | Google Tasks Tool              | Update existing Google Task                                     | Google Tasks Agent           |                                 |                                                                                                                                 |
| Delete task1            | Google Tasks Tool              | Delete Google Task (Prohibited by AI policy)                   | Google Tasks Agent           |                                 |                                                                                                                                 |
| Get many tasks1         | Google Tasks Tool              | Retrieve all Google Tasks                                      | Google Tasks Agent           |                                 |                                                                                                                                 |
| Get completed task1     | Google Tasks Tool              | Retrieve all completed Google Tasks                            | Google Tasks Agent           |                                 |                                                                                                                                 |
| Create task             | Microsoft To Do Tool           | Create new Microsoft To Do Task                                | Microsoft To Do Agent        |                                 |                                                                                                                                 |
| Update task             | Microsoft To Do Tool           | Update existing Microsoft To Do Task                           | Microsoft To Do Agent        |                                 |                                                                                                                                 |
| Delete task             | Microsoft To Do Tool           | Delete Microsoft To Do Task (Prohibited by AI policy)          | Microsoft To Do Agent        |                                 |                                                                                                                                 |
| Get Many Tasks          | Microsoft To Do Tool           | Retrieve all Microsoft To Do Tasks                             | Microsoft To Do Agent        |                                 |                                                                                                                                 |
| Sticky Note             | Sticky Note                   | Instruction: Schedule workflow frequency                        | None                         | None                            | "Schedule your workflow for whatever timeframe makes sense. The standard set is once per day."                                 |
| Sticky Note4            | Sticky Note                   | Reminder to update owner name in main AI agent prompt          | None                         | None                            | "Review the prompt in the Tasks AI Agent. Change Owners Name to your name or the name of the person that tasks will be managed."|
| Sticky Note5            | Sticky Note                   | Reminder to update personal task list IDs                      | None                         | None                            | "Change the Google Tasks and Microsoft To Do list to your personal tasks lists."                                               |
| Sticky Note6            | Sticky Note                   | Explanation of email parsing section                            | None                         | None                            | "The main purpose of this section is to avoid token limits or API rate limits. We will break down the emails to the necessary information." |
| Sticky Note7            | Sticky Note                   | High-level description of workflow purpose                      | None                         | None                            | "AI-Powered Task Orchestrator (Gmail/Outlook to Google Tasks/Microsoft To Do)\nAutomate your personal productivity by transforming actionable emails from Gmail and Outlook into structured tasks within Google Tasks or Microsoft To Do, all managed by a multi-agent AI system." |
| Sticky Note8            | Sticky Note                   | Future improvements suggestions                                | None                         | None                            | "Add or Remove email services as required. You can copy the AI Agent Tool and update it to the necessary email service.\nAdd summary email from the \"Success\" block on the main AI Agent." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to run once per day (default interval).  
   - This node starts the workflow automatically on schedule.  

2. **Add Gmail node (Get many messages):**  
   - Operation: getAll  
   - Filters: `receivedAfter` set to `={{ $today.minus({days:1}).toISO() }}`  
   - Return All: true  
   - Credentials: Configure with Gmail OAuth2 credentials.  
   - Connect output from Schedule Trigger.  

3. **Add Microsoft Outlook node (Get many messages1):**  
   - Operation: getAll  
   - Filters: `receivedAfter` set to `={{ $today.minus({days:1}).toISO() }}`  
   - Fields to retrieve: body, sender, subject, sentDateTime, toRecipients, conversationId  
   - Return All: true  
   - Credentials: Configure with Microsoft Outlook OAuth2 credentials.  
   - Connect output from Schedule Trigger.  

4. **Add Code node (JavaScript):**  
   - Mode: runOnceForEachItem  
   - Code: Strip HTML tags from Outlook email body content using regex.  
   - Input: Connect from Microsoft Outlook node.  

5. **Add two Data Table nodes for upserting emails:**  
   - **Upsert row(s):** For Gmail emails  
     - Data Table ID: Use your "To Do Database" table ID  
     - Operation: upsert  
     - Matching column: Email ID (using Gmail message id)  
     - Map columns: To, From, Date, Subject, Body, Email_iD from Gmail fields  
     - Input: Connect from Gmail node.  
   - **Upsert row(s)1:** For Outlook emails  
     - Same Data Table ID  
     - Operation: upsert  
     - Matching column: Email ID (using Outlook conversationId)  
     - Map columns: To, From, Date, Subject, Body, Email_iD from Outlook fields (plain text body from Code node)  
     - Input: Connect from Code node.  

6. **Add Merge node:**  
   - Connect both Data Table upsert nodes outputs to the Merge node.  
   - Purpose: Combine the two email data streams.  

7. **Add Data Table node (Get row(s) in Data table):**  
   - Operation: get  
   - Limit: 5 (batch size)  
   - Input: Connect from Merge node.  

8. **Add LangChain Agent node (Tasks AI Agent):**  
   - Role: Main orchestration agent.  
   - System prompt: Define role to route tasks to Google or Microsoft agents, enforce owner filter ("Jordan Hoyle"), batch size, and delay.  
   - Connect input from Get row(s) in Data table node.  

9. **Add LangChain Memory Buffer Window node (Simple Memory):**  
   - Session key: "1"  
   - Context window length: 10  
   - Connect as ai_memory input to Tasks AI Agent.  

10. **Add LangChain Agent Tool node (Google Tasks Agent):**  
    - Role: Process single Google Tasks task object; handle GET, UPDATE, CREATE operations; prohibit DELETE.  
    - Connect ai_tool input from Tasks AI Agent.  

11. **Add LangChain Agent Tool node (Microsoft To Do Agent):**  
    - Role: Process single Microsoft To Do task object; handle GET, UPDATE, CREATE operations; prohibit DELETE.  
    - Connect ai_tool input from Tasks AI Agent.  

12. **Add LangChain Memory Buffer Window nodes for sub-agents:**  
    - Simple Memory2: Session key "3" for Google Tasks Agent memory.  
    - Simple Memory1: Session key "2" for Microsoft To Do Agent memory.  
    - Connect respective memories as ai_memory inputs to corresponding sub-agents.  

13. **Add LangChain Language Model nodes:**  
    - Mistral Cloud Chat Model1 for Google Tasks Agent.  
    - Mistral Cloud Chat Model2 for Microsoft To Do Agent.  
    - Configure both with `mistral-large-latest` model.  
    - Connect as ai_languageModel inputs to respective agents.  
    - Provide Mistral Cloud API credentials.  

14. **Add Google Tasks Tool nodes:**  
    - Create task1: Create task operation with dynamic title, notes, dueDate from AI overrides.  
    - Update task1: Update task operation with Task_ID and updated fields.  
    - Delete task1: Delete task operation with Task_ID (not used by AI agents).  
    - Get many tasks1: Get all tasks operation with options showHidden and showCompleted true.  
    - Get completed task1: Get all tasks operation for completed tasks.  
    - Provide Google Tasks OAuth2 credentials.  
    - Connect these nodes as ai_tool inputs to Google Tasks Agent.  

15. **Add Microsoft To Do Tool nodes:**  
    - Create task: Create task operation with dynamic title and notes from AI overrides.  
    - Update task: Update task operation with Task_ID and updated fields.  
    - Delete task: Delete task operation with Task_ID (not used by AI agents).  
    - Get Many Tasks: Get all tasks operation.  
    - Provide Microsoft To Do OAuth2 credentials.  
    - Connect these nodes as ai_tool inputs to Microsoft To Do Agent.  

16. **Add Sticky Notes:**  
    - Add instructional sticky notes for scheduling, owner name update, task list update, and future enhancements.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| The workflow is designed to process emails for the owner "Jordan Hoyle"; update owner name in the main AI agent prompt to personalize.                                | Sticky Note4                                                                                                         |
| Update the Google Tasks and Microsoft To Do list IDs in the respective API operation nodes to match your personal task lists.                                         | Sticky Note5                                                                                                         |
| The email parsing section (HTML to plain text conversion) is intended to reduce token use and avoid API rate limits.                                                  | Sticky Note6                                                                                                         |
| Future improvements: Add or remove email services by copying and adjusting AI Agent Tools; add summary email notifications from the main AI agent's success block.    | Sticky Note8                                                                                                         |
| This workflow automates task creation from emails for personal productivity using a multi-agent AI system with Mistral Cloud LLM integration.                         | Sticky Note7                                                                                                         |
| Schedule the workflow to run once daily or as desired to keep task lists up to date with recent emails.                                                                | Sticky Note                                                                                                          |

---

This comprehensive documentation enables advanced users and AI agents to fully understand, reproduce, and extend the Email-to-Task Automation workflow integrating Gmail, Outlook, Google Tasks, Microsoft To Do, and Mistral AI agents.