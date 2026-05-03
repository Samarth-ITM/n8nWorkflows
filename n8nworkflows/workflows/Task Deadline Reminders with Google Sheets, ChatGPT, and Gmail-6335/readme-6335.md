Task Deadline Reminders with Google Sheets, ChatGPT, and Gmail

https://n8nworkflows.xyz/workflows/task-deadline-reminders-with-google-sheets--chatgpt--and-gmail-6335


# Task Deadline Reminders with Google Sheets, ChatGPT, and Gmail

### 1. Workflow Overview

This workflow, titled **Deadline Reminder Bot**, automates the process of sending personalized email reminders for tasks due on the current day. It is primarily designed for teams managing tasks in a Google Sheets document and wanting automated, professional, and motivating deadline reminders sent via email. The workflow integrates Google Sheets to fetch task data, OpenAI’s ChatGPT to generate personalized summary messages, and Gmail to send the notifications.

The workflow logic can be grouped into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow daily at a specified hour.
- **1.2 Data Retrieval from Google Sheets**: Fetches all task rows from a designated Google Sheet.
- **1.3 Filtering Tasks Due Today**: Filters rows to only include tasks whose due date matches the current date.
- **1.4 Summarization by Assignee**: Groups tasks by assignee, aggregating task names, statuses, and emails.
- **1.5 AI Message Generation**: Uses OpenAI’s ChatGPT to craft personalized reminder emails for each assignee.
- **1.6 Email Sending**: Sends the generated messages through Gmail to the respective assignees.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically every day at 9 AM. It acts as the entry point for the workflow execution.

- **Nodes Involved:**  
  - Schedule Trigger1

- **Node Details:**

  - **Schedule Trigger1**  
    - *Type:* Schedule Trigger  
    - *Technical Role:* Automatically triggers workflow execution on a time-based schedule.  
    - *Configuration:* Set to trigger daily at 9:00 AM (hour 9).  
    - *Expressions/Variables:* None.  
    - *Input Connections:* None (start node).  
    - *Output Connections:* Feeds into "Get row(s) in sheet1".  
    - *Version-specific Requirements:* Requires n8n version supporting scheduleTrigger v1.2 or above.  
    - *Edge Cases / Failure Modes:*  
      - Workflow won’t trigger if n8n instance is down or paused.  
      - Timezone configuration in n8n must be correct to ensure intended trigger time.  
    - *Sub-workflow:* None.

#### 1.2 Data Retrieval from Google Sheets

- **Overview:**  
  Retrieves all task rows from a specified Google Sheets document and sheet.

- **Nodes Involved:**  
  - Get row(s) in sheet1

- **Node Details:**

  - **Get row(s) in sheet1**  
    - *Type:* Google Sheets  
    - *Technical Role:* Reads rows from a Google Sheet document.  
    - *Configuration:*  
      - Document ID and Sheet Name are placeholders ("YOUR_SHEET_ID" and "YOUR_SHEET_NAME") and must be replaced with actual values.  
      - No additional options (filters or ranges) are specified, so it likely fetches all rows.  
    - *Expressions/Variables:* None directly in parameters.  
    - *Input Connections:* Receives trigger from Schedule Trigger1.  
    - *Output Connections:* Connects to the If1 node for filtering.  
    - *Credential:* Uses Google Sheets OAuth2 for authentication (configured with “Google Sheets – Team Account”).  
    - *Version-specific Requirements:* Uses Google Sheets node v4.6; ensure compatibility with OAuth2 credentials.  
    - *Edge Cases / Failure Modes:*  
      - Invalid or missing Document ID or Sheet Name will cause failure.  
      - OAuth token expiration or permission issues can cause auth errors.  
      - Large sheets may cause timeouts or slow performance.  
    - *Sub-workflow:* None.

#### 1.3 Filtering Tasks Due Today

- **Overview:**  
  Filters retrieved rows to only those tasks whose "Due Date" field equals today’s date.

- **Nodes Involved:**  
  - If1

- **Node Details:**

  - **If1**  
    - *Type:* If (Conditional)  
    - *Technical Role:* Filters data items based on condition.  
    - *Configuration:*  
      - Condition uses strict comparison (case-sensitive, type-validation strict).  
      - Compares the "Due Date" field of each row to the current date formatted as `yyyy-MM-dd`.  
      - Expression: `{{$json['Due Date']}} === {{$now.format('yyyy-MM-dd')}}`  
    - *Expressions/Variables:* Uses n8n date formatting function `$now.format()`.  
    - *Input Connections:* From "Get row(s) in sheet1".  
    - *Output Connections:* Successful condition (true branch) leads to "Summarize1".  
    - *Version-specific Requirements:* Requires n8n supporting If node v2.2 with expression evaluation.  
    - *Edge Cases / Failure Modes:*  
      - Tasks missing "Due Date" field or with improperly formatted dates will be excluded or cause errors.  
      - Timezone discrepancies may cause mismatches.  
      - If no tasks match, no output passes forward, possibly halting downstream nodes.  
    - *Sub-workflow:* None.

#### 1.4 Summarization by Assignee

- **Overview:**  
  Aggregates filtered tasks by "Assignee", collecting task names, emails, and statuses into grouped summaries per person.

- **Nodes Involved:**  
  - Summarize1

- **Node Details:**

  - **Summarize1**  
    - *Type:* Summarize  
    - *Technical Role:* Groups data by specified fields and aggregates others.  
    - *Configuration:*  
      - Groups by "Assignee" field.  
      - Aggregates the "Task" field by appending task names into a list.  
      - Extracts the minimum (first) "Email" per assignee.  
      - Appends the "Status" field values.  
      - Output format: Separate items for each group (each assignee gets one summarized item).  
    - *Expressions/Variables:* Uses field names for grouping and aggregation.  
    - *Input Connections:* Receives filtered rows from If1.  
    - *Output Connections:* Passes summarized data to "Message a model1".  
    - *Version-specific Requirements:* Summarize node v1.1 or higher.  
    - *Edge Cases / Failure Modes:*  
      - Missing or inconsistent "Assignee", "Email", or "Task" fields can reduce summarization quality.  
      - If no input, no output items are generated.  
    - *Sub-workflow:* None.

#### 1.5 AI Message Generation

- **Overview:**  
  Sends summarized task data to OpenAI ChatGPT (via Langchain node) to generate a personalized, professional, and motivating email for each assignee.

- **Nodes Involved:**  
  - Message a model1

- **Node Details:**

  - **Message a model1**  
    - *Type:* OpenAI (Langchain)  
    - *Technical Role:* Uses ChatGPT to generate text messages based on prompt context.  
    - *Configuration:*  
      - Model: "chatgpt-4o-latest" (latest GPT-4 model).  
      - Message prompt instructs the assistant to write a short task summary email for the assignee, including their email address, listing tasks and statuses due today, with clear, professional, and motivating tone.  
      - Prompt uses expressions to inject JSON fields:  
        - `{{ $json["Assignee"] }}`, `{{ $json.min_Email }}`, `{{ $json.appended_Task }}`, `{{ $json.appended_Status }}`  
      - Output format: JSON with generated message content.  
    - *Expressions/Variables:* Uses dynamic expressions within the prompt to customize message per assignee.  
    - *Input Connections:* Receives summarized items from Summarize1.  
    - *Output Connections:* Passes generated messages to "Send a message1".  
    - *Credential:* Uses OpenAI API credentials named “OpenAI – Assistant API”.  
    - *Version-specific Requirements:* Requires OpenAI node v1.8 with Langchain support.  
    - *Edge Cases / Failure Modes:*  
      - API rate limits or connectivity issues may cause failures.  
      - Prompt injection or unexpected input data might yield poor message quality.  
      - Missing or malformed assignee data could cause prompt errors.  
    - *Sub-workflow:* None.

#### 1.6 Email Sending

- **Overview:**  
  Sends the personalized reminder emails to each assignee using Gmail.

- **Nodes Involved:**  
  - Send a message1

- **Node Details:**

  - **Send a message1**  
    - *Type:* Gmail  
    - *Technical Role:* Sends email messages via Gmail SMTP using OAuth2 authentication.  
    - *Configuration:*  
      - Sends to address extracted from AI-generated message content (`{{$json.message.content.to}}`).  
      - Email subject and body are dynamically populated from AI-generated content (`{{$json.message.content.subject}}` and `{{$json.message.content.body}}`).  
      - No additional options configured.  
    - *Expressions/Variables:* Dynamic expressions for `sendTo`, `subject`, and `message`.  
    - *Input Connections:* From "Message a model1".  
    - *Output Connections:* None (final node).  
    - *Credential:* Gmail OAuth2 credential named “Gmail – Notification Account”.  
    - *Version-specific Requirements:* Gmail node v2.1 or newer to support OAuth2.  
    - *Edge Cases / Failure Modes:*  
      - OAuth token expiration or insufficient Gmail API permissions may cause send failures.  
      - Invalid recipient email addresses or malformed message content can cause errors.  
      - Gmail sending limits may throttle or block the emails if volume is high.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                      | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                       |
|---------------------|---------------------------|------------------------------------|------------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note         | Sticky Note               | Documentation                      | None                   | None                  | Deadline Reminder Bot<br> This workflow will:<br>📅 Check for tasks due today → 🧠 Summarize using OpenAI → 📧 Send personalized reminder to each person |
| Schedule Trigger1    | Schedule Trigger          | Workflow trigger at 9 AM daily     | None                   | Get row(s) in sheet1   |                                                                                                 |
| Get row(s) in sheet1 | Google Sheets             | Retrieve all task rows             | Schedule Trigger1      | If1                   |                                                                                                 |
| If1                 | If (Conditional)          | Filter tasks due today             | Get row(s) in sheet1   | Summarize1            |                                                                                                 |
| Summarize1          | Summarize                 | Group tasks by assignee            | If1                    | Message a model1       |                                                                                                 |
| Message a model1     | OpenAI (Langchain)        | Generate personalized emails       | Summarize1             | Send a message1        |                                                                                                 |
| Send a message1      | Gmail                     | Send emails to assignees           | Message a model1       | None                  |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note**  
   - Type: Sticky Note  
   - Content:  
     ```
     Deadline Reminder Bot

     This workflow will:

     📅 Check for tasks due today → 🧠 Summarize using OpenAI → 📧 Send personalized reminder to each person
     ```  
   - Position accordingly for documentation.

2. **Create Schedule Trigger Node**  
   - Name: Schedule Trigger1  
   - Type: Schedule Trigger  
   - Configure trigger rule:  
     - Trigger at hour: 9 (daily at 9 AM)  
   - No credentials needed.  
   - Connect output to the Google Sheets node.

3. **Create Google Sheets Node**  
   - Name: Get row(s) in sheet1  
   - Type: Google Sheets  
   - Credentials: Configure with a Google Sheets OAuth2 credential that has read access to the target sheet (e.g., “Google Sheets – Team Account”).  
   - Parameters:  
     - Document ID: Set the actual Google Sheet ID containing your task list.  
     - Sheet Name: Set to your task sheet name (e.g., “Tasks”).  
     - Leave other options default to fetch all rows.  
   - Connect output to the If node.

4. **Create If Node for Filtering**  
   - Name: If1  
   - Type: If (Conditional)  
   - Condition Type: String equals (strict, case sensitive)  
   - Left Value: Expression → `{{$json["Due Date"]}}`  
   - Right Value: Expression → `{{$now.format("yyyy-MM-dd")}}`  
   - Connect output (true) to Summarize node.

5. **Create Summarize Node**  
   - Name: Summarize1  
   - Type: Summarize  
   - Parameters:  
     - Output format: Separate Items  
     - Group by field: "Assignee"  
     - Fields to summarize:  
       - Task: Append (concatenate all tasks per assignee)  
       - Email: Minimum (to get single email per assignee)  
       - Status: Append  
   - Connect output to OpenAI node.

6. **Create OpenAI Node**  
   - Name: Message a model1  
   - Type: OpenAI (Langchain)  
   - Credentials: Configure with OpenAI API key credential (e.g., “OpenAI – Assistant API”).  
   - Parameters:  
     - Model ID: `chatgpt-4o-latest`  
     - Messages: Add one message of role `assistant` with content:  
       ```
       You are a productivity assistant. Write a short task summary email for {{ $json["Assignee"] }}.
       also write to and put their email {{ $json.min_Email }} into it.
       Here are their tasks due today:{{ $json.appended_Task }}
       Statuses:{{ $json.appended_Status }}
       Make it clear, professional, and motivating.
       ```  
     - JSON output enabled to parse generated message content.  
   - Connect output to Gmail node.

7. **Create Gmail Node**  
   - Name: Send a message1  
   - Type: Gmail  
   - Credentials: Configure with Gmail OAuth2 credential authorized to send emails (e.g., “Gmail – Notification Account”).  
   - Parameters:  
     - Send To: Expression → `{{$json.message.content.to}}`  
     - Subject: Expression → `{{$json.message.content.subject}}`  
     - Message: Expression → `{{$json.message.content.body}}`  
   - No additional options required.  
   - No output connection (end node).

8. **Connect the Nodes in Order:**  
   - Schedule Trigger1 → Get row(s) in sheet1 → If1 → Summarize1 → Message a model1 → Send a message1

9. **Replace Placeholder Values:**  
   - Google Sheets Document ID and Sheet Name must point to your actual data source.  
   - Ensure all credentials (Google Sheets OAuth2, OpenAI API, Gmail OAuth2) are properly configured and authorized.

10. **Test the Workflow:**  
    - Run manually or wait for scheduled time.  
    - Validate emails are sent with appropriate content.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                              |
|-------------------------------------------------------------------------------------------------|----------------------------------------------|
| The workflow sends personalized deadline reminders by combining Google Sheets, OpenAI, and Gmail.| Branding and workflow purpose                 |
| Replace `"YOUR_SHEET_ID"` and `"YOUR_SHEET_NAME"` with your actual Google Sheets identifiers.   | Google Sheets integration detail              |
| The OpenAI prompt is structured to produce professional and motivating emails per assignee.     | OpenAI prompt design                           |
| Ensure timezones in n8n settings align with your local timezone to avoid date mismatches.       | n8n scheduling and date handling              |
| Requires OAuth2 credentials for Google Sheets and Gmail with appropriate scopes.                 | Credential setup instructions                  |
| OpenAI API usage may incur cost; monitor usage and rate limits accordingly.                      | OpenAI API usage considerations                |
| Gmail sending limits may apply; for large teams consider batching or alternative sending setups.| Gmail sending quota considerations             |

---

This document fully describes the Deadline Reminder Bot workflow, enabling advanced users or AI agents to understand, reproduce, or modify it confidently.