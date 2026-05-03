Employee Time Tracking System with GPT-4o Reports & Gmail Notifications

https://n8nworkflows.xyz/workflows/employee-time-tracking-system-with-gpt-4o-reports---gmail-notifications-10189


# Employee Time Tracking System with GPT-4o Reports & Gmail Notifications

### 1. Workflow Overview

This workflow implements an **Employee Time Tracking System** that records start, break, and end times via webhook inputs, stores the data in a data table, and integrates AI-generated reports and email notifications. Its primary use case is to track employee attendance and breaks, generate monthly summaries using GPT-4o, and send timely reminders and reports via Gmail.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Routing**: Receives HTTP POST requests through a webhook to track time actions (start, break, end) and routes them accordingly.
- **1.2 Time Entry Management**: Inserts or updates records in a centralized data table for employee schedules, handling start times, breaks, and end times.
- **1.3 Response Handling**: Constructs and sends appropriate text responses back to the webhook caller based on the action performed or errors encountered.
- **1.4 Scheduled Analytics and AI Reporting**: Periodically retrieves time entries to generate AI-based monthly summaries and daily clock-in reminders.
- **1.5 Email Notification**: Sends the AI-generated reports and reminders to employees and management using Gmail OAuth2 credentials.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Routing

##### Overview  
This block receives POST requests from external sources to track employee time events and routes the requests based on the specified method (`start`, `break`, or `end`).

##### Nodes Involved  
- Webhook - Track Time  
- Switch  
- Sticky Note (START, BREAK, END, WAIT)

##### Node Details

- **Webhook - Track Time**  
  - *Type*: Webhook  
  - *Role*: Entry point receiving POST requests at path `/track-time`.  
  - *Configuration*: HTTP POST method; response mode set to respond from downstream node.  
  - *Inputs/Outputs*: No input; output to Switch node.  
  - *Failures*: Network issues, malformed requests, missing headers/body.

- **Switch**  
  - *Type*: Switch  
  - *Role*: Routes data based on `body.method` field value.  
  - *Configuration*: Routes to three outputs named `Start`, `Break`, and `End` based on exact string matches.  
  - *Inputs*: From webhook.  
  - *Outputs*: To either Insert row (start), Get row(s) (break), or Get row(s)2 (end).  
  - *Failures*: Missing or invalid `method` field.

- **Sticky Notes (START, BREAK, END, WAIT)**  
  - *Type*: Sticky Note  
  - *Role*: Visual grouping and documentation markers for sections of the workflow.  
  - *No technical configuration or connectivity.*

---

#### 2.2 Time Entry Management

##### Overview  
This block manages the CRUD operations on the employee schedule data table, inserting new rows on start, updating break durations, and setting end times. It performs condition checks to avoid duplicate entries.

##### Nodes Involved  
- Insert row  
- Get row(s)  
- Get row(s)2  
- If1  
- If  
- Set Break Duration  
- Update row(s)  
- Update row(s)1  
- end  
- ERROR  
- ERROR1  
- MESSAGE, MESSAGE1, MESSAGE2  

##### Node Details

- **Insert row**  
  - *Type*: Data Table (Insert)  
  - *Role*: Inserts a new schedule row when an employee clocks in (`start` method).  
  - *Configuration*: Inserts `start` with today's date and `Employee` from webhook header `id`.  
  - *Inputs*: From Switch (Start output).  
  - *Outputs*: MESSAGE2 node.  
  - *Failures*: Data table access issues, missing employee ID.

- **Get row(s)**  
  - *Type*: Data Table (Get)  
  - *Role*: Retrieves schedule rows to check if start time exists for break method.  
  - *Configuration*: Fetches all rows; filtered later in If1 node.  
  - *Inputs*: From Switch (Break output).  
  - *Outputs*: If1 node.  
  - *Failures*: Data table read errors.

- **Get row(s)2**  
  - *Type*: Data Table (Get)  
  - *Role*: Retrieves schedule rows for end method processing.  
  - *Inputs*: From Switch (End output).  
  - *Outputs*: If node.  
  - *Failures*: Data table read errors.

- **If1**  
  - *Type*: If  
  - *Role*: Checks if a schedule entry with today's date already exists to decide break update or error.  
  - *Configuration*: Compares row `start` date to current date stored in `$today`.  
  - *Inputs*: From Get row(s).  
  - *Outputs*: Set Break Duration (true), ERROR (false).  
  - *Failures*: Date comparisons failing due to format.

- **If**  
  - *Type*: If  
  - *Role*: Checks if a schedule entry for today exists to update end time or error.  
  - *Configuration*: Compares row `start` date to `$today`.  
  - *Inputs*: From Get row(s)2.  
  - *Outputs*: end (true), ERROR1 (false).  
  - *Failures*: Date comparisons failing.

- **Set Break Duration**  
  - *Type*: Set  
  - *Role*: Extracts and sets the break duration from webhook body to a variable.  
  - *Inputs*: From If1 (true).  
  - *Outputs*: Update row(s).  
  - *Failures*: Missing/invalid duration field.

- **Update row(s)**  
  - *Type*: Data Table (Update)  
  - *Role*: Updates the break duration and start time for today's record.  
  - *Inputs*: From Set Break Duration.  
  - *Outputs*: MESSAGE1.  
  - *Failures*: Data table update errors.

- **Update row(s)1**  
  - *Type*: Data Table (Update)  
  - *Role*: Updates the end time for today's record, preserving break duration.  
  - *Inputs*: From end node.  
  - *Outputs*: MESSAGE.  
  - *Failures*: Data table update errors.

- **end**  
  - *Type*: Set  
  - *Role*: Sets the current timestamp as end time variable.  
  - *Inputs*: From If (true).  
  - *Outputs*: Update row(s)1.  
  - *Failures*: None significant.

- **ERROR** and **ERROR1**  
  - *Type*: Set  
  - *Role*: Sets an error message "ERROR" on failure branches.  
  - *Outputs*: Respond to Webhook.  
  - *Failures*: None (used for error propagation).

- **MESSAGE, MESSAGE1, MESSAGE2**  
  - *Type*: Set  
  - *Role*: Sets human-readable confirmation messages indicating the action performed (e.g., break time tracked, start time exists).  
  - *Outputs*: Respond to Webhook.  
  - *Failures*: None.

---

#### 2.3 Response Handling

##### Overview  
This block sends back textual responses to the webhook caller confirming the action or reporting errors.

##### Nodes Involved  
- Respond to Webhook  
- Sticky Note5  
- Sticky Note7

##### Node Details

- **Respond to Webhook**  
  - *Type*: Respond to Webhook  
  - *Role*: Sends the message from previous nodes as HTTP response text.  
  - *Inputs*: From MESSAGE, MESSAGE1, MESSAGE2, ERROR, ERROR1 nodes.  
  - *Outputs*: None (terminal node).  
  - *Failures*: Network errors, response timeouts.

- **Sticky Notes (RESPONSE, COMPLETED)**  
  - *Type*: Sticky Note  
  - *Role*: Visual markers for response and completion stages.

---

#### 2.4 Scheduled Analytics and AI Reporting

##### Overview  
This block runs scheduled triggers to collect time tracking data and generate AI-driven summary reports and reminders using OpenAI GPT-4o. It fetches data from the schedule data table and sends messages to AI nodes for analysis.

##### Nodes Involved  
- EVERY MONTH (Schedule Trigger)  
- EVERY DAY (Schedule Trigger)  
- ANALYZE ENTIRE MONTH (Data Table Get)  
- ANALYZE TIME ENTRIES (Data Table Get)  
- Message a model  
- Message a model1  
- Sticky Note4  
- Sticky Note6

##### Node Details

- **EVERY MONTH**  
  - *Type*: Schedule Trigger  
  - *Role*: Triggers monthly at 06:00 to run monthly analytics.  
  - *Outputs*: ANALYZE ENTIRE MONTH.

- **EVERY DAY**  
  - *Type*: Schedule Trigger  
  - *Role*: Triggers daily at 10:00 to generate daily clock-in reminders.  
  - *Outputs*: ANALYZE TIME ENTRIES.

- **ANALYZE ENTIRE MONTH**  
  - *Type*: Data Table (Get)  
  - *Role*: Fetches all schedule entries for the month without limit.  
  - *Outputs*: Message a model.

- **ANALYZE TIME ENTRIES**  
  - *Type*: Data Table (Get)  
  - *Role*: Fetches all schedule entries to analyze clock-in status daily.  
  - *Outputs*: Message a model1.

- **Message a model**  
  - *Type*: Langchain OpenAI Node  
  - *Role*: Sends the monthly data to GPT-4o for summarization.  
  - *Configuration*: Uses prompt "Analyze the time tracking for last month and provide me with a summary" with `$json` data appended.  
  - *Credentials*: OpenAI API key configured.  
  - *Outputs*: EMAIL TO MANAGEMENT.

- **Message a model1**  
  - *Type*: Langchain OpenAI Node  
  - *Role*: Sends daily data to GPT-4o to check if clock-in is missing and generate reminder text.  
  - *Configuration*: Uses prompt "IF I HAVEN’T CLOCKED IN TODAY, REMIND ME BY EMAIL" with `$json` data appended.  
  - *Credentials*: OpenAI API key configured.  
  - *Outputs*: EMAIL TO EMPLOYEE.

- **Sticky Notes (MONTHLY SUMMARY, CLOCK-IN REMINDER)**  
  - *Type*: Sticky Note  
  - *Role*: Visual documentation for scheduled AI analytics blocks.

---

#### 2.5 Email Notification

##### Overview  
This block sends the AI-generated messages via Gmail to employees and management to notify them of summaries or reminders.

##### Nodes Involved  
- EMAIL TO MANAGEMENT  
- EMAIL TO EMPLOYEE

##### Node Details

- **EMAIL TO MANAGEMENT**  
  - *Type*: Gmail (OAuth2)  
  - *Role*: Sends monthly report email to management.  
  - *Configuration*: Sends to `josea.castillodiez@gmail.com`, subject "SCHEDULE", message body from AI node.  
  - *Credentials*: Gmail OAuth2 account configured.  
  - *Failures*: OAuth token expiration, Gmail API limits.

- **EMAIL TO EMPLOYEE**  
  - *Type*: Gmail (OAuth2)  
  - *Role*: Sends daily reminder email to employee.  
  - *Configuration*: Same as above but triggered from daily AI node.  
  - *Credentials*: Gmail OAuth2.  
  - *Failures*: Same as above.

---

### 3. Summary Table

| Node Name           | Node Type              | Functional Role                             | Input Node(s)                 | Output Node(s)               | Sticky Note                            |
|---------------------|------------------------|--------------------------------------------|------------------------------|-----------------------------|--------------------------------------|
| Webhook - Track Time | Webhook                | Receives time tracking POST requests       |                              | Switch                      |                                      |
| Switch              | Switch                 | Routes request by method (start/break/end) | Webhook - Track Time          | Insert row, Get row(s), Get row(s)2 |                                      |
| Insert row          | Data Table (Insert)     | Inserts new start time record               | Switch (Start output)         | MESSAGE2                    |                                      |
| Get row(s)           | Data Table (Get)       | Gets rows for break processing              | Switch (Break output)         | If1                         |                                      |
| If1                  | If                     | Checks if start exists for break            | Get row(s)                   | Set Break Duration, ERROR   |                                      |
| Set Break Duration   | Set                    | Sets break duration variable                 | If1 (true)                   | Update row(s)               |                                      |
| Update row(s)        | Data Table (Update)    | Updates break duration in row                | Set Break Duration            | MESSAGE1                    |                                      |
| MESSAGE1             | Set                    | Sets confirmation message for break         | Update row(s)                | Respond to Webhook          |                                      |
| ERROR                | Set                    | Sets error message                           | If1 (false)                  | Respond to Webhook          |                                      |
| Get row(s)2          | Data Table (Get)       | Gets rows for end processing                 | Switch (End output)           | If                         |                                      |
| If                   | If                     | Checks if start exists for end               | Get row(s)2                  | end, ERROR1                 |                                      |
| end                  | Set                    | Sets end timestamp                           | If (true)                    | Update row(s)1              |                                      |
| Update row(s)1       | Data Table (Update)    | Updates end time in row                       | end                         | MESSAGE                     |                                      |
| MESSAGE              | Set                    | Sets confirmation message for end            | Update row(s)1               | Respond to Webhook          |                                      |
| ERROR1                | Set                    | Sets error message                           | If (false)                   | Respond to Webhook          |                                      |
| MESSAGE2             | Set                    | Sets message if start time already tracked   | Insert row                   | Respond to Webhook          |                                      |
| Respond to Webhook   | Respond to Webhook     | Sends response text to webhook caller        | MESSAGE, MESSAGE1, MESSAGE2, ERROR, ERROR1 |                             | Sticky Note5 (RESPONSE), Sticky Note7 (COMPLETED) |
| EVERY MONTH          | Schedule Trigger       | Triggers monthly analytics                    |                              | ANALYZE ENTIRE MONTH        | Sticky Note4 (MONTHLY SUMMARY)       |
| EVERY DAY            | Schedule Trigger       | Triggers daily clock-in reminders             |                              | ANALYZE TIME ENTRIES        | Sticky Note6 (CLOCK-IN REMINDER)     |
| ANALYZE ENTIRE MONTH | Data Table (Get)       | Retrieves all monthly schedule data           | EVERY MONTH                  | Message a model             |                                      |
| ANALYZE TIME ENTRIES | Data Table (Get)       | Retrieves all schedule data for daily check   | EVERY DAY                   | Message a model1            |                                      |
| Message a model      | Langchain OpenAI       | Generates monthly summary using GPT-4o        | ANALYZE ENTIRE MONTH         | EMAIL TO MANAGEMENT         |                                      |
| Message a model1     | Langchain OpenAI       | Generates daily clock-in reminder               | ANALYZE TIME ENTRIES         | EMAIL TO EMPLOYEE           |                                      |
| EMAIL TO MANAGEMENT  | Gmail (OAuth2)         | Sends monthly report email to management       | Message a model              |                             |                                      |
| EMAIL TO EMPLOYEE    | Gmail (OAuth2)         | Sends daily reminder email to employee          | Message a model1             |                             |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Webhook - Track Time`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `track-time`  
   - Response Mode: `responseNode` (to send response from downstream node)  

2. **Add Switch Node**  
   - Name: `Switch`  
   - Input: Connect from `Webhook - Track Time`  
   - Rules:  
     - Output "Start": condition `{{$json.body.method}}` equals `start`  
     - Output "Break": condition equals `break`  
     - Output "End": condition equals `end`  

3. **Insert Row Node (Start)**  
   - Name: `Insert row`  
   - Type: Data Table (Insert)  
   - Data Table ID: Use your schedule table ID  
   - Columns:  
     - `start`: set to current date (`{{$today}}`)  
     - `Employee`: from webhook header `id` (`{{$json.headers.id}}`)  
   - Connect from Switch output "Start"  

4. **Set MESSAGE2 Node**  
   - Name: `MESSAGE2`  
   - Type: Set  
   - Assign `message` = "Start time already tracked."  
   - Connect from `Insert row`  

5. **Get Row(s) Node (Break)**  
   - Name: `Get row(s)`  
   - Type: Data Table (Get)  
   - Data Table ID: schedule table  
   - Connect from Switch output "Break"  

6. **If1 Node**  
   - Name: `If1`  
   - Type: If  
   - Condition: Check if any row's `start` matches `$today`  
   - Connect from `Get row(s)`  

7. **Set Break Duration Node**  
   - Name: `Set Break Duration`  
   - Type: Set  
   - Assign `break_duration` from webhook body duration (`{{$json.body.duration}}`)  
   - Connect from If1 (true branch)  

8. **Update Row(s) Node**  
   - Name: `Update row(s)`  
   - Type: Data Table (Update)  
   - Data Table ID: schedule table  
   - Columns to update:  
     - `break`: from `Set Break Duration` `break_duration`  
     - `start`: from `$today`  
     - `Employee`: from webhook header `id`  
   - Filter: rows where `start` = `$today`  
   - Connect from `Set Break Duration`  

9. **Set MESSAGE1 Node**  
   - Name: `MESSAGE1`  
   - Type: Set  
   - Assign `message` = "Tracked {{ break_duration }} minutes as break time."  
   - Connect from `Update row(s)`  

10. **Get Row(s)2 Node (End)**  
    - Name: `Get row(s)2`  
    - Type: Data Table (Get)  
    - Data Table ID: schedule table  
    - Connect from Switch output "End"  

11. **If Node**  
    - Name: `If`  
    - Type: If  
    - Condition: Check if any row's `start` matches `$today`  
    - Connect from `Get row(s)2`  

12. **Set end Node**  
    - Name: `end`  
    - Type: Set  
    - Assign `end` = current timestamp (`{{$now}}`)  
    - Connect from If (true branch)  

13. **Update Row(s)1 Node**  
    - Name: `Update row(s)1`  
    - Type: Data Table (Update)  
    - Data Table ID: schedule table  
    - Columns:  
      - `end`: from `end` node  
      - `break`: keep existing from `Get row(s)2`  
      - `start`: `$today`  
      - `Employee`: from webhook header `id`  
    - Filter: rows where `start` = `$today`  
    - Connect from `end`  

14. **Set MESSAGE Node**  
    - Name: `MESSAGE`  
    - Type: Set  
    - Assign `message` = "Tracked {{ break_duration }} minutes as break time."  
    - Connect from `Update row(s)1`  

15. **Set ERROR and ERROR1 Nodes**  
    - Name: `ERROR` and `ERROR1`  
    - Type: Set  
    - Assign `message` = "ERROR"  
    - Connect ERROR from If1 (false branch), ERROR1 from If (false branch)  

16. **Respond to Webhook Node**  
    - Name: `Respond to Webhook`  
    - Type: Respond to Webhook  
    - Response Body: `{{$json.message}}`  
    - Connect from MESSAGE, MESSAGE1, MESSAGE2, ERROR, ERROR1 nodes  

17. **Schedule Trigger Nodes**  
    - Name: `EVERY MONTH`  
      - Type: Schedule Trigger  
      - Run monthly at 06:00  
    - Name: `EVERY DAY`  
      - Type: Schedule Trigger  
      - Run daily at 10:00  

18. **ANALYZE ENTIRE MONTH Node**  
    - Type: Data Table (Get)  
    - Data Table: schedule  
    - Return all rows  
    - Connect from EVERY MONTH  

19. **ANALYZE TIME ENTRIES Node**  
    - Type: Data Table (Get)  
    - Data Table: schedule  
    - Return all rows  
    - Connect from EVERY DAY  

20. **Message a model Node**  
    - Type: Langchain OpenAI  
    - Model: chatgpt-4o-latest  
    - Prompt: "Analyze the time tracking for last month and provide me with a summary\n\n{{$json}}"  
    - Credentials: OpenAI API  
    - Connect from ANALYZE ENTIRE MONTH  

21. **Message a model1 Node**  
    - Type: Langchain OpenAI  
    - Model: chatgpt-4o-latest  
    - Prompt: "IF I HAVEN'T CLOCKED IN TODAY, REMIND ME BY EMAIL\n\n{{$json}}"  
    - Credentials: OpenAI API  
    - Connect from ANALYZE TIME ENTRIES  

22. **EMAIL TO MANAGEMENT Node**  
    - Type: Gmail (OAuth2)  
    - To: `josea.castillodiez@gmail.com`  
    - Subject: "SCHEDULE"  
    - Message: from Message a model output content  
    - Credentials: Gmail OAuth2  
    - Connect from Message a model  

23. **EMAIL TO EMPLOYEE Node**  
    - Type: Gmail (OAuth2)  
    - To: `josea.castillodiez@gmail.com`  
    - Subject: "SCHEDULE"  
    - Message: from Message a model1 output content  
    - Credentials: Gmail OAuth2  
    - Connect from Message a model1  

24. **Add Sticky Notes**  
    - Add sticky notes as visual markers for START, BREAK, END, WAIT, RESPONSE, MONTHLY SUMMARY, CLOCK-IN REMINDER, COMPLETED blocks according to positions.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                              |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow uses GPT-4o via Langchain OpenAI node for AI-driven analysis and reminders.             | Requires OpenAI API key configured in credentials.           |
| Gmail nodes use OAuth2 credentials; ensure token refresh and quota limits are managed.           | Gmail OAuth2 credentials setup needed.                        |
| Data Table nodes rely on a schedule data table identified by ID; this must be created and shared.| Data table ID: `P4XPPCTMQJ4pWSU8` (replace with your own).  |
| The workflow is designed to respond immediately to webhook calls with status messages.           | Response mode in webhook set to respond from downstream node.|
| Sticky Notes provide clear visual documentation of logical sections within the editor interface. | Useful for maintenance and onboarding.                        |

---

**Disclaimer:**  
The provided text is exclusively generated from an n8n workflow automation. It complies fully with applicable content policies and contains no illegal or protected material. All processed data are legal and public.