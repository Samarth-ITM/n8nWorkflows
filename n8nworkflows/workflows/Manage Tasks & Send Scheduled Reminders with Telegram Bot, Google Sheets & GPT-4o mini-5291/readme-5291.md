Manage Tasks & Send Scheduled Reminders with Telegram Bot, Google Sheets & GPT-4o mini

https://n8nworkflows.xyz/workflows/manage-tasks---send-scheduled-reminders-with-telegram-bot--google-sheets---gpt-4o-mini-5291


# Manage Tasks & Send Scheduled Reminders with Telegram Bot, Google Sheets & GPT-4o mini

### 1. Workflow Overview

This workflow facilitates task management and scheduled reminders through a Telegram bot integrated with Google Sheets and GPT-4o models. It enables users to add, update, list, complete, and delete tasks via Telegram messages. All task data is synchronized with Google Sheets to maintain a persistent and up-to-date todo list. Additionally, the workflow sends daily reminders for pending tasks that are due and sends a friendly summary of completed and upcoming tasks on a scheduled basis.

The workflow logic is divided into two main functional blocks:

- **1.1 Telegram Interaction & Task Management**  
  Handles incoming Telegram messages, processes user commands using an AI agent, syncs data with Google Sheets, and replies to the user.

- **1.2 Scheduled Task Reminders & Summaries**  
  Runs daily at a fixed time, reads tasks from Google Sheets, filters for due reminders, sends Telegram notifications for each, and provides a summarized update message.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Interaction & Task Management

**Overview:**  
This block listens for user messages on Telegram, interprets user intents via an AI agent powered by GPT-4o, interacts with Google Sheets to manage tasks accordingly, and replies back through Telegram.

**Nodes Involved:**  
- Telegram Trigger  
- AI Agent  
- OpenAI Chat Model  
- Google Sheets (multiple variants)  
- Telegram  
- Google Sheets7  
- If  
- Loop Over Items  
- Google Sheets8  
- Telegram3  

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Trigger  
  - *Role:* Entry point receiving Telegram messages with user commands.  
  - *Config:* Listens for "message" updates only; uses Telegram API credentials.  
  - *Inputs:* External (Telegram)  
  - *Outputs:* To AI Agent node  
  - *Edge Cases:* Telegram API downtime, invalid message formats, webhook errors.  

- **AI Agent** (@n8n/n8n-nodes-langchain.agent)  
  - *Type:* AI agent node using Langchain integration  
  - *Role:* Processes user's natural language commands, interprets intents (add/list/update/delete tasks), and formats responses.  
  - *Config:*  
    - Input text from Telegram message text expression.  
    - System message defines a warm, friendly assistant persona focused on task management via Google Sheets.  
    - Strict instructions to always update Google Sheets and confirm updates in natural language.  
    - Uses GPT-4o model through linked OpenAI Chat Model node (ai_languageModel connection).  
  - *Inputs:* From Telegram Trigger, Google Sheets nodes (through ai_tool connections).  
  - *Outputs:* To Telegram node for responding.  
  - *Edge Cases:* AI misinterpretation, Google Sheets sync failure, response generation errors.  
  - *Version:* 1.8  

- **OpenAI Chat Model** (@n8n/n8n-nodes-langchain.lmChatOpenAi)  
  - *Type:* Language model node  
  - *Role:* Provides GPT-4o model inference for AI Agent.  
  - *Config:* Model set to "gpt-4o" with OpenAI API credentials.  
  - *Inputs:* From AI Agent node.  
  - *Outputs:* To AI Agent node.  
  - *Edge Cases:* API rate limits, auth errors, timeouts.  

- **Google Sheets7** (Google Sheets node)  
  - *Type:* Google Sheets integration  
  - *Role:* Reads existing tasks from a specific Google Sheets document and sheet (gid=0).  
  - *Config:* Uses OAuth2 credentials; document ID and sheet name are predefined.  
  - *Inputs:* From Webhook node.  
  - *Outputs:* To If node for filtering tasks.  
  - *Edge Cases:* API quota exceeded, permission denied, sheet not found.  

- **If**  
  - *Type:* Conditional logic  
  - *Role:* Filters tasks where Status = "pending" AND Due Date is now or later (timezone Asia/Bangkok).  
  - *Config:* Checks these two conditions strictly.  
  - *Inputs:* From Google Sheets7 (task list).  
  - *Outputs:* True branch to Loop Over Items; false branch to Telegram3.  
  - *Edge Cases:* Date parsing errors, timezone conversion issues.  

- **Loop Over Items** (SplitInBatches)  
  - *Type:* Iterator  
  - *Role:* Iterates over filtered tasks to process each individually.  
  - *Config:* Default batch size (1 or unspecified).  
  - *Inputs:* From If node.  
  - *Outputs:* Two branches: one empty, one to Telegram3.  
  - *Edge Cases:* Large task lists causing performance delays.  

- **Telegram3** (Telegram node)  
  - *Type:* Messaging output  
  - *Role:* Sends reminder messages to users about pending due tasks.  
  - *Config:* Text template: "⏰ Reminder: “{{ Task }}” is due." Chat ID from incoming message.  
  - *Inputs:* From Loop Over Items node.  
  - *Outputs:* To Google Sheets8 node.  
  - *Edge Cases:* Telegram API failure, user blocked bot.  

- **Google Sheets8** (Google Sheets node)  
  - *Type:* Google Sheets integration  
  - *Role:* Updates the task status to "done" after sending reminder.  
  - *Config:* Append or update operation matching by Task column, changing Status to "done".  
  - *Inputs:* From Telegram3 node.  
  - *Outputs:* To Loop Over Items node (second output, empty).  
  - *Edge Cases:* Write conflicts, API errors.  

- **Telegram** (Telegram node)  
  - *Type:* Messaging output  
  - *Role:* Sends AI Agent's conversational response back to the Telegram user.  
  - *Config:* Text from AI Agent output, chat ID from original Telegram message sender.  
  - *Inputs:* From AI Agent.  
  - *Outputs:* None (end of flow).  
  - *Edge Cases:* Telegram API issues.  

- **Google Sheets, Google Sheets1, Google Sheets2** (Google Sheets Tool nodes)  
  - *Type:* Google Sheets helpers  
  - *Role:* Perform add/update (Google Sheets1), delete (Google Sheets2), and read (Google Sheets) operations on task data as instructed by AI Agent.  
  - *Config:* Use same document and sheet; map columns as per task data schema.  
  - *Inputs:* From AI Agent node via ai_tool connections.  
  - *Outputs:* To AI Agent node to confirm success/failure.  
  - *Edge Cases:* API errors, mismatched data, concurrency issues.  

- **Webhook**  
  - *Type:* Webhook (alternative entry point)  
  - *Role:* Provides a manual or external trigger to load tasks from Google Sheets7.  
  - *Inputs/Outputs:* Connected to Google Sheets7.  
  - *Edge Cases:* Unauthorized access, webhook misconfiguration.  

- **Sticky Notes (Telegram Start, Duplicate Not data to enter)**  
  - Provide contextual comments for the Telegram interaction, reminding about user input and data duplication rules.

---

#### 2.2 Scheduled Task Reminders & Summaries

**Overview:**  
This block triggers daily at 21:00 Bangkok time to send reminders for pending tasks due and to provide a friendly summary of completed and future tasks via Telegram.

**Nodes Involved:**  
- Schedule Trigger1  
- AI Agent2  
- OpenAI Chat Model2  
- Google Sheets5  
- Telegram4  

**Node Details:**

- **Schedule Trigger1**  
  - *Type:* Schedule trigger  
  - *Role:* Initiates the flow daily at 21:00 (9 PM) Bangkok time.  
  - *Config:* Hour-based interval trigger set to 21.  
  - *Inputs:* None  
  - *Outputs:* To AI Agent2 node.  
  - *Edge Cases:* Server downtime, timezone misconfiguration.  

- **Google Sheets5** (Google Sheets Tool)  
  - *Type:* Google Sheets read operation  
  - *Role:* Reads the full task list from the Google Sheets document.  
  - *Config:* Uses OAuth2 credentials; document and sheet IDs predefined.  
  - *Inputs:* From AI Agent2 (ai_tool connection).  
  - *Outputs:* To AI Agent2 node.  
  - *Edge Cases:* API errors, empty data, permission issues.  

- **AI Agent2** (@n8n/n8n-nodes-langchain.agent)  
  - *Type:* AI agent node  
  - *Role:* Generates a natural, warm summary message of tasks completed today and future pending tasks using GPT-4o-mini.  
  - *Config:*  
    - Input text prompts detailed filtering criteria for tasks completed today and future tasks based on status, due date, and creation date.  
    - Instructions to write in a conversational, friendly tone without technical jargon or timestamps.  
    - Uses OpenAI Chat Model2 node as the language model.  
  - *Inputs:* From Schedule Trigger1, Google Sheets5  
  - *Outputs:* To Telegram4 node.  
  - *Edge Cases:* AI text generation errors, empty task data.  
  - *Version:* 1.8  

- **OpenAI Chat Model2**  
  - *Type:* Language model node  
  - *Role:* Provides GPT-4o-mini model inference for AI Agent2.  
  - *Config:* Model set to "gpt-4o-mini" with OpenAI API credentials.  
  - *Inputs:* From AI Agent2 node.  
  - *Outputs:* To AI Agent2 node.  
  - *Edge Cases:* API rate limits, auth errors, timeouts.  

- **Telegram4** (Telegram node)  
  - *Type:* Messaging output  
  - *Role:* Sends the AI-generated summary message to the user on Telegram.  
  - *Config:* Text from AI Agent2 output, chat ID from JSON message field.  
  - *Inputs:* From AI Agent2 node.  
  - *Outputs:* None (end of scheduled flow).  
  - *Edge Cases:* Telegram API failure, user blocked bot.  

- **Sticky Note2**  
  - Notes that this block relates to the scheduled reminder and summary functionality.

---

### 3. Summary Table

| Node Name          | Node Type                              | Functional Role                               | Input Node(s)          | Output Node(s)            | Sticky Note                                    |
|--------------------|--------------------------------------|-----------------------------------------------|------------------------|---------------------------|------------------------------------------------|
| Telegram Trigger    | n8n-nodes-base.telegramTrigger       | Entry point for Telegram user messages         | External                | AI Agent                  | ## Telegram Start user to send message ai agent to work google sheet read,append or update and delete the data in sheet |
| AI Agent           | @n8n/n8n-nodes-langchain.agent       | Processes Telegram commands, syncs Google Sheets | Telegram Trigger, Google Sheets, Google Sheets1, Google Sheets2 | Telegram                  |                                                |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o model for AI Agent              | AI Agent                | AI Agent                  |                                                |
| Google Sheets       | n8n-nodes-base.googleSheetsTool      | Reads tasks from Google Sheets for AI Agent    | AI Agent                | AI Agent                  |                                                |
| Google Sheets1      | n8n-nodes-base.googleSheetsTool      | Adds or updates tasks in Google Sheets          | AI Agent                | AI Agent                  |                                                |
| Google Sheets2      | n8n-nodes-base.googleSheetsTool      | Deletes tasks in Google Sheets                   | AI Agent                | AI Agent                  |                                                |
| Google Sheets7      | n8n-nodes-base.googleSheets          | Reads all tasks from Google Sheets (gid=0)      | Webhook                 | If                        | ## Duplicate Not data to enter.                 |
| If                 | n8n-nodes-base.if                    | Filters pending tasks due now or later           | Google Sheets7          | Loop Over Items           |                                                |
| Loop Over Items     | n8n-nodes-base.splitInBatches         | Iterates over filtered pending tasks             | If                      | Telegram3, (empty branch) |                                                |
| Telegram3           | n8n-nodes-base.telegram              | Sends due task reminders to Telegram user        | Loop Over Items          | Google Sheets8            |                                                |
| Google Sheets8      | n8n-nodes-base.googleSheets          | Marks tasks as done after reminder sent          | Telegram3                | Loop Over Items (empty)   |                                                |
| Telegram            | n8n-nodes-base.telegram              | Sends AI Agent's conversational reply to user   | AI Agent                 | None                     |                                                |
| Webhook             | n8n-nodes-base.webhook               | Alternative trigger for loading tasks            | None                     | Google Sheets7            |                                                |
| Schedule Trigger1   | n8n-nodes-base.scheduleTrigger       | Daily trigger at 21:00 Bangkok time              | None                     | AI Agent2                 | ## Schedule                                    |
| AI Agent2           | @n8n/n8n-nodes-langchain.agent       | Generates daily summary message of tasks         | Schedule Trigger1, Google Sheets5 | Telegram4               |                                                |
| OpenAI Chat Model2  | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini model for AI Agent2         | AI Agent2                | AI Agent2                 |                                                |
| Google Sheets5      | n8n-nodes-base.googleSheetsTool      | Reads all tasks for scheduled summary            | AI Agent2                | AI Agent2                 |                                                |
| Telegram4           | n8n-nodes-base.telegram              | Sends daily task summary message to Telegram user | AI Agent2                | None                     |                                                |
| Sticky Note         | n8n-nodes-base.stickyNote            | Informative note about Telegram start block      | None                     | None                     | ## Telegram Start user to send message ai agent to work google sheet read,append or update and delete the data in sheet |
| Sticky Note1        | n8n-nodes-base.stickyNote            | Reminder about avoiding data duplication          | None                     | None                     | ## Duplicate Not data to enter.                 |
| Sticky Note2        | n8n-nodes-base.stickyNote            | Note indicating schedule-related block            | None                     | None                     | ## Schedule                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" update type only  
   - Credentials: Link a valid Telegram Bot API credential  
   - Position: Entry point at [0,0]

2. **Create AI Agent Node (AI Agent)**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text input: Expression `={{ $('Telegram Trigger').item.json.message.text }}`  
     - System Message: Use the detailed prompt defining the assistant persona, task operation rules, Google Sheets data structure, and conversational style as provided in the description  
     - Prompt Type: define  
     - Output parser enabled  
   - Position: [400,0]  
   - Connect input from Telegram Trigger node

3. **Create OpenAI Chat Model Node for AI Agent**  
   - Type: Langchain OpenAI Chat Model  
   - Parameters: Model set to "gpt-4o"  
   - Credentials: Configure OpenAI API credentials  
   - Position: [200,420]  
   - Connect as ai_languageModel input to AI Agent node

4. **Create Google Sheets Nodes for AI Agent Operations**  
   - Google Sheets (Read):  
     - Operation: Read rows  
     - Document ID and Sheet Name: Use your Google Sheets task list document and sheet  
     - Credentials: Google Sheets OAuth2  
     - Position: [500,400]  
     - Connect as ai_tool input to AI Agent node  
   - Google Sheets1 (Append or Update):  
     - Operation: Append or update rows by Task column  
     - Map columns Task, Notes, Status, Due Date, Created At  
     - Credentials: OAuth2  
     - Position: [720,400]  
     - Connect as ai_tool input to AI Agent node  
   - Google Sheets2 (Delete rows):  
     - Operation: Delete rows with parameters Start Row and Number of Rows from AI outputs  
     - Credentials: OAuth2  
     - Position: [940,400]  
     - Connect as ai_tool input to AI Agent node  

5. **Create Telegram Node for AI Agent Replies**  
   - Type: Telegram  
   - Parameters: Text from `={{ $json.output }}`, Chat ID from `={{ $('Telegram Trigger').item.json.message.from.id }}`  
   - Credentials: Telegram API  
   - Position: [940,0]  
   - Connect input from AI Agent node

6. **Create Webhook Node (Optional)**  
   - Type: Webhook  
   - Parameters: Path custom as needed  
   - Position: [-300,780]  
   - Connect output to Google Sheets7 node

7. **Create Google Sheets7 Node (Read All Tasks)**  
   - Type: Google Sheets  
   - Operation: Read rows from task list sheet  
   - Credentials: OAuth2  
   - Position: [-60,780]  
   - Connect input from Webhook

8. **Create If Node (Filter Pending Due Tasks)**  
   - Type: If  
   - Conditions:  
     - Status equals "pending"  
     - Due Date after or equals current datetime in Asia/Bangkok timezone (expression)  
   - Position: [220,780]  
   - Connect input from Google Sheets7

9. **Create Loop Over Items Node**  
   - Type: SplitInBatches  
   - Position: [500,780]  
   - Connect true branch output from If node

10. **Create Telegram3 Node (Send Reminders)**  
    - Type: Telegram  
    - Parameters: Text template `=⏰ Reminder: “{{ $json.Task }}” is due.`  
    - Chat ID expression: `{{ $json.message.from.id }}`  
    - Credentials: Telegram API  
    - Position: [760,780]  
    - Connect input from Loop Over Items node

11. **Create Google Sheets8 Node (Mark Task Done)**  
    - Type: Google Sheets  
    - Operation: Append or update rows, set Status to "done" for matching Task  
    - Credentials: OAuth2  
    - Position: [960,780]  
    - Connect input from Telegram3

12. **Create Schedule Trigger1 Node**  
    - Type: Schedule Trigger  
    - Parameters: Trigger every day at 21:00 (hour 21)  
    - Position: [-160,1260]

13. **Create Google Sheets5 Node (Read Tasks for Summary)**  
    - Type: Google Sheets  
    - Operation: Read rows from task list sheet  
    - Credentials: OAuth2  
    - Position: [280,1520]  
    - Connect as ai_tool input to AI Agent2 node

14. **Create AI Agent2 Node (Daily Summary Generator)**  
    - Type: Langchain Agent  
    - Parameters:  
      - Text: Detailed prompt specifying task filtering (completed today, future pending) and conversational tone as provided  
      - Prompt Type: define  
    - Position: [120,1260]  
    - Connect input from Schedule Trigger1 node

15. **Create OpenAI Chat Model2 Node**  
    - Type: Langchain OpenAI Chat Model  
    - Parameters: Model set to "gpt-4o-mini"  
    - Credentials: OpenAI API  
    - Position: [120,1520]  
    - Connect as ai_languageModel input to AI Agent2 node

16. **Create Telegram4 Node (Send Daily Summary)**  
    - Type: Telegram  
    - Parameters: Text from AI Agent2 output, Chat ID from JSON message field  
    - Credentials: Telegram API  
    - Position: [620,1260]  
    - Connect input from AI Agent2 node

17. **Create Sticky Notes**  
    - Place sticky notes near respective blocks with content:  
      - "## Telegram Start user to send message ai agent to work google sheet read,append or update and delete the data in sheet" near Telegram Trigger and AI Agent nodes  
      - "## Duplicate Not data to enter." near Google Sheets7  
      - "## Schedule" near Schedule Trigger1 and related nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The AI Agent system messages include strict instructions to always confirm Google Sheets updates in a warm, casual style, never skipping sync. | Core behavioral design for conversational AI task management.  |
| Timezone used throughout is Asia/Bangkok, ensure server and expressions respect this for date/time filtering and display.                      | Important for correct date filtering and reminders.             |
| Google Sheets document ID and sheet name (gid=0) are hardcoded; replace with your own for deployment.                                           | Integration parameter for Google Sheets nodes.                  |
| Telegram API credentials must be set up properly with OAuth2 or API token for bot authentication.                                               | Required for Telegram Trigger and Telegram nodes.               |
| OpenAI API keys for GPT-4o and GPT-4o-mini models are required and linked to respective nodes.                                                  | Required for AI Agent nodes.                                     |
| The workflow uses Langchain agent nodes with ai_tool and ai_languageModel custom connections to link Google Sheets and OpenAI nodes.            | Important for data flow and AI prompt handling.                 |
| Scheduled daily run at 21:00 Bangkok time sends reminders and task summaries automatically; ensure n8n instance timezone matches or converts time. | Scheduling and time consistency note.                           |

---

**Disclaimer:**  
The provided workflow and documentation comply strictly with all content policies. All data handled is legal and public. This analysis is based solely on the supplied n8n workflow JSON for technical integration and automation purposes.