Email Summary Agent 

https://n8nworkflows.xyz/workflows/email-summary-agent--2722


# Email Summary Agent 

### 1. Workflow Overview

The **Email Summary Agent** workflow automates the process of managing email overload for teams by fetching recent emails, summarizing key points and action items using AI, and sending concise summary emails twice daily. This helps reduce missed actions and improves meeting preparation efficiency.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow at a specified time (7 AM daily).
- **1.2 Email Retrieval:** Fetches all emails received in the past 24 hours from a specified Gmail account.
- **1.3 Data Organization:** Extracts and organizes relevant email fields such as sender, recipients, CC, and snippets.
- **1.4 AI Summarization:** Uses OpenAI’s GPT-4o-mini model to analyze the email data and generate structured summaries and action items.
- **1.5 Summary Email Dispatch:** Sends a styled HTML email containing the summarized information to a team mailing list.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow automatically every day at 7 AM to start the email summarization process.

- **Nodes Involved:**  
  - Daily 7AM Trigger  
  - Sticky Note (contextual comment)

- **Node Details:**

  - **Daily 7AM Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution daily at 7 AM.  
    - Configuration: Set to trigger at hour 7 (7 AM) every day.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Fetch Emails - Past 24 Hours" node.  
    - Edge Cases: If the n8n instance is down or paused at trigger time, the workflow will not start. Timezone is set to Asia/Kolkata, so ensure this matches your local time requirements.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides user instructions to adjust the trigger time if needed.  
    - Content: "- Starts the workflow every day at 7 AM. - Adjust the time if you want the workflow to run at a different hour."

---

#### 2.2 Email Retrieval

- **Overview:**  
  Fetches all emails received in the last 24 hours from a specific Gmail account using Gmail OAuth2 credentials.

- **Nodes Involved:**  
  - Fetch Emails - Past 24 Hours  
  - Sticky Note1 (contextual comment)

- **Node Details:**

  - **Fetch Emails - Past 24 Hours**  
    - Type: Gmail node (v2.1)  
    - Role: Retrieves emails from the Gmail inbox filtered by date and sender.  
    - Configuration:  
      - Operation: Get all emails.  
      - Filter Query: Uses Gmail search syntax to fetch emails from "isb.quantana@quantana.in" received after the date exactly 24 hours ago. The date is dynamically calculated using JavaScript expressions.  
      - Return All: True (fetches all matching emails).  
      - Credentials: Uses Gmail OAuth2 credentials named "Vishal Gmail".  
    - Inputs: Triggered by "Daily 7AM Trigger".  
    - Outputs: Passes data to "Organize Email Data - Morning".  
    - Edge Cases:  
      - Gmail API rate limits or authentication errors may cause failures.  
      - If no emails are found, downstream nodes should handle empty data gracefully.  
      - Timezone differences may affect the "after" date filter if not aligned.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains that this node fetches all emails received in the past 24 hours from the specified email address.

---

#### 2.3 Data Organization

- **Overview:**  
  Aggregates and extracts key fields from the fetched emails to prepare data for AI summarization.

- **Nodes Involved:**  
  - Organize Email Data - Morning  
  - Sticky Note2 (contextual comment)

- **Node Details:**

  - **Organize Email Data - Morning**  
    - Type: Aggregate node  
    - Role: Extracts specific fields from all fetched emails into a consolidated dataset.  
    - Configuration:  
      - Aggregate Mode: Aggregate all item data.  
      - Fields Included: id, From, To, CC, snippet (email preview).  
      - Options: Default.  
    - Inputs: Receives raw email data from "Fetch Emails - Past 24 Hours".  
    - Outputs: Sends organized data to "Summarize Emails with OpenAI - Morning".  
    - Edge Cases:  
      - If input data is empty, output will be empty; AI summarization should handle this gracefully.  
      - Missing fields in some emails may cause partial data; ensure AI prompt can handle incomplete data.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Describes that this node organizes fetched email data, extracting sender, receiver, CC, and snippet fields.

---

#### 2.4 AI Summarization

- **Overview:**  
  Uses OpenAI GPT-4o-mini to analyze the organized email data and produce a structured summary of key points and action items.

- **Nodes Involved:**  
  - Summarize Emails with OpenAI - Morning

- **Node Details:**

  - **Summarize Emails with OpenAI - Morning**  
    - Type: OpenAI node (LangChain integration)  
    - Role: Processes email data to generate a JSON-formatted summary and list of actions.  
    - Configuration:  
      - Model: GPT-4o-mini (a smaller GPT-4 variant).  
      - Input Message: A prompt instructing the model to extract key details, issues, and action items from the email summary.  
      - Input Data: Injects the JSON string of the organized email data (`{{ $json.data.toJsonString() }}`).  
      - Output: JSON output enabled to parse the model’s structured response.  
      - Credentials: Uses OpenAI API credentials named "OpenAi account".  
    - Inputs: Receives organized email data from "Organize Email Data - Morning".  
    - Outputs: Passes AI-generated summary to "Send Summary - Morning".  
    - Edge Cases:  
      - API rate limits or authentication errors may cause failures.  
      - Model may produce unexpected output if input data is malformed or empty.  
      - Network timeouts or latency can delay execution.  
      - Ensure prompt is clear to avoid ambiguous or incomplete summaries.

---

#### 2.5 Summary Email Dispatch

- **Overview:**  
  Sends a styled HTML email to the team containing the AI-generated summary and action items.

- **Nodes Involved:**  
  - Send Summary - Morning  
  - Sticky Note4 (contextual comment)

- **Node Details:**

  - **Send Summary - Morning**  
    - Type: Gmail node (v2.1)  
    - Role: Sends an email with the summarized content formatted in HTML.  
    - Configuration:  
      - Send To: "team-email@example.com" (replace with actual team email).  
      - CC List: "cc-list@example.com" (replace with actual CC emails).  
      - Subject: Dynamic subject line showing the date range covered (from 00:00 of previous day to 7:00 AM current day).  
      - Message: HTML email template embedding the summary and actions using n8n expressions to iterate over the AI output arrays.  
      - Options: No attribution appended, reply-to sender disabled.  
      - Credentials: Uses Gmail OAuth2 credentials named "Vishal Gmail".  
    - Inputs: Receives AI summary from "Summarize Emails with OpenAI - Morning".  
    - Outputs: None (end node).  
    - Edge Cases:  
      - Email sending failures due to SMTP or Gmail API issues.  
      - Malformed HTML or empty summary may produce poor email formatting.  
      - Ensure recipient emails are valid to avoid bounce backs.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Explains the purpose of the node and reminds to update recipient email addresses.

---

### 3. Summary Table

| Node Name                     | Node Type                     | Functional Role                 | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                  |
|-------------------------------|-------------------------------|--------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Daily 7AM Trigger              | Schedule Trigger              | Starts workflow daily at 7 AM  | None                         | Fetch Emails - Past 24 Hours   | - Starts the workflow every day at 7 AM. - Adjust the time if you want the workflow to run at a different hour. |
| Fetch Emails - Past 24 Hours  | Gmail                        | Fetches emails from last 24h   | Daily 7AM Trigger            | Organize Email Data - Morning  | Fetches all emails received in the past 24 hours from the email address                      |
| Organize Email Data - Morning | Aggregate                    | Extracts key email fields      | Fetch Emails - Past 24 Hours | Summarize Emails with OpenAI - Morning | Organizes the fetched email data, extracting fields like sender, receiver, CC, and a preview snippet. |
| Summarize Emails with OpenAI - Morning | OpenAI (LangChain)           | Generates summary & actions    | Organize Email Data - Morning | Send Summary - Morning         |                                                                                              |
| Send Summary - Morning         | Gmail                        | Sends summary email to team    | Summarize Emails with OpenAI - Morning | None                          | - Sends the summarized email report to recipients with a styled HTML layout. - Update the "sendTo" and "ccList" fields with the email addresses of your recipients. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 7 AM (hour = 7)  
   - Set timezone to Asia/Kolkata or your preferred timezone.

2. **Create a Gmail Node to Fetch Emails**  
   - Type: Gmail (v2.1)  
   - Operation: Get All  
   - Filters: Use Gmail search query to fetch emails from the last 24 hours for a specific email address.  
     Example query:  
     ```
     isb.quantana@quantana.in after:YYYY/MM/DD
     ```  
     where `YYYY/MM/DD` is dynamically calculated as yesterday’s date using JavaScript expression:  
     ```js
     (() => {
       const yesterday = new Date();
       yesterday.setDate(yesterday.getDate() - 1);
       return `isb.quantana@quantana.in after:${yesterday.getFullYear()}/${(yesterday.getMonth() + 1).toString().padStart(2, '0')}/${yesterday.getDate().toString().padStart(2, '0')}`;
     })()
     ```  
   - Return All: True  
   - Credentials: Configure Gmail OAuth2 credentials.

3. **Create an Aggregate Node to Organize Email Data**  
   - Type: Aggregate  
   - Aggregate Mode: Aggregate all item data  
   - Fields to Include: `id`, `From`, `To`, `CC`, `snippet`  
   - Input: Connect from Gmail fetch node.

4. **Create an OpenAI Node for Summarization**  
   - Type: OpenAI (LangChain)  
   - Model: GPT-4o-mini  
   - Input Message:  
     ```
     Go through this email summary and identify all key details mentioned, any specific issues to look at, and action items.
     Use this format to output
     {
       "summary_of_emails": [
         "Point 1",
         "Point 2",
         "Point 3"
       ],
       "actions": [
         {
           "name": "Name 1",
           "action": "Action 1"
         },
         {
           "name": "Name 1",
           "action": "Action 2"
         },
         {
           "name": "Name 2",
           "action": "Action 3"
         }
       ]
     }

     Input Data:

     {{ $json.data.toJsonString() }}
     ```  
   - Enable JSON output parsing.  
   - Credentials: Configure OpenAI API credentials.

5. **Create a Gmail Node to Send Summary Email**  
   - Type: Gmail (v2.1)  
   - Send To: Set to your team’s email address (e.g., team-email@example.com)  
   - CC List: Set to desired CC recipients (e.g., cc-list@example.com)  
   - Subject: Use dynamic expression to show date range, e.g.:  
     ```
     ESAgent - {{ new Date(new Date().setDate(new Date().getDate() - 1)).toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' }) }}-00:00 to {{ new Date().toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' }) }}-07:00AM
     ```  
   - Message: Use an HTML template embedding the AI summary and actions with n8n expressions, iterating over arrays to build lists.  
   - Options: Disable attribution and reply-to sender only.  
   - Credentials: Use Gmail OAuth2 credentials.

6. **Connect Nodes in Sequence:**  
   - Schedule Trigger → Fetch Emails → Organize Email Data → Summarize with OpenAI → Send Summary Email

7. **Set Workflow Timezone and Execution Settings:**  
   - Timezone: Asia/Kolkata (or your local timezone)  
   - Execution Order: Default (v1)  
   - Caller Policy: workflowsFromSameOwner

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow is designed to reduce email overload by summarizing emails twice daily (7 AM shown).   | Adjust schedule trigger times to add a second run at 9 PM or other desired times.                  |
| The email summary includes both key points and actionable items extracted by AI.                    | Useful for team meeting prep and follow-up management.                                            |
| Update the Gmail node’s filter query and recipient email addresses to match your organization’s needs. | Ensure Gmail OAuth2 credentials have appropriate scopes for reading and sending emails.            |
| The HTML email template uses inline CSS for styling and dynamic content insertion via n8n expressions. | Customize the template to match your branding or formatting preferences.                           |
| OpenAI GPT-4o-mini is used for cost-effective summarization; consider upgrading model for higher accuracy if needed. | Requires valid OpenAI API credentials with sufficient quota.                                      |
| Workflow timezone is set to Asia/Kolkata; adjust if your team operates in a different timezone.     |                                                                                                   |
| For more information on n8n Gmail node configuration, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ |                                                                                                   |
| For OpenAI node details and prompt design, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/ |                                                                                                   |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and modify the Email Summary Agent workflow effectively.