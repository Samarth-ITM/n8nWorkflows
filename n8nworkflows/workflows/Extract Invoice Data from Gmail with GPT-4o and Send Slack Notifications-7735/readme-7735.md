Extract Invoice Data from Gmail with GPT-4o and Send Slack Notifications

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-gmail-with-gpt-4o-and-send-slack-notifications-7735


# Extract Invoice Data from Gmail with GPT-4o and Send Slack Notifications

---

### 1. Workflow Overview

This workflow automates the detection and extraction of invoice-related data from unread Gmail emails using GPT-4o via an AI Agent node, and sends notifications to Slack users when invoices are detected. It targets scenarios where businesses need to streamline invoice processing and timely notifications without manual email scanning.

The workflow is structured into four main logical blocks:

- **1.1 Scheduled Email Retrieval:** Periodically triggers the workflow and fetches unread emails from Gmail.
- **1.2 AI Invoice Detection & Data Extraction:** Processes each email through an AI Agent powered by GPT-4o to determine if it is invoice-related and extracts key details (due date, amount).
- **1.3 Invoice Validation:** Uses conditional logic to route only invoice-positive emails for notification.
- **1.4 Slack Notification:** Sends a formatted Slack message to a specified user summarizing the invoice details.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Email Retrieval

**Overview:**  
This block triggers the workflow on a recurring schedule and retrieves all unread emails from the connected Gmail account.

**Nodes Involved:**  
- Schedule Trigger  
- Get Unread Emails

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow every hour to check for new emails automatically.  
  - *Configuration:* Interval set to 1 hour.  
  - *Input:* None (time-based trigger).  
  - *Output:* Triggers the next node once per interval.  
  - *Edge Cases:* If the node misfires or the schedule is modified incorrectly, emails could be missed or checked too frequently.  
  - *Sticky Note:* Explains scheduling and interval adjustment.  

- **Get Unread Emails**  
  - *Type:* Gmail Trigger (Get All)  
  - *Role:* Fetches all unread emails from the connected Gmail account.  
  - *Configuration:* Operation "getAll" with filter for "unread" emails only. Simple mode disabled to retrieve full email data.  
  - *Input:* Trigger from Schedule Trigger.  
  - *Output:* Sends array of email objects to AI Agent node.  
  - *Edge Cases:* Authentication errors if Gmail OAuth2 token expires; rate limits on Gmail API; large volume of emails may cause timeouts or slow execution.  
  - *Credential:* Requires Gmail OAuth2 connection.  
  - *Sticky Note:* Setup reminder to connect Gmail account.

---

#### 2.2 AI Invoice Detection & Data Extraction

**Overview:**  
Processes each unread email to determine if it is related to an invoice/payment notification. If yes, extracts structured data such as due date and amount due using GPT-4o.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain AI Agent (GPT-4o)  
  - *Role:* Acts as an intelligent assistant analyzing email content to classify invoice relevance and extract data.  
  - *Configuration:*  
    - Custom prompt instructing the model to: identify invoice-related emails, extract due date (YYYY-MM-DD or null), amount due (number or null), and metadata fields (email ID, thread ID, sender, subject).  
    - Output parser enabled to enforce JSON output format.  
    - On error: continue with regular output to avoid workflow failure on AI errors.  
  - *Input:* Email JSON from "Get Unread Emails" node.  
  - *Output:* JSON object indicating invoice status and extracted fields.  
  - *Key Expressions:* Uses JSON placeholders like `{{ $json.id }}`, `{{ $json.subject }}`, etc. for prompt interpolation.  
  - *Edge Cases:* AI misclassification; malformed JSON output; rate limits or API errors from OpenAI; prompt tuning required for accuracy improvements.  
  - *Credential:* Requires OpenAI API key configured under API credentials.  
  - *Sticky Note:* Describes AI Agent customization potential.

- **OpenAI Chat Model**  
  - *Type:* LangChain LM Chat OpenAI  
  - *Role:* Provides the GPT-4o language model for the AI Agent node.  
  - *Configuration:* Model set to "gpt-4o", cached result enabled for efficiency.  
  - *Input:* Connected internally as AI Agent’s language model.  
  - *Output:* Intermediate model output for parsing.  
  - *Edge Cases:* API key invalid or exhausted; model unavailability.  

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser  
  - *Role:* Validates and parses the AI Agent’s output to a strict JSON schema ensuring all expected fields are present and correctly typed.  
  - *Configuration:* Manual JSON schema defined with required fields: is_invoice (boolean), due_date (string|null), amount_due (number|null), email_id, thread_id, sender, subject.  
  - *Input:* AI Agent output.  
  - *Output:* Parsed and validated structured JSON for downstream processing.  
  - *Edge Cases:* Parsing errors if AI output is malformed or does not match schema; workflow may fail or need error handling.  

---

#### 2.3 Invoice Validation

**Overview:**  
Filters processed emails to route only those identified as invoices for notification, while ignoring others.

**Nodes Involved:**  
- Check If Email is Invoice  
- No Operation, do nothing

**Node Details:**

- **Check If Email is Invoice**  
  - *Type:* If Node  
  - *Role:* Conditional router that checks `is_invoice` boolean in AI Agent output.  
  - *Configuration:* Condition tests if `{{ $json.output.is_invoice }}` is true.  
  - *Input:* Parsed AI Agent output JSON.  
  - *Output:*  
    - True branch: forwards invoice emails to Slack notification node.  
    - False branch: sends non-invoice emails to a no-op node to effectively ignore them.  
  - *Edge Cases:* Missing or malformed `is_invoice` field; logic failure causing false negatives/positives.

- **No Operation, do nothing**  
  - *Type:* NoOp Node  
  - *Role:* Terminates workflow path for emails not related to invoices without further action.  
  - *Input:* False branch from "Check If Email is Invoice".  
  - *Output:* None.  
  - *Edge Cases:* None (safe termination).

---

#### 2.4 Slack Notification

**Overview:**  
Sends a notification message to a configured Slack user with extracted invoice details for follow-up.

**Nodes Involved:**  
- Notify Slack User

**Node Details:**

- **Notify Slack User**  
  - *Type:* Slack Node (Message)  
  - *Role:* Sends a direct message notification to a specified Slack user about the detected invoice.  
  - *Configuration:*  
    - Message text uses template expressions to include sender, amount due, and due date, e.g., `"Invoice from {{ $json.output.sender }} – ${{ $json.output.amount_due }} due {{ $json.output.due_date }}"`.  
    - User specified by username (empty by default, must be set).  
    - Authentication via OAuth2 Slack credentials.  
  - *Input:* True branch from "Check If Email is Invoice".  
  - *Output:* Slack API response.  
  - *Edge Cases:* Slack API errors (rate limits, invalid user), missing user configuration, message formatting issues.  
  - *Credential:* Requires Slack OAuth2 connection.  
  - *Sticky Note:* Guidance on customizing notification format and message content.

---

### 3. Summary Table

| Node Name               | Node Type                               | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                               |
|-------------------------|---------------------------------------|---------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                      | Initiates workflow hourly              | None                   | Get Unread Emails       | Explains scheduling interval and adjustment.                                                            |
| Get Unread Emails       | Gmail                                | Retrieves unread emails from Gmail    | Schedule Trigger       | AI Agent                | Setup reminder to connect Gmail account with OAuth2.                                                    |
| AI Agent               | LangChain AI Agent (GPT-4o)          | Detects invoice emails and extracts data | Get Unread Emails      | Check If Email is Invoice | Notes on AI customization and prompt usage for invoice detection.                                       |
| OpenAI Chat Model       | LangChain LM Chat OpenAI              | Provides GPT-4o language model         | Internal to AI Agent   | AI Agent                |                                                                                                          |
| Structured Output Parser| LangChain Output Parser               | Validates AI output against schema    | OpenAI Chat Model      | AI Agent                |                                                                                                          |
| Check If Email is Invoice| If                                  | Routes invoices for notification       | AI Agent               | Notify Slack User, No Operation |                                                                                                          |
| Notify Slack User       | Slack                                | Sends invoice notification to Slack user | Check If Email is Invoice | None                    | Guidance on customizing Slack notification message format.                                              |
| No Operation, do nothing| NoOp                                 | Terminates non-invoice email paths    | Check If Email is Invoice | None                    |                                                                                                          |
| Sticky Note2            | Sticky Note                          | Setup instructions                     | None                   | None                    | Setup Gmail, Slack OAuth2 connections, and OpenAI API key.                                              |
| Sticky Note1            | Sticky Note                          | Scheduling explanation                  | None                   | None                    | Explains schedule trigger hourly run.                                                                   |
| Sticky Note             | Sticky Note                          | Notification format customization      | None                   | None                    | Advice on customizing Slack notification message title and notes.                                       |
| Sticky Note3            | Sticky Note                          | AI Agent explanation                    | None                   | None                    | Information on AI prompt and customization potential.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to run every 1 hour (field: hours, value: 1).  
   - This will serve as the workflow’s entry point.

2. **Add a Gmail node ("Get Unread Emails")**  
   - Operation: `getAll`  
   - Filters: `readStatus` set to `unread`  
   - Set "Simple" mode to false to retrieve full email data.  
   - Connect output of Schedule Trigger to input of this Gmail node.  
   - Configure Gmail OAuth2 credentials in n8n credentials manager.

3. **Add an AI Agent node ("AI Agent")**  
   - Select LangChain AI Agent node type.  
   - Set prompt type to "Define".  
   - Enter the custom prompt instructing the AI to detect invoice-related emails and extract due date, amount due, and metadata. Use placeholders to insert email fields (`{{ $json.id }}`, `{{ $json.threadId }}`, etc.).  
   - Enable Output Parser.  
   - Set "On Error" to continue with regular output to avoid workflow failure.  
   - Connect output of "Get Unread Emails" to input of "AI Agent" node.  

4. **Create an OpenAI Chat Model node ("OpenAI Chat Model")**  
   - Set model to `gpt-4o`.  
   - Enable cached result for efficiency.  
   - Connect this node to the AI Agent’s language model input port.  
   - Configure OpenAI API key credentials in n8n.  

5. **Add a Structured Output Parser node ("Structured Output Parser")**  
   - Set parser type to "manual".  
   - Define the JSON schema requiring:  
     - `is_invoice` (boolean)  
     - `due_date` (string or null, format date)  
     - `amount_due` (number or null)  
     - `email_id` (string)  
     - `thread_id` (string)  
     - `sender` (string)  
     - `subject` (string)  
   - Connect this node to the AI Agent’s output parser input port.

6. **Add an If node ("Check If Email is Invoice")**  
   - Condition: Check if `{{ $json.output.is_invoice }}` is true (boolean equals true).  
   - Connect output of AI Agent node to input of this If node.

7. **Add a Slack node ("Notify Slack User")**  
   - Operation: Send message to user.  
   - Message text: `Invoice from {{ $json.output.sender }} – ${{ $json.output.amount_due }} due {{ $json.output.due_date }}`  
   - Set user by username or Slack user ID (must be configured).  
   - Connect the true output branch of the If node to this Slack node.  
   - Configure Slack OAuth2 credentials.

8. **Add a No Operation node ("No Operation, do nothing")**  
   - Connect the false output branch of the If node to this node to discard non-invoice emails.

9. **Arrange Sticky Note nodes (optional)**  
   - Add reminders for setup steps: Gmail and Slack OAuth2 connections, OpenAI API key configuration, scheduling interval, and notification customization.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Setup requires connecting Gmail and Slack accounts via OAuth2 and adding OpenAI API key in n8n. | Sticky Note2: Setup instructions.                                                             |
| The Schedule Trigger runs every hour by default; adjust interval as needed for email checking.  | Sticky Note1: Scheduling explanation.                                                         |
| AI prompt can be customized to detect other email types (e.g., receipts, contracts).            | Sticky Note3: AI Agent customization guidance.                                                |
| Slack notification message format can be customized to include additional invoice details.     | Sticky Note: Notification format customization tips.                                          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---