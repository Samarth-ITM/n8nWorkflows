Schedule Supplier Follow-ups from Airtable POs to Google Calendar with AI, Slack & Gmail

https://n8nworkflows.xyz/workflows/schedule-supplier-follow-ups-from-airtable-pos-to-google-calendar-with-ai--slack---gmail-11297


# Schedule Supplier Follow-ups from Airtable POs to Google Calendar with AI, Slack & Gmail

### 1. Workflow Overview

This automated workflow is designed to streamline supplier follow-up scheduling based on purchase orders (POs) stored in Airtable. It targets procurement teams who need to efficiently monitor overdue POs and trigger timely follow-ups with suppliers. The workflow runs every weekday at 10 AM and performs the following logical blocks:

- **1.1 Trigger & Data Fetch**: Scheduled trigger initiates the workflow and fetches open purchase orders from Airtable that are older than 30 days and have no existing follow-up scheduled.

- **1.2 AI Processing & Validation**: For each PO, filters for valid supplier emails and then uses GPT-4 AI to generate a professional, concise meeting agenda and next steps for the supplier follow-up.

- **1.3 Calendar Event Creation & Notifications**: Creates Google Calendar events with the AI-generated agendas, updates Airtable with follow-up links and statuses, and sends notifications through Slack and Gmail.

- **1.4 Error Monitoring**: Monitors critical nodes for errors and sends Slack alerts for rapid debugging to ensure workflow reliability.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Data Fetch

**Overview:**  
This block triggers the workflow daily on weekdays at 10 AM and retrieves the relevant purchase orders from Airtable that require supplier follow-up.

**Nodes Involved:**  
- Schedule Trigger  
- Fetch Overdue POs from Airtable  
- Process Each PO Individually  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow on schedule  
  - Configuration: Cron expression `0 10 * * 1-5` (10:00 AM Monday to Friday)  
  - Inputs: None  
  - Outputs: Connects to "Fetch Overdue POs from Airtable"  
  - Edge Cases: Misconfigured cron could cause missed or extra runs

- **Fetch Overdue POs from Airtable**  
  - Type: Airtable Node  
  - Role: Fetches open POs older than 30 days without follow-up links  
  - Configuration: Uses a filter formula:
    ```
    AND(
      {Status} = "Open",
      IS_BEFORE({PO Date}, DATEADD(TODAY(), -30, 'days')),
      OR(
        {Follow-up Link} = "",
        {Follow-up Link} = BLANK()
      )
    )
    ```
  - Inputs: From Schedule Trigger  
  - Outputs: List of POs to "Process Each PO Individually"  
  - Credentials: Airtable Personal Access Token  
  - Edge Cases: Airtable API key invalid or missing; empty results if no matching records

- **Process Each PO Individually**  
  - Type: SplitInBatches  
  - Role: Processes each PO separately for detailed handling  
  - Inputs: List of POs from Airtable fetch  
  - Outputs: Branches to initial Slack alert and email validation  
  - Edge Cases: Large batch sizes could cause delays or timeouts

---

#### 1.2 AI Processing & Validation

**Overview:**  
Validates supplier email presence, sends PO details to GPT-4 AI to generate a structured follow-up agenda and next steps, parsing AI output into structured JSON.

**Nodes Involved:**  
- Initial Slack Alert  
- If (Email Validation)  
- Send a message (Slack alert for missing email)  
- AI Memory: Candidate Context  
- AI Model: GPT-4 Screening Engine  
- Parse AI Output (Structured JSON)  
- AI Agent: Agent Writer  

**Node Details:**

- **Initial Slack Alert**  
  - Type: Slack Node (webhook)  
  - Role: Sends preliminary notification that a supplier follow-up is triggered  
  - Configuration: Posts PO ID, supplier, email, PO date, and placeholder for calendar link  
  - Inputs: From "Process Each PO Individually"  
  - Outputs: Connects to "If" node  
  - Edge Cases: Slack webhook misconfigured or channel ID incorrect

- **If (Email Validation)**  
  - Type: If Node  
  - Role: Checks if supplier email field is non-empty  
  - Condition: `Supplier Email` field is not empty  
  - Inputs: From "Process Each PO Individually"  
  - Outputs:  
    - True: Proceed to AI processing  
    - False: Send Slack alert about missing email  
  - Edge Cases: Missing or malformed email fields

- **Send a message**  
  - Type: Slack Node (webhook)  
  - Role: Sends alert to Slack channel if supplier email is missing  
  - Inputs: From "If" (false branch)  
  - Outputs: Ends branch  
  - Edge Cases: Slack API failures

- **AI Memory: Candidate Context**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation context with AI by session key "AI Agenda"  
  - Inputs: From "AI Model: GPT-4 Screening Engine" (ai_memory channel)  
  - Outputs: To "AI Agent: Agent Writer" (ai_memory channel)  
  - Edge Cases: Memory buffer overflow or session key collisions

- **AI Model: GPT-4 Screening Engine**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Invokes GPT-4 model to process input data  
  - Configuration: Model "gpt-4o-mini" with temperature 0.7 for creativity  
  - Credentials: OpenAI API Key  
  - Inputs: From "If" (true branch) as main data, also feeds memory  
  - Outputs: To "Parse AI Output (Structured JSON)" and memory node  
  - Edge Cases: API key errors, rate limits, response delays

- **Parse AI Output (Structured JSON)**  
  - Type: LangChain Output Parser Structured JSON  
  - Role: Parses AI response strictly according to schema:
    ```json
    {
      "agendaSummary": "Brief summary of the situation",
      "nextSteps": [
        "Step 1 clearly defined",
        "Step 2 clearly defined",
        "Step 3 clearly defined"
      ],
      "status": "Pending | Scheduled | Missing Info",
      "poId": "same as incoming PO ID",
      "followUpReason": "Short justification why follow-up is required"
    }
    ```
  - Inputs: From AI Model output  
  - Outputs: To "AI Agent: Agent Writer"  
  - Edge Cases: Parsing errors if AI output deviates from schema

- **AI Agent: Agent Writer**  
  - Type: LangChain Agent  
  - Role: Generates final structured agenda and next steps for follow-up  
  - Configuration:  
    - System message instructs AI to respond only in valid JSON  
    - Prompt includes detailed PO data fields  
  - Inputs: From "Parse AI Output" (ai_outputParser) and memory  
  - Outputs: To "Create Calendar Event"  
  - Edge Cases: AI output invalid JSON, missing critical data

---

#### 1.3 Calendar Event Creation & Notifications

**Overview:**  
Creates Google Calendar event with AI-generated agenda, updates Airtable with the event link and status, and sends notifications via Slack and Gmail.

**Nodes Involved:**  
- Create Calendar Event  
- Save Calendar Link to Airtable  
- Send Slack Notification  
- Send Email Confirmation  

**Node Details:**

- **Create Calendar Event**  
  - Type: Google Calendar Node  
  - Role: Creates event with AI-generated agenda as description  
  - Credentials: OAuth2 Google Calendar  
  - Inputs: From "AI Agent: Agent Writer"  
  - Outputs: To "Save Calendar Link to Airtable"  
  - Edge Cases: OAuth token expiry, API rate limits, invalid calendar ID

- **Save Calendar Link to Airtable**  
  - Type: Airtable Node  
  - Role: Updates the PO record with the calendar event link and sets follow-up status to "Pending"  
  - Inputs: From "Create Calendar Event"  
  - Credentials: Airtable Personal Access Token  
  - Configuration: Matches record by internal ID, updates "Follow-up Link" and "Follow-up Status" fields  
  - Outputs: To "Send Slack Notification"  
  - Edge Cases: Airtable API failures, record not found

- **Send Slack Notification**  
  - Type: Slack Node (webhook)  
  - Role: Sends detailed notification to Slack channel including PO details, agenda summary, next steps, and calendar link  
  - Inputs: From "Save Calendar Link to Airtable"  
  - Outputs: To "Send Email Confirmation"  
  - Edge Cases: Slack webhook failures, message formatting issues

- **Send Email Confirmation**  
  - Type: Gmail Node  
  - Role: Sends supplier follow-up notification email with agenda and calendar link  
  - Configuration: Recipient email must be replaced from placeholder (`example@gmail.com`) to actual address  
  - Credentials: Gmail OAuth2  
  - Inputs: From "Send Slack Notification"  
  - Outputs: Loops back to "Process Each PO Individually" to continue next PO processing  
  - Edge Cases: Gmail OAuth token expiry, spam filtering, incorrect recipient

---

#### 1.4 Error Monitoring

**Overview:**  
Captures errors from key nodes and sends Slack alerts for prompt debugging and reliability.

**Nodes Involved:**  
- Error Trigger  
- Slack - Error Alert  

**Node Details:**

- **Error Trigger**  
  - Type: Error Trigger Node  
  - Role: Listens for any errors raised during workflow execution  
  - Inputs: None (system-level)  
  - Outputs: To "Slack - Error Alert"  
  - Edge Cases: May not catch errors outside n8n execution context

- **Slack - Error Alert**  
  - Type: Slack Node (webhook)  
  - Role: Sends formatted Slack alert with error details including node name, error message, and timestamp  
  - Inputs: From "Error Trigger"  
  - Outputs: Ends flow  
  - Credentials: Slack API  
  - Edge Cases: Slack API issues, alert channel misconfiguration

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                                    | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                 |
|------------------------------|-----------------------------------|---------------------------------------------------|----------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------|
| Sticky Note                  | Sticky Note                       | Documentation overview                            | None                             | None                              | ## 🔄 Automated Supplier Follow-up System ... Required fields in Airtable                    |
| Sticky Note1                 | Sticky Note                       | Trigger & Data Fetch overview                      | None                             | None                              | ## ⏰ Trigger & Data Fetch Runs every weekday at 10 AM ...                                  |
| Sticky Note2                 | Sticky Note                       | AI Processing & Validation overview                | None                             | None                              | ## 🤖 AI Processing & Validation Filters POs with valid emails, uses GPT-4 ...              |
| Sticky Note3                 | Sticky Note                       | Calendar & Notifications overview                  | None                             | None                              | ## 📅 Calendar & Notifications Creates Google Calendar events ...                           |
| Sticky Note4                 | Sticky Note                       | Credentials & Security notes                        | None                             | None                              | ## 🔐 Credentials & Security Use OAuth2 for Google Calendar, Slack, Gmail ...               |
| Sticky Note5                 | Sticky Note                       | Error Monitoring overview                           | None                             | None                              | ## 🚨 Error Monitoring Catches errors from key nodes ...                                    |
| Schedule Trigger             | Schedule Trigger                  | Start workflow on schedule                          | None                             | Fetch Overdue POs from Airtable   |                                                                                             |
| Fetch Overdue POs from Airtable | Airtable                         | Fetch open overdue POs needing follow-up           | Schedule Trigger                 | Process Each PO Individually       |                                                                                             |
| Process Each PO Individually | SplitInBatches                   | Process each PO record separately                   | Fetch Overdue POs from Airtable  | Initial Slack Alert, If            |                                                                                             |
| Initial Slack Alert          | Slack                            | Sends preliminary Slack notification                | Process Each PO Individually     | If                               |                                                                                             |
| If                         | If                              | Validate supplier email presence                    | Process Each PO Individually     | AI Model: GPT-4 Screening Engine, Send a message |                                                                                             |
| Send a message              | Slack                            | Alert Slack if supplier email is missing            | If (false branch)                | None                             |                                                                                             |
| AI Model: GPT-4 Screening Engine | LangChain LM Chat OpenAi          | Generate AI response for follow-up agenda           | If (true branch)                 | Parse AI Output, AI Memory         |                                                                                             |
| AI Memory: Candidate Context | LangChain Memory Buffer Window  | Maintain AI session context                          | AI Model                        | AI Agent: Agent Writer             |                                                                                             |
| Parse AI Output (Structured JSON) | LangChain Output Parser Structured | Parse AI JSON output                                 | AI Model                        | AI Agent: Agent Writer             |                                                                                             |
| AI Agent: Agent Writer      | LangChain Agent                 | Generate structured follow-up agenda & next steps  | Parse AI Output, AI Memory       | Create Calendar Event             |                                                                                             |
| Create Calendar Event       | Google Calendar                 | Creates calendar event with AI agenda               | AI Agent: Agent Writer           | Save Calendar Link to Airtable     |                                                                                             |
| Save Calendar Link to Airtable | Airtable                         | Update PO record with calendar link and status      | Create Calendar Event            | Send Slack Notification            |                                                                                             |
| Send Slack Notification     | Slack                            | Notify team with follow-up details                   | Save Calendar Link to Airtable   | Send Email Confirmation            |                                                                                             |
| Send Email Confirmation     | Gmail                            | Send follow-up confirmation email                    | Send Slack Notification          | Process Each PO Individually       | Replace example@gmail.com with actual recipient address                                    |
| Error Trigger              | Error Trigger                   | Capture any errors in workflow                       | None                             | Slack - Error Alert                |                                                                                             |
| Slack - Error Alert        | Slack                            | Send Slack alert when errors detected                | Error Trigger                   | None                             | Sends an error alert message if Topic Classifier, FAQ Generator, or Notion fails.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set cron expression to `0 10 * * 1-5` for 10 AM Monday to Friday  
   - No credentials needed  

2. **Create Airtable Node: Fetch Overdue POs**  
   - Set operation: Search  
   - Configure Airtable credentials with Personal Access Token  
   - Select base and table for Purchase Orders  
   - Use filter formula to find POs where Status = "Open", PO Date older than 30 days, and Follow-up Link is empty or blank  

3. **Create SplitInBatches Node: Process Each PO Individually**  
   - Connect output of Airtable fetch to this node  
   - Default batch size is OK (process one PO at a time)  

4. **Create Slack Node: Initial Slack Alert**  
   - Use Slack webhook or API credentials with channel ID  
   - Message template includes PO ID, Supplier Name, Email, PO Date, and placeholder for calendar link  
   - Connect from "Process Each PO Individually"  

5. **Create If Node: Validate Supplier Email**  
   - Condition: Check if `Supplier Email` field is not empty  
   - Connect from "Process Each PO Individually"  

6. **Create Slack Node: Send a message (Missing Email Alert)**  
   - Send alert to Slack channel indicating missing supplier email  
   - Connect from If node false branch  

7. **Create LangChain LM Chat OpenAI Node: AI Model (GPT-4 Screening Engine)**  
   - Use GPT-4 model variant (e.g., gpt-4o-mini) with temperature 0.7  
   - Connect from If node true branch  
   - Set up OpenAI API credentials  

8. **Create LangChain Memory Buffer Window Node: AI Memory**  
   - Session key: "AI Agenda" (string)  
   - Connect AI Model output to AI Memory (ai_memory channel)  

9. **Create LangChain Output Parser Structured Node**  
   - Define JSON schema for AI response (agendaSummary, nextSteps, status, poId, followUpReason)  
   - Connect AI Model output to parser (ai_languageModel channel)  

10. **Create LangChain Agent Node: AI Agent Writer**  
    - Provide system message to enforce JSON-only response and define rules  
    - Use prompt template with PO details  
    - Connect output parser and AI memory to agent node  

11. **Create Google Calendar Node: Create Calendar Event**  
    - Configure with Google Calendar OAuth2 credentials  
    - Select appropriate calendar  
    - Use AI Agent output to fill event description and details  
    - Connect from AI Agent node  

12. **Create Airtable Node: Save Calendar Link to Airtable**  
    - Update PO record by matching internal ID  
    - Update "Follow-up Link" with calendar event link and set "Follow-up Status" to "Pending"  
    - Use Airtable Personal Access Token credentials  
    - Connect from Google Calendar node  

13. **Create Slack Node: Send Slack Notification**  
    - Notify team channel with PO details, agenda summary, next steps, and calendar link  
    - Connect from Airtable update node  

14. **Create Gmail Node: Send Email Confirmation**  
    - Configure Gmail OAuth2 credentials  
    - Set recipient email (replace placeholder `example@gmail.com` with actual address)  
    - Email content includes PO details, agenda, next steps, and calendar event link  
    - Connect from Slack notification node  
    - Output loops back to "Process Each PO Individually" for next PO  

15. **Create Error Trigger Node**  
    - Add to workflow to catch any runtime errors  

16. **Create Slack Node: Slack - Error Alert**  
    - Connect from Error Trigger  
    - Send error details (node name, message, timestamp) to Slack channel for monitoring  

17. **Add Sticky Note Nodes**  
    - Add descriptive sticky notes to document workflow overview, blocks, credentials, and error monitoring as per content described in the workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow automates supplier follow-ups by integrating Airtable, AI (OpenAI GPT-4), Google Calendar, Slack, and Gmail. Runs weekdays 10 AM on schedule.        | General workflow description                                    |
| Replace placeholder email `example@gmail.com` in Gmail node with actual recipient before deployment to ensure notifications are delivered correctly.            | Gmail Node configuration note                                   |
| OAuth2 authentication is required for Google Calendar, Slack, and Gmail nodes to ensure secure API access. Airtable uses Personal Access Token for API access. | Credentials & Security notes                                    |
| Error Trigger and Slack error alerts help maintain reliability by notifying failures in AI or integration nodes promptly.                                      | Error monitoring and reliability                                |
| Slack notifications provide real-time updates on supplier follow-up triggers and missing data conditions.                                                     | Slack notification usage                                        |

---

Disclaimer: The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.