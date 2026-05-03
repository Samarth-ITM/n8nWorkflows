Manage Personal Data with Multiple GPT-4o AI Agents: Gmail, Tasks & Finance on Telegram

https://n8nworkflows.xyz/workflows/manage-personal-data-with-multiple-gpt-4o-ai-agents--gmail--tasks---finance-on-telegram-8063


# Manage Personal Data with Multiple GPT-4o AI Agents: Gmail, Tasks & Finance on Telegram

### 1. Workflow Overview

This comprehensive n8n workflow orchestrates multiple AI-powered personal assistant agents, each specialized in managing different facets of personal data and productivity via Telegram, Gmail, Google Sheets, and AI models (OpenAI GPT-4o and Google Gemini). It targets users who want seamless, conversational management of their:

- **Personal finances** (income, expenses) synchronized with Google Sheets,
- **Email communication** with Gmail triggers and AI-driven email analysis and responses,
- **Task management** with Google Sheets integration for todo lists,
- **Work time tracking** including start/end times, breaks, and monthly reports.

The workflow is structured into four major logical blocks reflecting these domains, plus integration and notification layers:

- **1.1 Financial Management Block**: Receives Telegram messages about financial operations, employs an AI agent to interpret and update Google Sheets, then confirms back via Telegram.
- **1.2 Gmail Management Block**: Triggers on new Gmail emails, fetches full content, processes it with AI to analyze and summarize in Arabic, then sends enriched notifications to Telegram.
- **1.3 Task Management Block**: Handles Telegram messages related to task operations, managing todo lists in Google Sheets via an AI agent, with status confirmations sent to Telegram.
- **1.4 Work Tracking & Reporting Block**: Manages start/end work time logs via Telegram inputs, updates Google Sheets, and generates detailed monthly work hour reports in Arabic, formatted as PDFs and sent to Telegram.

At the core, a **Switch node** routes incoming Telegram messages to the appropriate block based on the Telegram forum topic. Each block includes AI agents configured with Arabic prompts, Google Sheets nodes to sync data, and Telegram nodes for user interaction.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Financial Management Block

- **Overview:**  
Processes Telegram messages about financial transactions in Arabic, updates a Google Sheets document tracking income and expenses, and sends friendly confirmation messages back to Telegram.

- **Nodes Involved:**  
  - Incoming Message (Telegram Trigger)  
  - Switch (message topic routing)  
  - AI Agent (Langchain agent: finance assistant)  
  - Google Sheets (read)  
  - Google Sheets1 (append or update)  
  - Google Sheets2 (delete rows)  
  - Simple Memory (context memory for session)  
  - OpenAI Chat Model (GPT-4o)  
  - Telegram (send confirmation message)  
  - AI Agent2 (summary generator)  
  - Telegram4 (send summary message)  
  - Schedule Trigger1 (periodic summary trigger)

- **Node Details:**  
  - **Incoming Message:** Telegram trigger listening for new messages; captures user input including chat and message text.  
  - **Switch:** Routes messages to "Expenses" topic for finance operations.  
  - **AI Agent:** Uses Langchain agent with a detailed system prompt in Arabic that instructs it to always update Google Sheets for financial operations (add, list, update, delete), enforce Arabic language, and confirm updates in a friendly, conversational style without technical jargon. The prompt includes JSON data structure expectations for syncing with Sheets.  
  - **Google Sheets / Google Sheets1 / Google Sheets2:** Perform reading, appending/updating, and deleting of rows in the designated Google Sheets document ("Expenses" sheet). Mapping is defined explicitly with columns like iD, Amount, Currency, Note, Debit / Credit, Date, Time.  
  - **Simple Memory:** Maintains conversational memory keyed by Telegram user ID to allow context preservation across interactions.  
  - **OpenAI Chat Model:** GPT-4o used as the underlying language model for the agent.  
  - **Telegram:** Sends the AI agent's output as a friendly confirmation message back to the Telegram group/channel/thread.  
  - **AI Agent2 and Schedule Trigger1:** Used to generate and send periodic financial summaries in Arabic to the Telegram channel.  

- **Edge Cases & Failure Modes:**  
  - Google Sheets API failure (e.g., quota limits, auth errors) – AI agent instructed to explain issues gracefully to the user.  
  - Input format ambiguity or missing fields – AI agent expected to enforce data structure and prompt accordingly.  
  - Telegram message thread or chat ID changes – could cause delivery failures.  
  - Time zone handling critical (Bangkok time zone used).  
  - Concurrent edits and race conditions in Google Sheets.  

---

#### 1.2 Gmail Management Block

- **Overview:**  
Listens for new incoming Gmail emails, fetches full email content, processes it through AI models (Google Gemini and OpenAI GPT-4o-mini) to generate Arabic analysis including priority, summary, and recommended actions, then sends notifications to a dedicated Telegram thread.

- **Nodes Involved:**  
  - Gmail Trigger (New Emails)  
  - Fetch Full Email Body (Gmail node)  
  - Parse & Structure Email Data (Set node)  
  - Message a model (Google Gemini agent)  
  - Message a model3 (OpenAI GPT-4o-mini agent fallback)  
  - Send to Telegram (Gmail Channel)  
  - Send to Telegram (Gmail Channel)6 (error message)  

- **Node Details:**  
  - **Gmail Trigger:** Polls Gmail API every minute for new emails, requires OAuth2 Gmail credentials.  
  - **Fetch Full Email Body:** Retrieves complete message HTML/text for AI analysis.  
  - **Parse & Structure Email Data:** Extracts relevant email fields (from, subject, body, messageId) into a clean JSON object for AI consumption.  
  - **Message a model (Google Gemini):** Uses Google Gemini AI to analyze email and produce a plain-text Arabic summary with fields: summary, priority, action, urgency, sentiment. Includes prompt with strict formatting instructions—no JSON or code blocks, plain text only. Retries on failure up to 3 times.  
  - **Message a model3 (OpenAI GPT-4o-mini):** Backup AI model to process email analysis if Gemini fails.  
  - **Send to Telegram (Gmail Channel):** Posts the AI-generated email summary into a dedicated Telegram topic thread for Gmail notifications.  
  - **Send to Telegram (Gmail Channel)6:** Sends an error or status message if AI processing is delayed or fails.  

- **Edge Cases & Failure Modes:**  
  - Gmail API auth or quota errors.  
  - Full email content fetch failures (e.g., message deleted).  
  - AI model failures or timeouts.  
  - Parsing or prompt misformatting causing incomplete output.  
  - Telegram message thread ID or chat ID changes causing send failures.  

---

#### 1.3 Task Management Block

- **Overview:**  
Manages task operations triggered by Telegram messages under the "task" topic. AI agent interprets commands to add, list, update, complete, or delete tasks synchronized with Google Sheets, responding in Arabic with friendly confirmation messages.

- **Nodes Involved:**  
  - AI Agent3 (Langchain agent for tasks)  
  - Google Sheets3 (read tasks)  
  - Google Sheets4 (append or update tasks)  
  - Google Sheets6 (delete tasks)  
  - Google Sheets7 (read tasks for summary)  
  - Simple Memory2 (session memory for tasks)  
  - OpenAI Chat Model3 (GPT-4o)  
  - Telegram1 (send task confirmations)  
  - Schedule Trigger (daily summary trigger)  
  - AI Agent4 (task summary generator)  
  - OpenAI Chat Model4 (GPT-4o-mini for summaries)  
  - Telegram5 (send daily task summary)  

- **Node Details:**  
  - **AI Agent3:** System prompt instructs agent to always update Google Sheets for any task operation, respond in Arabic, and confirm changes in a warm, conversational style with occasional emojis. Data structure for tasks includes fields like intent, task description, status, date, notes.  
  - **Google Sheets3,4,6,7:** Used to read, append/update, delete, and read tasks for summary respectively, all operating on a "task" sheet within the Google Sheets document. Matching uses the "Task" column as key.  
  - **Simple Memory2:** Maintains conversational context per user.  
  - **OpenAI Chat Model3 and 4:** GPT-4o and GPT-4o-mini models used for language generation and summaries.  
  - **Telegram1 and Telegram5:** Send task operation confirmations and daily summaries to Telegram channel threads.  
  - **Schedule Trigger:** Runs daily (hour 11) to trigger task summary generation.  

- **Edge Cases & Failure Modes:**  
  - Concurrency issues updating the same task row.  
  - Invalid or missing task descriptions or status values.  
  - Google Sheets API errors.  
  - Telegram message delivery failures.  
  - User commands outside expected intents or malformed messages.  

---

#### 1.4 Work Tracking & Reporting Block

- **Overview:**  
Handles work time tracking via Telegram messages routed to the "Work" topic. AI agent manages start/end work records, updates Google Sheets, calculates total hours including breaks, and generates monthly detailed reports as PDFs sent to Telegram.

- **Nodes Involved:**  
  - Work Tracking AI Agent (Langchain agent for work tracking)  
  - Read Work Data & Read Work Data1 (Google Sheets read for work logs)  
  - Add/Update Work Record (Google Sheets append/update for work logs)  
  - Delete Work Record (Google Sheets delete for work logs)  
  - Simple Memory3 (session memory for work tracking)  
  - OpenAI Chat Model5 and 6 (GPT-4o models)  
  - Monthly Report Trigger (monthly schedule)  
  - Work Hours Analyzer (AI agent to create monthly reports)  
  - Split Workplace Reports (Code node to parse AI JSON output)  
  - Generate PDF Content (Code node to generate styled HTML report)  
  - Convert to file (Code node to convert HTML to file binary)  
  - Send PDF to Telegram1 (Telegram node to send PDF reports)  
  - Telegram Response (send responses to Telegram)  

- **Node Details:**  
  - **Work Tracking AI Agent:** System prompt in Arabic instructs the agent to always update Google Sheets with work start/end/query data, calculate total hours including 30-minute breaks for >4 hour shifts, and respond warmly in Arabic. Data structure includes action, date, start/end time, place, note, total hours, with time zone Baghdad.  
  - **Google Sheets nodes:** Read, append/update, and delete operations on the "work" sheet. Data includes Date, start/end time, place, note, total hours.  
  - **Simple Memory3:** Maintains session context per user.  
  - **Monthly Report Trigger:** Fires monthly to generate work reports.  
  - **Work Hours Analyzer:** AI agent that analyzes raw work data, groups by workplace, calculates totals and averages, and generates JSON reports.  
  - **Split Workplace Reports:** Parses AI JSON output to multiple items for report generation.  
  - **Generate PDF Content:** Builds a styled Arabic HTML report including tables of daily work details.  
  - **Convert to file:** Encodes HTML to base64 binary for file sending.  
  - **Send PDF to Telegram1:** Sends generated PDF reports to Telegram channel thread.  
  - **Telegram Response:** Sends user responses for immediate queries or confirmations.  

- **Edge Cases & Failure Modes:**  
  - Incorrect or missing start/end times leading to wrong hour calculations.  
  - Google Sheets API failures or concurrency conflicts.  
  - Errors parsing AI JSON reports.  
  - PDF generation issues or format errors.  
  - Telegram delivery failures.  
  - Time zone mismatches or daylight saving time considerations.  

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                              | Input Node(s)                      | Output Node(s)                           | Sticky Note                                                                                                   |
|-------------------------------|---------------------------------------|----------------------------------------------|----------------------------------|------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Incoming Message               | Telegram Trigger                      | Entry point for Telegram messages             |                                  | Switch                                  |                                                                                                               |
| Switch                        | Switch                               | Routes messages by Telegram forum topic       | Incoming Message                 | AI Agent, AI Agent1, AI Agent3, Work Tracking AI Agent |                                                                                                               |
| AI Agent                     | Langchain Agent                      | Finance AI agent to manage personal finances  | Switch                         | Telegram                                |                                                                                                               |
| Telegram                     | Telegram                             | Sends finance confirmation messages            | AI Agent                       |                                          |                                                                                                               |
| Google Sheets                | Google Sheets Tool                   | Reads finance data from "Expenses" sheet       | AI Agent (ai_tool)              | AI Agent (ai_tool)                       |                                                                                                               |
| Google Sheets1               | Google Sheets Tool                   | Append or update finance records                | AI Agent (ai_tool)              | AI Agent (ai_tool)                       |                                                                                                               |
| Google Sheets2               | Google Sheets Tool                   | Deletes finance rows                             | AI Agent (ai_tool)              | AI Agent (ai_tool)                       |                                                                                                               |
| Simple Memory                | Langchain Memory Buffer              | Maintains session memory for finance agent     | Incoming Message                | AI Agent                               |                                                                                                               |
| OpenAI Chat Model            | OpenAI GPT-4o                        | Language model for finance AI agent             | AI Agent (ai_languageModel)     | AI Agent                               |                                                                                                               |
| AI Agent2                    | Langchain Agent                     | Generates financial summaries periodically      | Schedule Trigger1               | Telegram4                             |                                                                                                               |
| Telegram4                    | Telegram                            | Sends financial summary messages                | AI Agent2                      |                                          |                                                                                                               |
| Schedule Trigger1            | Schedule Trigger                    | Triggers periodic financial summary generation |                                | AI Agent2                             |                                                                                                               |
| Gmail Trigger (New Emails)   | Gmail Trigger                      | Triggers on new incoming Gmail emails           |                                | Fetch Full Email Body                   | Triggers on new emails. Ensure OAuth2 is configured for Gmail API.                                           |
| Fetch Full Email Body        | Gmail                             | Retrieves full email content                     | Gmail Trigger                  | Parse & Structure Email Data            | Fetches the complete HTML/Body of the email for deeper AI analysis.                                          |
| Parse & Structure Email Data | Set                                | Structures email data for AI processing          | Fetch Full Email Body           | Message a model                        | Creates a clean, structured data object for the AI agent to process.                                         |
| Message a model              | Langchain Google Gemini Agent      | AI analyzes email content and generates summary | Parse & Structure Email Data    | Send to Telegram (Gmail Channel)       |                                                                                                               |
| Message a model3             | Langchain OpenAI Agent             | Backup AI email analysis with GPT-4o-mini       | Message a model                | Send to Telegram (Gmail Channel), Send to Telegram (Gmail Channel)6 |                                                                                                               |
| Send to Telegram (Gmail Channel) | Telegram                        | Sends AI email analysis to Telegram channel     | Message a model, Message a model3 |                                      | Sends the enriched notification to a dedicated Telegram topic thread. Uses env vars for IDs.                  |
| Send to Telegram (Gmail Channel)6 | Telegram                        | Sends error or status message for AI delays     | Message a model3               |                                          | Sends the enriched notification to a dedicated Telegram topic thread. Uses env vars for IDs.                  |
| AI Agent1                   | Langchain Agent                   | AI agent managing email user data & Gmail sending | Switch                        | Send to Telegram (Gmail Channel)8      |                                                                                                               |
| Send to Telegram (Gmail Channel)8 | Telegram                      | Sends email operation confirmations             | AI Agent1                     |                                          | Sends the enriched notification to a dedicated Telegram topic thread. Uses env vars for IDs.                  |
| Simple Memory1              | Langchain Memory Buffer            | Session memory for email management              | Incoming Message              | AI Agent1                             |                                                                                                               |
| Append row in sheet in Google Sheets | Google Sheets Tool            | Adds new email user data to Google Sheets        | AI Agent1                     | AI Agent1                             |                                                                                                               |
| Update row in sheet in Google Sheets | Google Sheets Tool            | Updates email user data in Google Sheets          | AI Agent1                     | AI Agent1                             |                                                                                                               |
| Get row(s) in sheet in Google Sheets1 | Google Sheets Tool          | Retrieves specific email user rows                | AI Agent1                     | AI Agent1                             |                                                                                                               |
| Delete rows or columns from sheet in Google Sheets | Google Sheets Tool | Deletes user rows in Google Sheets                 | AI Agent1                     | AI Agent1                             |                                                                                                               |
| Send a message in Gmail1    | Gmail Tool                        | Sends emails via Gmail                             | AI Agent1                     | AI Agent1                             |                                                                                                               |
| Send a text message in Telegram | Telegram Tool                   | Sends arbitrary Telegram messages                  | AI Agent1                     | AI Agent1                             |                                                                                                               |
| AI Agent3                   | Langchain Agent                   | Task management AI agent for todo lists            | Switch                        | Telegram1                            |                                                                                                               |
| Telegram1                   | Telegram                         | Sends task operation confirmation messages          | AI Agent3                    |                                          |                                                                                                               |
| Google Sheets3              | Google Sheets Tool               | Reads tasks data from Google Sheets                 | AI Agent3                    | AI Agent3                            |                                                                                                               |
| Google Sheets4              | Google Sheets Tool               | Appends or updates tasks data                        | AI Agent3                    | AI Agent3                            |                                                                                                               |
| Google Sheets6              | Google Sheets Tool               | Deletes tasks rows                                    | AI Agent3                    | AI Agent3                            |                                                                                                               |
| Google Sheets7              | Google Sheets Tool               | Reads tasks data for summaries                        | AI Agent4                    | AI Agent4                            |                                                                                                               |
| Simple Memory2              | Langchain Memory Buffer          | Session memory for task agent                         | Incoming Message             | AI Agent3                            |                                                                                                               |
| OpenAI Chat Model3          | OpenAI GPT-4o                   | Language model for task AI agent                       | AI Agent3                    | AI Agent3                            |                                                                                                               |
| OpenAI Chat Model4          | OpenAI GPT-4o-mini              | Language model for task summary AI agent               | AI Agent4                    | AI Agent4                            |                                                                                                               |
| AI Agent4                   | Langchain Agent                 | Task summary generator AI agent                        | Schedule Trigger             | Telegram5                           |                                                                                                               |
| Telegram5                   | Telegram                       | Sends daily task summaries                            | AI Agent4                    |                                          |                                                                                                               |
| Schedule Trigger            | Schedule Trigger               | Triggers daily task summary generation                |                              | AI Agent4                           |                                                                                                               |
| Work Tracking AI Agent      | Langchain Agent               | Work hours tracking AI agent                            | Switch                      | Telegram Response                  |                                                                                                               |
| Telegram Response           | Telegram                     | Sends work tracking confirmations                      | Work Tracking AI Agent       |                                          |                                                                                                               |
| Read Work Data              | Google Sheets Tool           | Reads work tracking data                                | Work Tracking AI Agent       | Work Tracking AI Agent             |                                                                                                               |
| Add/Update Work Record      | Google Sheets Tool           | Adds or updates work time records                        | Work Tracking AI Agent       | Work Tracking AI Agent             |                                                                                                               |
| Delete Work Record          | Google Sheets Tool           | Deletes work time records                                | Work Tracking AI Agent       | Work Tracking AI Agent             |                                                                                                               |
| Simple Memory3              | Langchain Memory Buffer      | Session memory for work tracking agent                   | Incoming Message             | Work Tracking AI Agent             |                                                                                                               |
| Monthly Report Trigger      | Schedule Trigger             | Monthly trigger to generate work hours report             |                              | Work Hours Analyzer              |                                                                                                               |
| Work Hours Analyzer         | Langchain Agent             | Generates monthly work hour reports in Arabic             | Monthly Report Trigger       | Split Workplace Reports          |                                                                                                               |
| Split Workplace Reports     | Code                        | Parses AI JSON output into structured items                | Work Hours Analyzer          | Generate PDF Content             |                                                                                                               |
| Generate PDF Content        | Code                        | Generates styled Arabic HTML report for work hours          | Split Workplace Reports      | convert to file                 |                                                                                                               |
| convert to file             | Code                        | Converts HTML report to base64 encoded binary for PDF       | Generate PDF Content         | Send PDF to Telegram1           |                                                                                                               |
| Send PDF to Telegram1       | Telegram                   | Sends monthly work hours PDF reports to Telegram channel    | convert to file              |                                  |                                                                                                               |
| Schedule Trigger1           | Schedule Trigger            | Periodic trigger for financial summary generation          |                              | AI Agent2                      |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node: "Incoming Message"**  
   - Type: Telegram Trigger  
   - Parameters: listen for "message" updates only  
   - Set webhook ID as needed  

2. **Add a Switch Node: "Switch"**  
   - Route based on `{{$json.message.reply_to_message.forum_topic_created.name}}`  
   - Conditions:  
     - "Expenses" → Finance block  
     - "Gmail" → Gmail block  
     - "task" → Task management block  
     - "Work" → Work tracking block  

3. **Finance Block Setup:**  
   - Langchain Agent Node "AI Agent":  
     - Prompt: Arabic system message instructing finance operations (add/list/update/delete) with Google Sheets sync, friendly tone, Arabic only, Bangkok timezone.  
     - Text Expression: Telegram message text  
     - Output parser enabled  
   - Google Sheets Nodes:  
     - "Google Sheets" to read from "Expenses" sheet (ID: 661184683, doc ID: 1S_M-0IxU84HYrkNHlIC7CivdI0KYlC3tWh48rAIx3O8)  
     - "Google Sheets1" appendOrUpdate with defined columns (iD, Amount, Currency, Note, Debit / Credit, Date, Time)  
     - "Google Sheets2" delete rows operation with dynamic startIndex and numberToDelete  
   - Simple Memory Node "Simple Memory":  
     - Session key: `{{$json.message.from.id}}`  
   - OpenAI Chat Model Node "OpenAI Chat Model":  
     - Model: "gpt-4o"  
   - Telegram Node "Telegram":  
     - Sends AI agent's output text to Telegram chat ID -1003057912037, message_thread_id 324  
   - Schedule Trigger "Schedule Trigger1":  
     - Set interval to daily or desired period  
   - Langchain Agent "AI Agent2":  
     - Prompt to generate financial summary in Arabic based on Google Sheets data  
   - Telegram Node "Telegram4":  
     - Send summary message to Telegram channel  

4. **Gmail Block Setup:**  
   - Gmail Trigger Node "Gmail Trigger (New Emails)":  
     - Poll every minute, OAuth2 Gmail credentials required  
   - Gmail Node "Fetch Full Email Body":  
     - Get full email content by message ID  
   - Set Node "Parse & Structure Email Data":  
     - Extract from, subject, body, messageId fields  
   - Langchain Google Gemini Agent "Message a model":  
     - System prompt to analyze email and produce Arabic formatted summary  
     - Retry on failure up to 3 times  
     - Google Gemini API credentials required  
   - Langchain OpenAI Agent "Message a model3":  
     - Backup AI model GPT-4o-mini with similar prompt  
   - Telegram Nodes "Send to Telegram (Gmail Channel)" and "Send to Telegram (Gmail Channel)6":  
     - Post AI summaries and error/status messages to Telegram topic thread 2 in chat -1003057912037  

5. **Email User Data Management:**  
   - Langchain Agent "AI Agent1":  
     - System prompt in Arabic for managing Gmail user data and sending emails  
     - Integrates with Gmail Tool nodes to send emails  
   - Google Sheets nodes: append, update, get rows, delete rows on "email" sheet (gid=0)  
   - Gmail Tool "Send a message in Gmail1":  
     - Sends emails using Gmail credentials  
   - Telegram nodes for confirmations  

6. **Task Management Block:**  
   - Langchain Agent "AI Agent3":  
     - Arabic prompt for todo list management with Google Sheets sync, friendly style  
   - Google Sheets nodes: read (Google Sheets3), append/update (Google Sheets4), delete (Google Sheets6), and read for summaries (Google Sheets7) targeting "task" sheet (ID: 259919770)  
   - Simple Memory "Simple Memory2" for context  
   - OpenAI Chat Models "OpenAI Chat Model3" and "OpenAI Chat Model4" for text generation and summaries  
   - Telegram Nodes "Telegram1" and "Telegram5" for task operation confirmations and daily summary  
   - Schedule Trigger for daily summary  

7. **Work Tracking Block:**  
   - Langchain Agent "Work Tracking AI Agent":  
     - Arabic prompt for managing work start/end times, calculating total hours including breaks, updating Google Sheets  
   - Google Sheets nodes: read (Read Work Data, Read Work Data1), append/update (Add/Update Work Record), delete (Delete Work Record) on "work" sheet (ID: 1596536961)  
   - Simple Memory3 for session context  
   - OpenAI Chat Models "OpenAI Chat Model5" and "OpenAI Chat Model6" for work tracking AI and report generation  
   - Schedule Trigger "Monthly Report Trigger" for monthly reports  
   - Langchain Agent "Work Hours Analyzer":  
     - Analyzes monthly work data, groups by workplace, calculates totals, outputs JSON report  
   - Code Nodes:  
     - "Split Workplace Reports": parse JSON from AI output  
     - "Generate PDF Content": build Arabic styled HTML report  
     - "convert to file": convert HTML to base64 binary for PDF  
   - Telegram Node "Send PDF to Telegram1":  
     - Sends the monthly work hour PDF report to Telegram chat thread 334  

8. **Connect all nodes accordingly:**  
   - Inputs from Telegram "Incoming Message" feed into "Switch" for routing.  
   - AI agents connect downstream to respective Google Sheets nodes and Telegram output nodes.  
   - Schedule triggers connect to periodic summary/report generation agents.  
   - Gmail trigger chain connected from Gmail trigger through email processing to Telegram notification.  

9. **Configure credentials:**  
   - Telegram Bot Token for all Telegram nodes.  
   - Google OAuth2 credentials for Google Sheets and Gmail nodes.  
   - OpenAI credentials for GPT-4o and GPT-4o-mini models.  
   - Google Gemini API credentials for Gemini agent.  

10. **Set environment variables for chat IDs and message thread IDs as used (e.g., -1003057912037).**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Workflow text and AI prompts are entirely in Arabic to maintain consistent user experience and cultural relevance.                                                                                                                                                                                                                                                                                          | Communication Style in AI Agent Prompts                                                                                     |
| Time zones are critical for date/time operations: Bangkok timezone for finance and task agents; Baghdad timezone for work tracking.                                                                                                                                                                                                                                                                        | System message prompts for agents                                                                                           |
| Google Sheets document URL: https://docs.google.com/spreadsheets/d/1S_M-0IxU84HYrkNHlIC7CivdI0KYlC3tWh48rAIx3O8                                                                                                                                                                                                                                                                                              | Master Google Sheets doc ID used in multiple nodes                                                                          |
| Telegram chat ID -1003057912037 is the main channel for all user communications; message_thread_id is used to organize threads for Expenses (324), Gmail (2), Tasks (163), and Work Reports (334).                                                                                                                                                                                                             | Telegram node parameters                                                                                                    |
| Gmail API requires OAuth2 with appropriate scopes for triggers and sending emails.                                                                                                                                                                                                                                                                                                                        | Gmail Trigger and Gmail Tool nodes                                                                                          |
| The workflow uses Langchain n8n nodes extensively to leverage AI capabilities with custom system prompts and memory buffers for contextual conversations.                                                                                                                                                                                                                                                  | Langchain node usage                                                                                                        |
| PDF report generation uses custom HTML+CSS styled for Arabic right-to-left display, enhancing readability and professionalism.                                                                                                                                                                                                                                                                              | Generate PDF Content code node                                                                                              |
| Error handling includes fallback AI models, retry logic, and user-friendly error messages sent via Telegram.                                                                                                                                                                                                                                                                                              | Message a model nodes and Telegram error messages                                                                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow designed with n8n, respecting all content policies and handling only legal and public data.