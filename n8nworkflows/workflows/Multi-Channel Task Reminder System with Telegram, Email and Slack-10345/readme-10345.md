Multi-Channel Task Reminder System with Telegram, Email and Slack

https://n8nworkflows.xyz/workflows/multi-channel-task-reminder-system-with-telegram--email-and-slack-10345


# Multi-Channel Task Reminder System with Telegram, Email and Slack

---

### 1. Workflow Overview

This workflow implements a **Multi-Channel Task Reminder System** designed to receive tasks via webhook, store them in a PostgreSQL database, and send timely reminders through multiple communication channels: **Email, Slack, and Telegram**. It is intended for scenarios where users or systems need automated notifications about tasks scheduled for future times, with flexible channel delivery options.

The workflow is logically divided into two main blocks:

- **1.1 Task Reception and Storage:** Handles incoming task creation requests through a webhook, validates inputs, stores tasks in the database, and sends appropriate responses.

- **1.2 Reminder Dispatching:** Periodically queries the database for pending tasks due for reminders, routes each task reminder to the user’s preferred communication channel, sends the reminder, and updates the task status to prevent duplicate notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Task Reception and Storage

- **Overview:**  
  This block serves as the entry point for new task reminders. It receives incoming HTTP requests, validates the required fields, stores the task in the PostgreSQL database, and responds with success or error messages.

- **Nodes Involved:**  
  - Webhook - Receive Task  
  - Validate Input  
  - Save Task to Database  
  - Success Response  
  - Error Response  
  - Sticky Notes: "Webhook Entry Point", "Input Validation", "Database Storage", "Response Handlers"

- **Node Details:**

  - **Webhook - Receive Task**  
    - *Type:* Webhook (HTTP Request Listener)  
    - *Role:* Receives JSON task creation requests from users or external systems.  
    - *Configuration:* Path set as `task-YOUR_OPENAI_KEY_HERE-webhook` (replace with actual key). Expects POST requests.  
    - *Input/Output:* Inputs HTTP JSON body; outputs JSON data to Validate Input node.  
    - *Edge Cases:* Missing or malformed requests; invalid JSON format.  
    - *Sticky Note Content:* Describes expected JSON format with fields: userId, taskName, reminderTime, description (optional), channel (optional).  

  - **Validate Input**  
    - *Type:* If (Conditional) Node  
    - *Role:* Checks if `userId`, `taskName`, and `reminderTime` fields are non-empty strings.  
    - *Configuration:* Conditions using expressions like `={{ $json.body.userId }}` for strict non-empty validation.  
    - *Input/Output:* Inputs JSON from webhook; outputs to Save Task (if valid) or Error Response (if invalid).  
    - *Edge Cases:* Empty or missing required fields cause routing to error handler.  
    - *Sticky Note Content:* Explains required and optional fields and routing logic.  

  - **Save Task to Database**  
    - *Type:* PostgreSQL Node  
    - *Role:* Inserts new task record into `tasks` table in PostgreSQL.  
    - *Configuration:* Uses credentials for Postgres-test; columns mapped automatically from input data; table is `tasks` in `public` schema.  
    - *Input/Output:* Receives validated JSON, outputs inserted row data including task ID.  
    - *Edge Cases:* Database connection failures; schema mismatch; duplicate keys.  
    - *Sticky Note Content:* Describes table schema and alternative DB options.  

  - **Success Response**  
    - *Type:* Respond to Webhook  
    - *Role:* Returns HTTP 200 response with JSON confirming task creation and details (taskId, taskName, reminderTime).  
    - *Configuration:* Sends JSON with success=true and task info using expressions like `$json.id`.  
    - *Input/Output:* Input from DB insertion node; output to webhook client.  
    - *Edge Cases:* Response failures if node executes without valid input.  

  - **Error Response**  
    - *Type:* Respond to Webhook  
    - *Role:* Returns HTTP 400 response with JSON indicating invalid input.  
    - *Configuration:* Static JSON error message specifying missing required fields.  
    - *Input/Output:* Triggered on validation failure; output to webhook client.  
    - *Edge Cases:* None beyond validation catch.  

  - **Sticky Notes**  
    - Provide documentation for webhook input, validation logic, database schema, and response formats.  

---

#### 2.2 Reminder Dispatching

- **Overview:**  
  This block runs every 5 minutes to fetch pending tasks due for reminders, checks if any exist, splits tasks to process individually, routes each task reminder by specified channel, sends the reminder via the selected channel, and updates the task status to 'sent' to avoid duplicate notifications.

- **Nodes Involved:**  
  - Schedule Trigger - Every 5 Minutes  
  - Fetch Due Tasks  
  - Check Tasks Exist  
  - Split Into Items  
  - Route by Channel  
  - Send Email Reminder  
  - Send Slack Reminder  
  - Send Telegram Reminder  
  - Update Task Status  
  - Sticky Notes: "Schedule Trigger", "Fetch Due Tasks", "Task Checker", "Split Into Items", "Channel Router", "Email Sender", "Slack Sender", "Telegram Sender", "Workflow Complete"

- **Node Details:**

  - **Schedule Trigger - Every 5 Minutes**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates task fetching every 5 minutes using cron expression `*/5 * * * *`.  
    - *Input/Output:* Triggers Fetch Due Tasks node.  
    - *Edge Cases:* Cron misconfiguration; workflow not active.  
    - *Sticky Note Content:* Details cron schedule and adjustment options.  

  - **Fetch Due Tasks**  
    - *Type:* PostgreSQL Node  
    - *Role:* Executes SQL query to retrieve tasks with status 'pending' and reminder_time within the next 5 minutes window.  
    - *Configuration:* Query:  
      ```sql
      SELECT * FROM tasks WHERE status = 'pending' AND reminder_time <= NOW() + INTERVAL '5 minutes' AND reminder_time > NOW()
      ```  
    - *Input/Output:* Triggered by schedule; outputs array of pending tasks.  
    - *Edge Cases:* Empty result set; DB errors; time zone issues.  
    - *Sticky Note Content:* Explains selection criteria for timely reminders without duplicates.  

  - **Check Tasks Exist**  
    - *Type:* If Node  
    - *Role:* Checks if any tasks were fetched (length > 0).  
    - *Configuration:* Condition compares `{{$json.length}}` > 0.  
    - *Input/Output:* If true, proceeds to Split Into Items; otherwise no action.  
    - *Edge Cases:* Zero tasks cause no downstream execution.  
    - *Sticky Note Content:* Describes conditional routing based on task availability.  

  - **Split Into Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes each fetched task individually for personalized reminder sending.  
    - *Input/Output:* Receives array of tasks; outputs one task at a time to Channel Router.  
    - *Edge Cases:* Large batch sizes could affect performance.  
    - *Sticky Note Content:* Emphasizes individual processing necessity.  

  - **Route by Channel**  
    - *Type:* Switch Node  
    - *Role:* Routes task reminder to the correct communication channel based on the `channel` field (email, slack, telegram, SMS).  
    - *Configuration:* Conditions on task's `channel` property; outputs mapped to respective sender nodes.  
    - *Input/Output:* Receives single task; outputs to Email, Slack, Telegram sender nodes accordingly.  
    - *Edge Cases:* Missing or unsupported channel values; no fallback defined.  
    - *Sticky Note Content:* Explains routing outputs, including support for multiple channels.  

  - **Send Email Reminder**  
    - *Type:* EmailSend Node  
    - *Role:* Sends an HTML email reminder to the user’s email address.  
    - *Configuration:*  
      - Subject: `"⏰ Task Reminder: {{ $json.task_name }}"`  
      - To: `={{ $json.user_id }}` (assumed to be email address)  
      - From: `"noreply@yourdomain.com"` (configurable)  
      - SMTP credentials required and configured.  
    - *Input/Output:* Receives task item; outputs to Update Task Status.  
    - *Edge Cases:* SMTP authentication failure; invalid email format; network timeouts.  
    - *Sticky Note Content:* Details email formatting and setup requirements.  

  - **Send Slack Reminder**  
    - *Type:* Slack Node  
    - *Role:* Sends direct Slack messages to users with task details using markdown formatting.  
    - *Configuration:*  
      - Text includes task name, reminder time, description (defaulted to 'No description' if empty).  
      - Sends message to user ID (assumed valid Slack user).  
      - Requires Slack API credentials and proper app setup.  
    - *Input/Output:* Receives task item; outputs to Update Task Status.  
    - *Edge Cases:* Slack API errors; invalid user IDs; message formatting issues.  
    - *Sticky Note Content:* Explains Slack app and bot token setup.  

  - **Send Telegram Reminder**  
    - *Type:* Telegram Node  
    - *Role:* Sends Telegram messages directly to user chat IDs with formatted task details.  
    - *Configuration:*  
      - Text uses bold formatting for task fields.  
      - Chat ID mapped from `user_id`.  
      - Requires Telegram bot token and user conversation initiated with bot.  
    - *Input/Output:* Receives task item; outputs to Update Task Status.  
    - *Edge Cases:* User never started chat with bot; invalid chat ID; bot token errors.  
    - *Sticky Note Content:* Details Telegram bot requirements.  

  - **Update Task Status**  
    - *Type:* PostgreSQL Node  
    - *Role:* Updates task status to 'sent' and sets `sent_at` timestamp to current time to prevent duplicate reminders.  
    - *Configuration:* SQL query:  
      ```sql
      UPDATE tasks SET status = 'sent', sent_at = NOW() WHERE id = {{ $json.id }}
      ```  
    - *Input/Output:* Receives task item; no further outputs connected.  
    - *Edge Cases:* DB errors; concurrent updates.  

  - **Sticky Notes**  
    - Provide detailed explanations for scheduling, querying, splitting, routing, sending, and final checklist with example SQL for creating the `tasks` table.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                              | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                      |
|----------------------------|-------------------------|----------------------------------------------|--------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Webhook - Receive Task      | Webhook                 | Entry point for task creation requests       | None                           | Validate Input                   | Describes expected webhook JSON input format and URL.                                         |
| Validate Input             | If                      | Validates required fields in input            | Webhook - Receive Task          | Save Task to Database, Error Response | Explains validation logic and handling missing fields.                                        |
| Save Task to Database       | PostgreSQL              | Stores new task record in database             | Validate Input                 | Success Response                 | Describes database schema and storage details.                                                |
| Success Response            | Respond to Webhook      | Sends success JSON response on task creation  | Save Task to Database           | None                            | Explains response format with task details.                                                   |
| Error Response              | Respond to Webhook      | Sends error JSON response on validation failure | Validate Input                 | None                            | Explains error response for invalid inputs.                                                  |
| Schedule Trigger - Every 5 Minutes | Schedule Trigger       | Triggers periodic task fetching every 5 minutes | None                           | Fetch Due Tasks                 | Details cron schedule and frequency options.                                                  |
| Fetch Due Tasks             | PostgreSQL              | Queries database for pending tasks due soon   | Schedule Trigger - Every 5 Minutes | Check Tasks Exist              | Explains SQL query criteria to find due tasks.                                               |
| Check Tasks Exist           | If                      | Checks if any due tasks exist                  | Fetch Due Tasks                | Split Into Items                | Describes conditional routing based on task availability.                                   |
| Split Into Items            | SplitInBatches          | Processes tasks individually                   | Check Tasks Exist              | Route by Channel                | Emphasizes need for individual task processing.                                              |
| Route by Channel            | Switch                  | Routes task reminders by communication channel | Split Into Items              | Send Email Reminder, Send Slack Reminder, Send Telegram Reminder | Explains routing outputs for email, Slack, Telegram, SMS (SMS not fully configured).          |
| Send Email Reminder         | EmailSend               | Sends task reminder email                       | Route by Channel               | Update Task Status             | Describes email formatting and SMTP credentials needed.                                     |
| Send Slack Reminder         | Slack                   | Sends Slack task reminder message               | Route by Channel               | Update Task Status             | Describes Slack app and bot token setup.                                                     |
| Send Telegram Reminder      | Telegram                | Sends Telegram task reminder message            | Route by Channel               | Update Task Status             | Details Telegram bot requirements.                                                           |
| Update Task Status          | PostgreSQL              | Marks task as sent to avoid duplicate reminders | Send Email Reminder, Send Slack Reminder, Send Telegram Reminder | None                            | Updates task status and sent timestamp.                                                      |
| Sticky Note (Webhook Entry) | Sticky Note             | Documentation on webhook input format and usage | None                           | None                           | Provides full JSON example and webhook URL instructions.                                    |
| Sticky Note1 (Validation)  | Sticky Note             | Documentation on input validation               | None                           | None                           | Details required and optional fields for validation.                                        |
| Sticky Note2 (DB Storage)  | Sticky Note             | Documentation on database schema and storage   | None                           | None                           | Describes table schema and alternatives.                                                    |
| Sticky Note3 (Response)    | Sticky Note             | Documentation on success and error response    | None                           | None                           | Explains response formats and status codes.                                                 |
| Sticky Note4 (Scheduler)   | Sticky Note             | Documentation on schedule trigger settings     | None                           | None                           | Provides cron schedule details.                                                             |
| Sticky Note5 (Fetch Tasks) | Sticky Note             | Documentation on task fetching SQL and logic   | None                           | None                           | Explains selection criteria for due tasks.                                                 |
| Sticky Note6 (Task Checker)| Sticky Note             | Documentation on task existence check           | None                           | None                           | Describes routing based on task presence.                                                  |
| Sticky Note7 (Split Items) | Sticky Note             | Documentation on splitting tasks for processing | None                           | None                           | Explains rationale for splitting tasks individually.                                       |
| Sticky Note8 (Router)      | Sticky Note             | Documentation on routing by communication channel | None                           | None                           | Lists supported channels and output assignments.                                           |
| Sticky Note9 (Email Sender)| Sticky Note             | Documentation on email reminder setup           | None                           | None                           | Details email format and setup instructions.                                              |
| Sticky Note10 (Slack Sender)| Sticky Note            | Documentation on Slack reminder setup           | None                           | None                           | Details Slack app and bot token requirements.                                              |
| Sticky Note11 (Telegram Sender) | Sticky Note         | Documentation on Telegram reminder setup        | None                           | None                           | Explains Telegram bot setup and user requirements.                                        |
| Sticky Note13 (Workflow Complete) | Sticky Note        | Final setup checklist and database creation SQL | None                           | None                           | Provides checklist and example SQL to create `tasks` table.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Webhook - Receive Task):**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `task-YOUR_OPENAI_KEY_HERE-webhook` (replace placeholder with actual key)  
   - Purpose: Receive JSON task creation requests.  

2. **Create If Node (Validate Input):**  
   - Type: If (Condition)  
   - Conditions:  
     - Check `$json.body.userId` is a non-empty string  
     - Check `$json.body.taskName` is a non-empty string  
     - Check `$json.body.reminderTime` is a non-empty string  
   - Output 1 (true): To Save Task to Database  
   - Output 2 (false): To Error Response  

3. **Create PostgreSQL Node (Save Task to Database):**  
   - Credentials: Use configured PostgreSQL credentials  
   - Operation: Insert  
   - Table: `tasks` in schema `public`  
   - Columns mapped automatically from input JSON fields: user_id, task_name, reminder_time, description, channel, status (default to 'pending'), created_at (current timestamp)  
   - Connect input from Validate Input (true output)  

4. **Create Respond to Webhook Node (Success Response):**  
   - Response Type: JSON  
   - Response Body:  
     ```json
     {
       "success": true,
       "message": "Task created successfully!",
       "taskId": {{ $json.id }},
       "taskName": {{ $json.task_name }},
       "reminderTime": {{ $json.reminder_time }}
     }
     ```  
   - Connect input from Save Task to Database output  

5. **Create Respond to Webhook Node (Error Response):**  
   - Response Type: JSON  
   - Response Body:  
     ```json
     {
       "success": false,
       "message": "Invalid input. Required fields: userId, taskName, reminderTime"
     }
     ```  
   - Connect input from Validate Input (false output)  

6. **Create Schedule Trigger Node (Schedule Trigger - Every 5 Minutes):**  
   - Cron Expression: `*/5 * * * *`  
   - Purpose: Trigger the reminder dispatch flow every 5 minutes  

7. **Create PostgreSQL Node (Fetch Due Tasks):**  
   - Credentials: PostgreSQL credentials  
   - Operation: Execute Query  
   - Query:  
     ```sql
     SELECT * FROM tasks WHERE status = 'pending' AND reminder_time <= NOW() + INTERVAL '5 minutes' AND reminder_time > NOW()
     ```  
   - Connect input from Schedule Trigger output  

8. **Create If Node (Check Tasks Exist):**  
   - Condition: Check if `{{$json.length}}` > 0  
   - True output: Connect to Split Into Items  
   - False output: No further action  

9. **Create SplitInBatches Node (Split Into Items):**  
   - Default batch size (can be left default or adjusted as needed)  
   - Connect input from Check Tasks Exist (true output)  

10. **Create Switch Node (Route by Channel):**  
    - Rules based on `{{$json.channel}}` string value  
    - Outputs:  
      - Output 0: "email" → Connect to Send Email Reminder  
      - Output 1: "slack" → Connect to Send Slack Reminder  
      - Output 2: "telegram" → Connect to Send Telegram Reminder  
      - Output 3: "sms" (optional, not fully configured)  

11. **Create EmailSend Node (Send Email Reminder):**  
    - SMTP Credentials: Configure SMTP credentials with valid SMTP server  
    - To Email: `={{ $json.user_id }}` (assumed email)  
    - From Email: Set to your sender email, e.g., `noreply@yourdomain.com`  
    - Subject: `⏰ Task Reminder: {{ $json.task_name }}`  
    - Email Body: Compose with task details and professional styling (can use HTML or plain text)  
    - Connect input from Route by Channel (email output)  

12. **Create Slack Node (Send Slack Reminder):**  
    - Slack API Credentials: Configure Slack bot token with appropriate scope  
    - Text:  
      ```
      🔔 *Task Reminder*
      
      *Task:* {{ $json.task_name }}
      *Time:* {{ $json.reminder_time }}
      *Description:* {{ $json.description || 'No description' }}
      ```  
    - User: Map to Slack user ID (may require mapping in DB or external lookup)  
    - Connect input from Route by Channel (slack output)  

13. **Create Telegram Node (Send Telegram Reminder):**  
    - Telegram API Credentials: Configure Telegram bot token  
    - Chat ID: `={{ $json.user_id }}` (must be Telegram user chat ID)  
    - Text:  
      ```
      🔔 *Task Reminder*
      
      *Task:* {{ $json.task_name }}
      *Time:* {{ $json.reminder_time }}
      *Description:* {{ $json.description || 'No description' }}
      ```  
    - Connect input from Route by Channel (telegram output)  

14. **Create PostgreSQL Node (Update Task Status):**  
    - Credentials: PostgreSQL credentials  
    - Operation: Execute Query  
    - Query:  
      ```sql
      UPDATE tasks SET status = 'sent', sent_at = NOW() WHERE id = {{ $json.id }}
      ```  
    - Connect input from each reminder sender node output (Email, Slack, Telegram)  

15. **Test the entire flow:**  
    - Send sample webhook POST requests with valid and invalid payloads.  
    - Verify database insertion and response correctness.  
    - Confirm scheduled reminders are sent via configured channels at correct times.  
    - Check database status updates to 'sent' after reminders.  

16. **Additional Setup:**  
    - Create `tasks` table in PostgreSQL as per provided SQL schema.  
    - Ensure all credentials (PostgreSQL, SMTP, Slack, Telegram) are configured and tested.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| The workflow assumes `user_id` field represents the contact identifier in each channel (email address, Slack user ID, Telegram chat ID). Proper mapping or validation may be needed upstream.                                                                                                                                                                      | Best practice to ensure accurate user/channel mapping.                                                       |
| Slack integration requires creating a Slack App, adding bot token with necessary scopes, and installing into the workspace.                                                                                                                                                                                                                                       | Slack API documentation: https://api.slack.com/apps                                                          |
| Telegram bot requires user to start conversation with the bot via @BotFather for messages to be delivered.                                                                                                                                                                                                                                                       | Telegram Bot API documentation: https://core.telegram.org/bots/api                                           |
| SMTP server must support authenticated sending from the configured `fromEmail`. Use secure credentials and test sending outside n8n first.                                                                                                                                                                                                                      | SMTP setup guides vary by provider (e.g., Gmail SMTP, SendGrid)                                              |
| The schedule trigger frequency is adjustable to balance timely reminders vs system load. Consider edge cases such as daylight saving changes and timezone consistency between n8n server and database.                                                                                                                                                              | Cron expression reference: https://crontab.guru/                                                             |
| For production use, consider adding error handling nodes for database errors, communication failures, and logging for audit trails.                                                                                                                                                                                                                             | n8n supports additional error workflow triggers and logging nodes                                            |
| Sample SQL to create the tasks table:  
  ```sql  
  CREATE TABLE tasks (  
    id SERIAL PRIMARY KEY,  
    user_id VARCHAR(255),  
    task_name VARCHAR(255),  
    reminder_time TIMESTAMP,  
    description TEXT,  
    channel VARCHAR(50),  
    status VARCHAR(50),  
    created_at TIMESTAMP,  
    sent_at TIMESTAMP  
  );  
  ```                                                                                                                                                                                                                               | Initial database setup step for this workflow                                                                |

---

**Disclaimer:**  
The provided documentation is derived exclusively from an automated n8n workflow. It respects all content policies and contains no illegal or protected data. All handled data is legal and publicly accessible.