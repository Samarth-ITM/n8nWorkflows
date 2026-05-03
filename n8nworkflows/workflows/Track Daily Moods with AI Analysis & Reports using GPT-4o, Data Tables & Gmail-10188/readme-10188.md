Track Daily Moods with AI Analysis & Reports using GPT-4o, Data Tables & Gmail

https://n8nworkflows.xyz/workflows/track-daily-moods-with-ai-analysis---reports-using-gpt-4o--data-tables---gmail-10188


# Track Daily Moods with AI Analysis & Reports using GPT-4o, Data Tables & Gmail

### 1. Workflow Overview

This workflow automates the tracking, analysis, and reporting of daily mood entries submitted via a webhook. It is designed for personal mood journaling with AI-enhanced summaries delivered by email. The core use case is to record mood data with optional notes, then generate weekly and monthly insights using GPT-4o, and finally send summarized reports through Gmail.

The workflow is logically divided into the following blocks:

- **1.1 Mood Data Reception and Storage**: Captures mood entries via a POST webhook, enriches with timestamp, and inserts records into a Data Table.
- **1.2 Weekly Summary Generation and Delivery**: Triggered every Sunday at 8:00 PM, it queries the last 7 days from the Data Table, runs an aggregation script, sends the summary to GPT-4o for AI analysis, and emails the results.
- **1.3 Monthly Summary Generation and Delivery**: Triggered monthly on the first day at 8:00 AM, it queries the last ~30 days, aggregates data similarly, requests GPT-4o analysis, and emails the report.
- **1.4 Webhook Response**: Sends an immediate JSON confirmation response after mood data submission.

Supporting sticky notes provide documentation within the workflow regarding workflow sections and configuration reminders.

---

### 2. Block-by-Block Analysis

#### 2.1 Mood Data Reception and Storage

- **Overview:**  
  This block receives mood submissions via an HTTP POST webhook, extracts and sets structured mood data including current date and time, and inserts the data as a new row into a configured Data Table.

- **Nodes Involved:**  
  - Webhook - Mood  
  - Set Mood Data  
  - Insert Mood Row  
  - Respond to Webhook1  
  - Sticky Note - Registro  
  - Sticky Note (configuration reminder)

- **Node Details:**  

  - **Webhook - Mood**  
    - *Type:* Webhook (Trigger)  
    - *Role:* Entry point receiving POST requests at the `/mood` path.  
    - *Configuration:* HTTP method POST; responseMode set to respond using a downstream node.  
    - *Expressions:* None.  
    - *Input:* External HTTP POST requests.  
    - *Output:* Passes incoming JSON body to next node.  
    - *Failures:* Could fail on invalid HTTP requests, network issues, or misconfiguration of webhook path.  

  - **Set Mood Data**  
    - *Type:* Set  
    - *Role:* Constructs a new JSON object with the current date (`yyyy-MM-dd`), current hour (`HH:mm`), and mood/note fields extracted from the webhook payload.  
    - *Expressions:*  
      - Date: `{{$now.toFormat('yyyy-MM-dd')}}`  
      - Hour: `{{$now.format('HH:mm')}}`  
      - Mood: `{{$json.body.mood}}`  
      - Note: `{{$json.body.note}}`  
    - *Input:* Webhook node output.  
    - *Output:* Structured mood entry JSON for storage.  
    - *Failures:* Expression evaluation issues if `$now` or `$json.body` is undefined.  

  - **Insert Mood Row**  
    - *Type:* Data Table (Insert Operation)  
    - *Role:* Inserts the structured mood data as a new row into the configured Data Table.  
    - *Configuration:* Data Table ID must be set by user before running. Columns: date, hour, mood, note.  
    - *Expressions:* Each column value is set based on previous node's JSON output fields.  
    - *Input:* Set Mood Data output.  
    - *Output:* Confirms insertion, passes data forward.  
    - *Failures:* Missing Data Table ID, authentication errors, schema mismatches.  

  - **Respond to Webhook1**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends a JSON response confirming the mood was saved.  
    - *Configuration:* Responds with JSON: `{"message": "Emotional state saved. Come back tomorrow 👌"}`  
    - *Input:* Insert Mood Row output.  
    - *Output:* HTTP response to client.  
    - *Failures:* Timeout or webhook response errors if the client disconnects early.  

  - **Sticky Note - Registro** and **Sticky Note (configuration reminder)**  
    - *Type:* Sticky Note  
    - *Role:* Document the mood log block and remind user to configure the Data Table ID in "Insert Mood Row".  
    - *Failures:* None.  

---

#### 2.2 Weekly Summary Generation and Delivery

- **Overview:**  
  This block triggers every Sunday at 8:00 PM, retrieves the last 7 days of mood data, aggregates and scores moods, requests GPT-4o to analyze the summary and moods with actionable insights, then emails the results.

- **Nodes Involved:**  
  - Schedule Weekly (SUNDAY 20:00)  
  - List Rows (Weekly)  
  - Aggregate (7d)  
  - ChatGPT Weekly Analysis  
  - Gmail (Weekly)  
  - Sticky Note - Weekly

- **Node Details:**  

  - **Schedule Weekly (SUNDAY 20:00)**  
    - *Type:* Schedule Trigger  
    - *Role:* Triggers the workflow weekly on Sundays at 20:00.  
    - *Configuration:* Weekly interval, trigger hour 20, no specific day set (assuming Sunday from sticky note).  
    - *Input:* None (trigger only).  
    - *Output:* Initiates the weekly branch.  
    - *Failures:* Scheduling misconfiguration, timezone issues.  

  - **List Rows (Weekly)**  
    - *Type:* Data Table (Get Operation)  
    - *Role:* Retrieves all rows from the Data Table for processing.  
    - *Configuration:* Return all rows; Data Table ID must be set by user.  
    - *Input:* Trigger from schedule node.  
    - *Output:* Passes all mood entries to aggregation.  
    - *Failures:* Missing or invalid Data Table ID, connectivity issues.  

  - **Aggregate (7d)**  
    - *Type:* Code (JavaScript)  
    - *Role:* Filters rows from last 7 days, counts mood occurrences, calculates average score, labels overall mood, and generates a simple text bar chart.  
    - *Key code logic:*  
      - Defines `parseDate` to safely parse date strings.  
      - Scores moods: 🙂=2, 😐=1, 😩=0.  
      - Creates summary text with counts, average, label, and bar chart.  
    - *Input:* Rows from List Rows (Weekly).  
    - *Output:* JSON with summary, counts, avg, label, and filtered rows.  
    - *Failures:* Date parsing errors if date field is malformed; empty datasets; runtime exceptions.  

  - **ChatGPT Weekly Analysis**  
    - *Type:* OpenAI (Langchain)  
    - *Role:* Sends the weekly summary and full data JSON to GPT-4o for generating an AI analysis with trends, triggers, and recommendations.  
    - *Configuration:* Model GPT-4o; message template includes summary and JSON stringified rows.  
    - *Credentials:* OpenAI API key required.  
    - *Input:* Aggregated data JSON.  
    - *Output:* AI-generated textual analysis.  
    - *Failures:* API authentication errors, rate limits, malformed prompts.  

  - **Gmail (Weekly)**  
    - *Type:* Gmail  
    - *Role:* Emails the automatic weekly summary and AI analysis to a configured email address.  
    - *Configuration:* Send to user email; subject "Weekly Mood Summary"; message concatenates aggregation summary and AI analysis content.  
    - *Credentials:* Gmail OAuth2 account.  
    - *Input:* AI analysis output.  
    - *Output:* Email sent confirmation.  
    - *Failures:* Gmail OAuth expiration, quota limits, invalid recipient email.  

  - **Sticky Note - Weekly**  
    - *Role:* Documents the weekly summary block and schedule.  
    - *Failures:* None.  

---

#### 2.3 Monthly Summary Generation and Delivery

- **Overview:**  
  Triggered on the 1st day of each month at 8:00 AM, this block retrieves mood data for the last ~30 days, performs similar aggregation and scoring, sends data to GPT-4o for a detailed monthly analysis, and emails the report.

- **Nodes Involved:**  
  - Schedule Monthly (1th, 08:00)  
  - List Rows (Monthly)  
  - Aggregate (~30d)  
  - ChatGPT Monthly Analysis  
  - Gmail (Monthly)  
  - Sticky Note - Monthly

- **Node Details:**  

  - **Schedule Monthly (1th, 08:00)**  
    - *Type:* Schedule Trigger  
    - *Role:* Fires monthly trigger on day 1 at 08:00.  
    - *Configuration:* Monthly interval, trigger hour 8.  
    - *Failures:* Similar to weekly schedule node; day/time misconfiguration.  

  - **List Rows (Monthly)**  
    - *Type:* Data Table (Get Operation)  
    - *Role:* Retrieves all rows from the Data Table.  
    - *Configuration:* Return all rows; same Data Table ID as weekly.  
    - *Failures:* Same as weekly listing.  

  - **Aggregate (~30d)**  
    - *Type:* Code (JavaScript)  
    - *Role:* Filters mood rows from the last 30 days, computes counts, average, label, and textual bar chart similarly to weekly aggregation.  
    - *Differences:* 30-day window instead of 7-day.  
    - *Failures:* Same as weekly aggregate node.  

  - **ChatGPT Monthly Analysis**  
    - *Type:* OpenAI (Langchain)  
    - *Role:* Sends monthly summary and data to GPT-4o, requesting detailed trends, notable correlations, and specific habits for the month.  
    - *Configuration:* GPT-4o model; detailed prompt tailored for monthly insights.  
    - *Credentials:* OpenAI API key.  
    - *Failures:* Same as weekly ChatGPT node.  

  - **Gmail (Monthly)**  
    - *Type:* Gmail  
    - *Role:* Sends the monthly report email with summary and AI analysis.  
    - *Configuration:* Subject "Monthly Mood Summary"; recipient email configured.  
    - *Credentials:* Gmail OAuth2.  
    - *Failures:* Same as weekly Gmail node.  

  - **Sticky Note - Monthly**  
    - *Role:* Documents the monthly summary block and schedule.  
    - *Failures:* None.  

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                              | Input Node(s)           | Output Node(s)                      | Sticky Note                                                      |
|--------------------------|---------------------------|----------------------------------------------|-------------------------|-----------------------------------|-----------------------------------------------------------------|
| Webhook - Mood           | Webhook                   | Receives POST mood data                       | -                       | Set Mood Data                     | Sticky Note - Registro: "Webhook POST /mood receives mood and optional note" |
| Set Mood Data            | Set                       | Structures mood data with date and hour      | Webhook - Mood           | Insert Mood Row                   | Sticky Note - Registro                                            |
| Insert Mood Row          | Data Table (Insert)       | Inserts mood entry into Data Table            | Set Mood Data            | Respond to Webhook1               | ⚠️ Configure your own Data Table ID in the "Insert Mood Row" node before running the workflow. |
| Respond to Webhook1      | Respond to Webhook        | Sends JSON confirmation response              | Insert Mood Row          | -                               | Sticky Note - Registro                                            |
| Schedule Weekly (SUNDAY 20:00) | Schedule Trigger         | Triggers weekly summary process                | -                       | List Rows (Weekly)                | Sticky Note - Weekly: "Sundays 8:00 PM - Analysis of the last 7 days" |
| List Rows (Weekly)       | Data Table (Get)          | Retrieves all mood rows for weekly summary    | Schedule Weekly          | Aggregate (7d)                   | Sticky Note - Weekly                                             |
| Aggregate (7d)           | Code                      | Aggregates last 7 days moods, scores, charts | List Rows (Weekly)       | ChatGPT Weekly Analysis           | Sticky Note - Weekly                                             |
| ChatGPT Weekly Analysis  | OpenAI (Langchain)        | AI analyzes weekly summary and data           | Aggregate (7d)           | Gmail (Weekly)                   | Sticky Note - Weekly                                             |
| Gmail (Weekly)           | Gmail                     | Emails weekly summary and AI analysis         | ChatGPT Weekly Analysis  | -                               | Sticky Note - Weekly                                             |
| Schedule Monthly (1th, 08:00) | Schedule Trigger         | Triggers monthly summary process               | -                       | List Rows (Monthly)               | Sticky Note - Monthly: "Day 1 of the month 8:00 AM - Analysis of the last 30 days" |
| List Rows (Monthly)      | Data Table (Get)          | Retrieves all mood rows for monthly summary   | Schedule Monthly         | Aggregate (~30d)                 | Sticky Note - Monthly                                            |
| Aggregate (~30d)         | Code                      | Aggregates last ~30 days moods, scores, charts| List Rows (Monthly)      | ChatGPT Monthly Analysis          | Sticky Note - Monthly                                            |
| ChatGPT Monthly Analysis | OpenAI (Langchain)        | AI analyzes monthly summary and data          | Aggregate (~30d)         | Gmail (Monthly)                  | Sticky Note - Monthly                                            |
| Gmail (Monthly)          | Gmail                     | Emails monthly summary and AI analysis         | ChatGPT Monthly Analysis | -                               | Sticky Note - Monthly                                            |
| Sticky Note - Registro   | Sticky Note               | Documents mood log block                       | -                       | -                               | Sticky Note - Registro                                            |
| Sticky Note - Weekly     | Sticky Note               | Documents weekly summary block                 | -                       | -                               | Sticky Note - Weekly                                             |
| Sticky Note - Monthly    | Sticky Note               | Documents monthly summary block                | -                       | -                               | Sticky Note - Monthly                                            |
| Sticky Note (config reminder) | Sticky Note           | Warns to configure Data Table ID in Insert Mood Row | -                       | -                               | ⚠️ Configure your own Data Table ID in the "Insert Mood Row" node before running the workflow. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Webhook - Mood`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `mood`  
   - Response Mode: `Response Node` (will respond from a downstream node)  

2. **Create Set Node**  
   - Name: `Set Mood Data`  
   - Type: Set  
   - Assign fields:  
     - `date`: `={{ $now.toFormat('yyyy-MM-dd') }}`  
     - `hour`: `={{ $now.format('HH:mm') }}`  
     - `mood`: `={{ $json.body.mood }}`  
     - `note`: `={{ $json.body.note }}`  
   - Connect input from `Webhook - Mood`  

3. **Create Data Table Insert Node**  
   - Name: `Insert Mood Row`  
   - Type: Data Table  
   - Operation: Insert  
   - Configure Data Table ID (must be created beforehand in n8n Data Tables)  
   - Define columns: `date`, `hour`, `mood`, `note`  
   - Map each column from `Set Mood Data` output JSON fields  
   - Connect input from `Set Mood Data`  

4. **Create Respond to Webhook Node**  
   - Name: `Respond to Webhook1`  
   - Type: Respond to Webhook  
   - Response Body (JSON): `{"message": "Emotional state saved. Come back tomorrow 👌"}`  
   - Connect input from `Insert Mood Row`  

5. **Create Weekly Schedule Trigger Node**  
   - Name: `Schedule Weekly (SUNDAY 20:00)`  
   - Type: Schedule Trigger  
   - Configure to trigger weekly on Sundays at 20:00 (verify correct day setting in n8n)  

6. **Create Data Table Get Node (Weekly)**  
   - Name: `List Rows (Weekly)`  
   - Type: Data Table  
   - Operation: Get  
   - Return All: true  
   - Use same Data Table ID as Insert node  
   - Connect input from `Schedule Weekly (SUNDAY 20:00)`  

7. **Create Code Node for Weekly Aggregation**  
   - Name: `Aggregate (7d)`  
   - Type: Code (JavaScript)  
   - Paste provided JS code for 7-day aggregation (see code block in analysis)  
   - Connect input from `List Rows (Weekly)`  

8. **Create OpenAI Node for Weekly Analysis**  
   - Name: `ChatGPT Weekly Analysis`  
   - Type: OpenAI (Langchain)  
   - Model: `gpt-4o`  
   - Message Content:  
     ```
     Analyze these moods from the past 7 days and generate a summary with trends, potential triggers (based on 'note'), and 3 actionable recommendations. Keep it short and sweet.

     {{$json.summary}}

     Full data:
     {{JSON.stringify($json.rows)}}
     ```  
   - Provide OpenAI API credentials  
   - Connect input from `Aggregate (7d)`  

9. **Create Gmail Node for Weekly Email**  
   - Name: `Gmail (Weekly)`  
   - Type: Gmail  
   - To: your.email@example.com (replace with target email)  
   - Subject: `Weekly Mood Summary`  
   - Message:  
     ```
     Automatic summary (text + graphic):

     {{ $('Aggregate (7d)').item.json.summary }}

     ---

     AI Analysis:

     {{ $json.message.content }}
     ```  
   - Provide Gmail OAuth2 credentials  
   - Connect input from `ChatGPT Weekly Analysis`  

10. **Create Monthly Schedule Trigger Node**  
    - Name: `Schedule Monthly (1th, 08:00)`  
    - Type: Schedule Trigger  
    - Configure to trigger monthly on day 1 at 08:00  

11. **Create Data Table Get Node (Monthly)**  
    - Name: `List Rows (Monthly)`  
    - Type: Data Table  
    - Operation: Get  
    - Return All: true  
    - Use same Data Table ID  
    - Connect input from `Schedule Monthly (1th, 08:00)`  

12. **Create Code Node for Monthly Aggregation**  
    - Name: `Aggregate (~30d)`  
    - Type: Code (JavaScript)  
    - Paste provided JS code for 30-day aggregation (see code block in analysis)  
    - Connect input from `List Rows (Monthly)`  

13. **Create OpenAI Node for Monthly Analysis**  
    - Name: `ChatGPT Monthly Analysis`  
    - Type: OpenAI (Langchain)  
    - Model: `gpt-4o`  
    - Message Content:  
      ```
      Analyze my moods from the past month. Summarize trends, notable days/times, correlations between 'note' and mood, and 3 specific habits to try this month. Be specific.

      {{$json.summary}}

      Full data:
      {{JSON.stringify($json.rows)}}
      ```  
    - Provide OpenAI API credentials  
    - Connect input from `Aggregate (~30d)`  

14. **Create Gmail Node for Monthly Email**  
    - Name: `Gmail (Monthly)`  
    - Type: Gmail  
    - To: your.email@example.com (replace accordingly)  
    - Subject: `Monthly Mood Summary`  
    - Message:  
      ```
      Automatic summary (text + graphic):

      {{ $('Aggregate (~30d)').item.json.summary }}

      ---

      AI Analysis:

      {{ $json.message.content }}
      ```  
    - Provide Gmail OAuth2 credentials  
    - Connect input from `ChatGPT Monthly Analysis`  

15. **Create Sticky Notes** (Optional for documentation)  
    - Add notes for mood log, weekly summary, monthly summary, and Data Table configuration reminder as per original content.  

16. **Configure Credentials**  
    - Set up OpenAI API credentials with valid API key (required for GPT-4o).  
    - Set up Gmail OAuth2 credentials for sending emails.  

17. **Create and Configure Data Table**  
    - Create a Data Table with columns: `date` (string), `hour` (string), `mood` (string), `note` (string).  
    - Copy the Data Table ID into both Insert and List Rows nodes.  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                        |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| ⚠️ Configure your own Data Table ID in the "Insert Mood Row" node before running the workflow. | Critical setup step to enable data storage.                          |
| Workflow leverages GPT-4o model for advanced AI mood analysis and actionable recommendations.  | Requires OpenAI account and API key with access to GPT-4o.           |
| Emails are sent via Gmail OAuth2; ensure OAuth credentials are valid and have necessary scopes.| Gmail node setup and OAuth consent required.                         |
| The workflow uses n8n Data Tables feature for structured mood data storage.                    | Data Table creation and management documentation in n8n docs.       |
| Timezone considerations: schedule triggers depend on n8n instance timezone settings.          | Verify instance timezone to align schedule triggers correctly.      |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.