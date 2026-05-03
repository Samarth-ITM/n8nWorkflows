Outlook Inbox Tamer: GPT-4.1 Powered Categorization, Auto Replies & Team Alerts

https://n8nworkflows.xyz/workflows/outlook-inbox-tamer--gpt-4-1-powered-categorization--auto-replies---team-alerts-9529


# Outlook Inbox Tamer: GPT-4.1 Powered Categorization, Auto Replies & Team Alerts

---
### 1. Workflow Overview

This workflow, titled **"Outlook Inbox Tamer: GPT-4.1 Powered Categorization, Auto Replies & Team Alerts,"** automates the classification and handling of incoming Outlook emails using AI (OpenAI GPT-4.1 models). It targets professionals and teams seeking intelligent inbox management by sorting emails into categories, generating replies, and notifying relevant stakeholders.

The workflow is logically divided into these blocks:

- **1.1 Manual Test Email Sender:** Fetches example emails from Google Sheets and sends them to a test Outlook mailbox for validation.
- **1.2 Outlook Email Trigger & AI Classification:** Listens for new Outlook emails, runs AI classification into four categories, and cleans email content.
- **1.3 Routing by Category:** Routes emails based on AI classification into designated Outlook folders.
- **1.4 Category-Specific Automation:** For each category, automates tailored responses, moves emails, and sends Telegram notifications:
  - High Priority: Drafts email replies and alerts teams via Telegram.
  - Customer Support: Generates auto-replies and notifies support teams.
  - Promotions: Summarizes promotional emails with recommendations.
  - Finance/Billing: Summarizes financial emails and notifies finance teams.
- **1.5 Telegram Notifications:** Sends instant Telegram updates for critical actions across categories.
- **1.6 Documentation and User Notes:** Sticky notes provide workflow overview, instructions, and links to community resources.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Test Email Sender

**Overview:**  
Allows manual testing by sending sample emails fetched from a Google Sheet to a test Outlook mailbox to validate classification logic.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get row(s) in sheet (Google Sheets)  
- Send a message (Gmail)  
- Sticky Note (Instructions)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Purpose: Starts the workflow manually to run test email sending  
  - Inputs: None  
  - Outputs: Triggers next node  
  - Edge Cases: None typical; manual initiation required  

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Config: Reads rows from a specific sheet "Demo_Emails" in a Google Spreadsheet  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Manual Trigger output  
  - Outputs: Email data rows for sending  
  - Edge Cases: Sheet access errors, empty rows, rate limits  

- **Send a message**  
  - Type: Gmail Node  
  - Config: Sends email to a fixed Outlook test email address ("sandy4v@hotmail.com") with subject and body taken from the fetched sheet row  
  - Credentials: Gmail OAuth2  
  - Inputs: Email data from sheet  
  - Outputs: None (end of branch)  
  - Edge Cases: SMTP failures, auth issues, invalid email content  

- **Sticky Note**  
  - Content: Step-by-step instructions for testing the email classifier with Google Sheets sample data and sending to Outlook  
  - Purpose: User guidance  

---

#### 1.2 Outlook Email Trigger & AI Classification

**Overview:**  
Monitors incoming Outlook emails in real-time, passes each email through an AI agent (GPT-4.1-mini) that classifies it into one of four categories with cleaned JSON output.

**Nodes Involved:**  
- Microsoft Outlook Trigger  
- Email_Classifier_Agent (OpenAI GPT)  
- Sticky Note (Step 1 and Step 2 instructions)

**Node Details:**  

- **Microsoft Outlook Trigger**  
  - Type: Microsoft Outlook Trigger (Polling every minute)  
  - Config: Watches inbox for new emails, outputs raw email data  
  - Credentials: Microsoft Outlook OAuth2  
  - Inputs: None (trigger node)  
  - Outputs: Raw email JSON to classifier  
  - Edge Cases: API polling limits, auth expiration, missing emails  

- **Email_Classifier_Agent**  
  - Type: OpenAI Node (Langchain)  
  - Config: Uses GPT-4.1-mini to classify emails into High_Priority, Customer_Support, Promotions, Finance/Billing with detailed JSON output including cleaned body, sender, timestamps, and classification reason  
  - Credentials: OpenAI API Key  
  - Inputs: Email JSON from Outlook Trigger  
  - Outputs: Classified email JSON with category  
  - Expressions: Uses email subject and body content for AI prompt  
  - Edge Cases: API rate limits, malformed email content, AI classification errors, HTML cleanup failures  

- **Sticky Notes**  
  - Step 1: Explains Outlook Trigger role  
  - Step 2: Explains Email classification categories  

---

#### 1.3 Routing by Category

**Overview:**  
Routes classified emails to the appropriate Outlook folder based on their category using a Switch node.

**Nodes Involved:**  
- Routing (Switch node)  
- Move to High Priority  
- Move_To_Customer_Support  
- Move_To_Promotions  
- Move_To_Finance/Billing  
- Sticky Note (Step 3 instructions)

**Node Details:**  

- **Routing**  
  - Type: Switch Node  
  - Config: Routes based on `$json.message.content.classification` matching one of the four categories  
  - Inputs: AI classified email JSON  
  - Outputs: Separate outputs for each category  
  - Edge Cases: Classification value missing or unexpected, falls through without routing  

- **Move to High Priority**  
  - Type: Microsoft Outlook – Move Email  
  - Config: Moves email to "01 High Priority" folder using message ID  
  - Credentials: Outlook OAuth2  
  - Inputs: Routing output for High_Priority  
  - Outputs: Triggers High Priority response generation  
  - Edge Cases: Folder ID changes, message not found, permission errors  

- **Move_To_Customer_Support**  
  - Type: Microsoft Outlook – Move Email  
  - Config: Moves to "02 Customer Support" folder  
  - Credentials: Outlook OAuth2  
  - Edge Cases: Same as above  

- **Move_To_Promotions**  
  - Type: Microsoft Outlook – Move Email  
  - Config: Moves to "03 Promotions" folder  
  - Credentials: Outlook OAuth2  
  - Edge Cases: Same as above  

- **Move_To_Finance/Billing**  
  - Type: Microsoft Outlook – Move Email  
  - Config: Moves to "04 Finance and Billing" folder  
  - Credentials: Outlook OAuth2  
  - Edge Cases: Same as above  

- **Sticky Note**  
  - Explains routing logic and folder destinations  

---

#### 1.4 Category-Specific Automation

**Overview:**  
Handles each email category with tailored AI responses, email drafting/sending, and team notifications via Telegram.

**Nodes Involved:**  

- High Priority Path:  
  - Create_High_Priority_Response (OpenAI)  
  - Create a draft (Outlook)  
  - Send a text message (Telegram)  

- Customer Support Path:  
  - Creating Support Email (OpenAI)  
  - Send Cust Support Response (Outlook)  
  - Notify_Support_Team (Telegram)  

- Promotions Path:  
  - Summary & Rec (OpenAI)  
  - Notify_Support_Team1 (Telegram)  

- Finance/Billing Path:  
  - Summary for Finance Dept (OpenAI)  
  - Send to Finance DL (Outlook)  
  - Notify_Finance_Head (Telegram)  

- Sticky Notes (Step 4 and Step 5 instructions)

**Node Details:**  

- **Create_High_Priority_Response**  
  - Type: OpenAI GPT-4.1-mini  
  - Role: Executive assistant crafting response subject, body, and a text alert message  
  - Inputs: Email from Outlook Trigger  
  - Outputs: JSON with Subject, Message, Text  
  - Edge Cases: AI fails to generate proper response, API limits  

- **Create a draft**  
  - Type: Microsoft Outlook – Create Draft Email  
  - Config: Drafts the AI-generated high priority reply using subject and message content  
  - Inputs: AI response JSON  
  - Outputs: Triggers Telegram text message  
  - Edge Cases: Draft creation failures, permissions  

- **Send a text message (High Priority)**  
  - Type: Telegram  
  - Config: Sends text alert with summary to team chat (must set chat ID)  
  - Inputs: AI text message from draft creation  
  - Edge Cases: Telegram API limits, invalid chat ID  

- **Creating Support Email**  
  - Type: OpenAI GPT-4.1-mini  
  - Role: Generates customer support reply and summary text notifying support team  
  - Inputs: Classified email JSON from AI classifier  
  - Outputs: JSON with reply and notification text  
  - Edge Cases: AI errors, incomplete responses  

- **Send Cust Support Response**  
  - Type: Microsoft Outlook – Send Email  
  - Config: Sends auto-reply to original sender with generated subject and body  
  - Inputs: AI-generated reply JSON  
  - Edge Cases: Sending failures, invalid recipient  

- **Notify_Support_Team**  
  - Type: Telegram  
  - Config: Sends notification text to support team chat  
  - Edge Cases: Same as Telegram above  

- **Summary & Rec (Promotions)**  
  - Type: OpenAI GPT-4.1  
  - Role: Summarizes promotional emails and provides recommendation text  
  - Inputs: Email from Outlook Trigger  
  - Outputs: Text summary for Telegram notification  
  - Edge Cases: Same AI-related issues  

- **Notify_Support_Team1 (Promotions)**  
  - Type: Telegram  
  - Sends promotional summary to team chat  

- **Summary for Finance Dept**  
  - Type: OpenAI GPT-4.1  
  - Role: Summarizes finance/billing emails with concise message for finance team  
  - Inputs: Email from Outlook Trigger and AI classifier  
  - Outputs: JSON with subject, message, notification text  

- **Send to Finance DL**  
  - Type: Microsoft Outlook – Send Email  
  - Sends summarized message to finance distribution list (configured email)  

- **Notify_Finance_Head**  
  - Type: Telegram  
  - Sends finance summary notification with prefix message to finance team chat  

- **Move to High Priority, Move_To_Customer_Support, Move_To_Promotions, Move_To_Finance/Billing** nodes are triggers for these category-specific automations as described above.

- **Sticky Notes**  
  - Step 4: Explains smart response automation per category  
  - Step 5: Explains Telegram notification purpose and chat ID setup  

---

#### 1.5 Telegram Notifications

**Overview:**  
Telegram nodes spread across categories send alerts and summaries to designated Telegram chats to keep teams informed without manual Outlook checks.

**Nodes Involved:**  
- Send a text message (High Priority)  
- Notify_Support_Team (Customer Support)  
- Notify_Support_Team1 (Promotions)  
- Notify_Finance_Head (Finance/Billing)

**Node Details:**  

- All Telegram nodes require a valid Telegram API credential and a chat ID to send messages.  
- Messages are dynamically generated by preceding AI nodes.  
- Edge Cases: Invalid chat ID, API token expiry or revocation, network issues.

---

#### 1.6 Documentation and User Notes

**Overview:**  
Sticky Notes provide stepwise user guidance, workflow overview, and links to creator resources.

**Nodes Involved:**  
- Multiple Sticky Notes positioned around the workflow

**Node Details:**  

- Contain workflow purpose, setup instructions, step explanations, and community links.  
- Important for onboarding and quick understanding.

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                                | Input Node(s)                           | Output Node(s)                         | Sticky Note                                                                                          |
|-------------------------------|---------------------------------|------------------------------------------------|---------------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Initiates manual test email sending             | None                                  | Get row(s) in sheet                   | Run this first. Send emails automatically to test the classifier. Google Sheets data source link provided. |
| Get row(s) in sheet            | Google Sheets                   | Reads test emails from Google Sheets            | When clicking ‘Execute workflow’      | Send a message                       | Run this first.                                                                                     |
| Send a message                 | Gmail                          | Sends test email to Outlook test mailbox         | Get row(s) in sheet                   | None                                | Run this first.                                                                                     |
| Microsoft Outlook Trigger      | Microsoft Outlook Trigger      | Triggers workflow on new incoming Outlook emails | None                                  | Email_Classifier_Agent               | Step 1: Fetch emails from Outlook explained.                                                       |
| Email_Classifier_Agent         | OpenAI (Langchain)             | Classifies emails into categories using GPT-4.1 | Microsoft Outlook Trigger             | Routing                            | Step 2: Classify emails with AI.                                                                    |
| Routing                       | Switch                        | Routes classified emails by category             | Email_Classifier_Agent                | Move to High Priority, Move_To_Customer_Support, Move_To_Promotions, Move_To_Finance/Billing | Step 3: Route emails by category explained.                                                         |
| Move to High Priority          | Microsoft Outlook              | Moves emails to High Priority folder             | Routing                             | Create_High_Priority_Response       | Step 3: Route emails by category explained.                                                         |
| Create_High_Priority_Response  | OpenAI (Langchain)             | Creates draft reply and alert text for High Priority | Move to High Priority               | Create a draft                      | Step 4: Automate smart responses per category.                                                     |
| Create a draft                | Microsoft Outlook              | Creates draft email for High Priority reply      | Create_High_Priority_Response        | Send a text message                 | Step 4: Automate smart responses per category.                                                     |
| Send a text message            | Telegram                       | Sends Telegram alert for High Priority emails    | Create a draft                      | None                                | Step 5: Telegram notifications explained.                                                          |
| Move_To_Customer_Support       | Microsoft Outlook              | Moves emails to Customer Support folder          | Routing                             | Creating Support Email              | Step 3: Route emails by category explained.                                                         |
| Creating Support Email         | OpenAI (Langchain)             | Generates customer support reply and notification | Move_To_Customer_Support             | Send Cust Support Response          | Step 4: Automate smart responses per category.                                                     |
| Send Cust Support Response     | Microsoft Outlook              | Sends auto-reply email to customer                | Creating Support Email               | Notify_Support_Team                 | Step 4: Automate smart responses per category.                                                     |
| Notify_Support_Team            | Telegram                       | Sends Telegram notification to Support Team      | Send Cust Support Response           | None                                | Step 5: Telegram notifications explained.                                                          |
| Move_To_Promotions             | Microsoft Outlook              | Moves emails to Promotions folder                 | Routing                             | Summary & Rec                      | Step 3: Route emails by category explained.                                                         |
| Summary & Rec                 | OpenAI (Langchain)             | Summarizes promotional emails with recommendations | Move_To_Promotions                  | Notify_Support_Team1                | Step 4: Automate smart responses per category.                                                     |
| Notify_Support_Team1           | Telegram                       | Sends Telegram notification for Promotions        | Summary & Rec                      | None                                | Step 5: Telegram notifications explained.                                                          |
| Move_To_Finance/Billing        | Microsoft Outlook              | Moves emails to Finance/Billing folder            | Routing                             | Summary for Finance Dept           | Step 3: Route emails by category explained.                                                         |
| Summary for Finance Dept       | OpenAI (Langchain)             | Summarizes finance/billing emails                  | Move_To_Finance/Billing             | Send to Finance DL                 | Step 4: Automate smart responses per category.                                                     |
| Send to Finance DL             | Microsoft Outlook              | Sends summarized finance email to finance distribution list | Summary for Finance Dept           | Notify_Finance_Head                | Step 4: Automate smart responses per category.                                                     |
| Notify_Finance_Head            | Telegram                       | Sends Telegram notification to finance team       | Send to Finance DL                 | None                                | Step 5: Telegram notifications explained.                                                          |
| Sticky Note                   | Sticky Note                   | User guidance, overview, instructions              | None                                  | None                                | Multiple sticky notes with overview, steps, and links to resources.                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Test Email Sender Block**  
   a. Add **Manual Trigger** node named "When clicking ‘Execute workflow’".  
   b. Add **Google Sheets** node named "Get row(s) in sheet":  
      - Connect to your Google Sheets account with OAuth2 credentials.  
      - Configure to read a specific sheet (e.g., "Demo_Emails") and spreadsheet ID.  
   c. Add **Gmail** node named "Send a message":  
      - Connect Gmail OAuth2 credentials.  
      - Set "Send To" as your Outlook test mailbox email address.  
      - Map Subject and Body from Google Sheets data.  
   d. Connect the chain: Manual Trigger → Google Sheets → Gmail.  

2. **Create Outlook Email Trigger & Classification Block**  
   a. Add **Microsoft Outlook Trigger** node named "Microsoft Outlook Trigger":  
      - Connect Outlook OAuth2 credentials.  
      - Set polling interval (e.g., every minute).  
   b. Add **OpenAI (Langchain)** node named "Email_Classifier_Agent":  
      - Connect OpenAI API credentials.  
      - Use model "gpt-4.1-mini".  
      - Set prompt to classify emails into four categories with cleaned JSON output, including subject, body, sender, timestamps, classification, and reason.  
      - Map email subject and body content from Outlook Trigger.  
   c. Connect Outlook Trigger → Email_Classifier_Agent.  

3. **Create Routing Block**  
   a. Add **Switch** node named "Routing":  
      - Use expression `$json.message.content.classification` to route emails.  
      - Add outputs for "High_Priority", "Customer_Support", "Promotions", and "Finance/Billing".  
   b. Add four **Microsoft Outlook** nodes (Move Email) named:  
      - "Move to High Priority"  
      - "Move_To_Customer_Support"  
      - "Move_To_Promotions"  
      - "Move_To_Finance/Billing"  
      - Configure each to move the message by ID to the corresponding Outlook folder (01 High Priority, 02 Customer Support, 03 Promotions, 04 Finance and Billing).  
      - Connect Switch outputs accordingly.  

4. **Create High Priority Automation**  
   a. Add **OpenAI (Langchain)** node named "Create_High_Priority_Response":  
      - Use GPT-4.1-mini to generate response subject, body, and Telegram text alert.  
      - Input: Email data from "Move to High Priority".  
   b. Add **Microsoft Outlook** node named "Create a draft":  
      - Create draft email using AI-generated subject and message.  
      - Input: From previous OpenAI node.  
   c. Add **Telegram** node named "Send a text message":  
      - Configure with Telegram API credentials and chat ID.  
      - Send AI-generated alert text.  
   d. Connect: Move to High Priority → Create_High_Priority_Response → Create a draft → Send a text message.  

5. **Create Customer Support Automation**  
   a. Add **OpenAI (Langchain)** node named "Creating Support Email":  
      - Use GPT-4.1-mini to compose customer support reply and summary alert.  
      - Input: From "Move_To_Customer_Support".  
   b. Add **Microsoft Outlook** node named "Send Cust Support Response":  
      - Sends reply to original sender with generated subject and body.  
   c. Add **Telegram** node named "Notify_Support_Team":  
      - Sends notification to support team chat.  
   d. Connect: Move_To_Customer_Support → Creating Support Email → Send Cust Support Response → Notify_Support_Team.  

6. **Create Promotions Automation**  
   a. Add **OpenAI (Langchain)** node named "Summary & Rec":  
      - Use GPT-4.1 to summarize promotional emails and generate notification text.  
      - Input: From "Move_To_Promotions".  
   b. Add **Telegram** node named "Notify_Support_Team1":  
      - Sends promotional summary to the team.  
   c. Connect: Move_To_Promotions → Summary & Rec → Notify_Support_Team1.  

7. **Create Finance/Billing Automation**  
   a. Add **OpenAI (Langchain)** node named "Summary for Finance Dept":  
      - Use GPT-4.1 to summarize finance/billing emails with message for finance team.  
      - Input: From "Move_To_Finance/Billing".  
   b. Add **Microsoft Outlook** node named "Send to Finance DL":  
      - Sends summarized email to finance distribution list (set to proper email).  
   c. Add **Telegram** node named "Notify_Finance_Head":  
      - Sends finance summary notification to finance chat.  
   d. Connect: Move_To_Finance/Billing → Summary for Finance Dept → Send to Finance DL → Notify_Finance_Head.  

8. **Add Sticky Notes for User Guidance**  
   - Add sticky notes near relevant nodes with content describing each step and instructions as in the original workflow.  

9. **Set Credentials**  
   - Configure Google Sheets OAuth2 credentials for sheet access.  
   - Configure Gmail OAuth2 credentials for sending test emails.  
   - Configure Microsoft Outlook OAuth2 credentials for inbox monitoring, moving emails, draft creation, and sending emails.  
   - Configure OpenAI API credentials for GPT-4.1 and GPT-4.1-mini models.  
   - Configure Telegram API credentials and set chat IDs for notifications.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow created by **Sandeep Patharkar**                                                                                    | [LinkedIn Profile](https://www.linkedin.com/in/sandeeppatharkar/)                                   |
| Join the Skool community for n8n + AI automation tutorials, live Q&As, and workflow templates                                 | [Skool Community](https://www.skool.com/n8n-ai-automation-champions)                                |
| Workflow uses OpenAI GPT-4.1-mini and GPT-4.1 models for email classification and response generation                         | Ensure OpenAI API key has access to these models                                                    |
| Telegram chat IDs must be replaced with your own team chat IDs for notifications to work correctly                             | Telegram node parameters require valid chatId                                                       |
| Google Sheets sample data available for testing email sending: [Google Sheet Link](https://docs.google.com/spreadsheets/d/1Lz3yPqAfKfCvFjfkWKHj9d_3hyscgF6MdCSQMLAKE58/edit?usp=sharing) | Used for populating test emails                                                                       |
| The workflow uses polling trigger for Outlook; consider webhook or push triggers if supported for more real-time responsiveness | Microsoft Outlook Trigger node configuration note                                                    |

---

**Disclaimer:**  
The provided text is derived exclusively from an n8n automated workflow. It complies with all content policies and contains no illegal, offensive, or protected elements. All data processed are legal and publicly accessible.