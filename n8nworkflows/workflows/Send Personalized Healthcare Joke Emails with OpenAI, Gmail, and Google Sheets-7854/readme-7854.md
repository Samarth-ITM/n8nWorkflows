Send Personalized Healthcare Joke Emails with OpenAI, Gmail, and Google Sheets

https://n8nworkflows.xyz/workflows/send-personalized-healthcare-joke-emails-with-openai--gmail--and-google-sheets-7854


# Send Personalized Healthcare Joke Emails with OpenAI, Gmail, and Google Sheets

### 1. Workflow Overview

This workflow automates sending personalized healthcare-themed joke emails daily to a contact list stored in Google Sheets. It is designed for healthcare professionals or organizations aiming to maintain friendly, automated outreach with a touch of humor. The workflow includes the following logical blocks:

- **1.1 Input Reception & Filtering:** Scheduled trigger initiates the workflow daily, fetching contacts from a Google Sheet and limiting the batch to 10 recipients per run to avoid spamming or hitting API limits.
  
- **1.2 Validation & Sequential Processing:** Each contact is validated for a proper email address, then processed one by one to generate personalized emails.
  
- **1.3 AI-Powered Email Generation:** Uses OpenAI via LangChain to create custom, friendly emails incorporating personalized jokes and recipient names.
  
- **1.4 Email Dispatch & Status Update:** Sends the generated email via Gmail, waits a random 2-5 minutes between sends to mimic human behavior, then updates the Google Sheet with a timestamp to track progress.
  
- **1.5 Rate Limiting and Batch Control:** Integrated through wait nodes and limit nodes to prevent spam flags and control workflow load.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:**  
  The workflow starts with a scheduled trigger at 1 PM daily. It reads the contact list from a configured Google Sheet, then limits the number of contacts to 10 per run to manage load and prevent spam.

- **Nodes Involved:**  
  - Daily Trigger (1 PM)  
  - Healthcare_Contact_List (Google Sheets)  
  - Limit to 10 Contacts

- **Node Details:**  

  - **Daily Trigger (1 PM)**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow daily at 1 PM CST.  
    - Configuration: Set to trigger once daily at hour 13 (1 PM).  
    - Inputs: None  
    - Outputs: Healthcare_Contact_List  
    - Edge cases: Trigger failure due to system downtime or misconfigured timezone.  
    - Version: 1.2

  - **Healthcare_Contact_List**  
    - Type: Google Sheets Read  
    - Role: Fetches contacts from specified Google Sheet and Sheet1 tab.  
    - Configuration: Uses document ID placeholder "YOUR_GOOGLE_SHEET_ID_HERE"; filters contacts where "Emailed" column is empty (i.e., not emailed yet).  
    - Inputs: Trigger output  
    - Outputs: List of contacts with rows containing at least 'First Name' and 'Email' fields.  
    - Edge cases: API quota limits, invalid documentId, empty sheet, or missing required columns.  
    - Version: 4.5

  - **Limit to 10 Contacts**  
    - Type: Limit  
    - Role: Restricts processed contacts to max 10 per workflow run.  
    - Configuration: maxItems set to 10.  
    - Inputs: Contact list from Google Sheets node  
    - Outputs: Maximum 10 contacts forwarded downstream.  
    - Edge cases: If less than 10 contacts remain, processes only available entries.  
    - Version: 1

#### 2.2 Validation & Sequential Processing

- **Overview:**  
  Ensures each contact has a valid, non-empty email address. Processes contacts sequentially to maintain order and manage rate limits.

- **Nodes Involved:**  
  - Validate Email Address (IF)  
  - Skip Invalid Email (NoOp)  
  - Process One by One (SplitInBatches)

- **Node Details:**  

  - **Validate Email Address**  
    - Type: IF Node  
    - Role: Checks if the 'Email' field is non-empty and valid for each contact.  
    - Configuration: Conditions check that 'Email' is not empty (string nonEmpty).  
    - Inputs: Limited contacts batch  
    - Outputs: Valid emails proceed to processing; invalid emails sent to Skip Invalid Email node.  
    - Edge cases: Malformed email addresses are not explicitly validated beyond non-empty; may need regex enhancement for stricter validation.  
    - Version: 2.2

  - **Skip Invalid Email**  
    - Type: No Operation  
    - Role: Placeholder node for invalid emails; effectively ignores them.  
    - Inputs: Contacts failing validation  
    - Outputs: None, stops invalid emails from further processing.  
    - Edge cases: None  
    - Version: 1

  - **Process One by One**  
    - Type: SplitInBatches  
    - Role: Splits valid contacts to process sequentially (batch size of 1 implied by default).  
    - Configuration: Default batch size (1) to process contacts individually.  
    - Inputs: Validated contacts  
    - Outputs: Single contact per iteration to AI Email Generator.  
    - Edge cases: Large contact lists processed in multiple workflow executions; possible slowdown.  
    - Version: 3

#### 2.3 AI-Powered Email Generation

- **Overview:**  
  For each contact, an AI agent generates a personalized, friendly healthcare outreach email with a humorous medical joke, using OpenAI's GPT-4o-mini model via LangChain integration.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - AI Email Generator  
  - Prepare Email Data

- **Node Details:**  

  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides the language model (GPT-4o-mini) for AI-driven text generation.  
    - Configuration: Model set to "gpt-4o-mini" for optimized balance of performance and cost.  
    - Inputs: Prompt from AI Email Generator node.  
    - Outputs: AI-generated text continuation.  
    - Edge cases: API key missing or invalid, rate limits, model unavailability, response timeouts.  
    - Version: 1.2

  - **AI Email Generator**  
    - Type: LangChain Agent Node  
    - Role: Defines the prompt and system message for personalized email generation.  
    - Configuration:  
      - System Message: Professional healthcare rep writing friendly outreach emails including recipient's first name and a fixed medical joke.  
      - Prompt: "Write a friendly email to {{ $json['First Name'] }}..." instructing to return only the plain text email body.  
    - Inputs: One contact item at a time from Process One by One node.  
    - Outputs: AI-generated email body text.  
    - Edge cases: Expression failures if 'First Name' is missing; incomplete or malformed AI response.  
    - Version: 1.9

  - **Prepare Email Data**  
    - Type: Set Node  
    - Role: Prepares and organizes the email data for sending.  
    - Configuration: Assigns 'Email', 'First Name', and 'EmailContent' (AI output) fields to the outgoing data.  
    - Inputs: AI Email Generator output  
    - Outputs: Structured data for sending node.  
    - Edge cases: Missing AI output or email fields would cause send failures.  
    - Version: 3.4

#### 2.4 Email Dispatch & Status Update

- **Overview:**  
  Sends the personalized email via Gmail, then waits a random 2 to 5 minutes before processing the next contact. After the wait, updates the Google Sheet with a timestamp marking that the email was sent.

- **Nodes Involved:**  
  - Send Email (Gmail)  
  - Random Wait (2-5min)  
  - Update Email Status (Google Sheets)

- **Node Details:**  

  - **Send Email**  
    - Type: Gmail  
    - Role: Sends the generated email text to the recipient's email address.  
    - Configuration:  
      - Recipient email dynamically set from contact list.  
      - Subject fixed as "Medical joke of the day".  
      - Email type: plain text.  
      - Disable automatic attribution footer.  
    - Inputs: Prepared email data.  
    - Outputs: Proceed to wait and update status.  
    - Edge cases: Authentication errors, quota exceeded, invalid emails not caught earlier, send failures.  
    - Version: 2.1  
    - Credential: Gmail OAuth2 required.

  - **Random Wait (2-5min)**  
    - Type: Wait  
    - Role: Introduces random delay between 2 and 5 minutes after each email sent.  
    - Configuration: Amount is randomized via expression `Math.floor(Math.random() * 4) + 2`.  
    - Inputs: After Send Email node.  
    - Outputs: Triggers next iteration and status update.  
    - Edge cases: Workflow timeout if wait is too long; possible scheduler conflicts.  
    - Version: 1.1

  - **Update Email Status**  
    - Type: Google Sheets Update  
    - Role: Updates the "Emailed" column with the current timestamp in the contact list sheet to mark progress.  
    - Configuration:  
      - Uses row_number from earlier Google Sheets node for accurate row update.  
      - Timestamp generated by `$now`.  
    - Inputs: After Random Wait node (parallel to Process One by One to next batch).  
    - Outputs: None (end of flow for that contact).  
    - Edge cases: Conflicts if multiple instances update simultaneously; API quota limits.  
    - Version: 4.5

#### 2.5 Rate Limiting and Batch Control

- **Overview:**  
  This is a conceptual block managed by the Limit node and Random Wait node, integrated with the sequential processing approach to avoid spam detection and service throttling.

- **Nodes Involved:**  
  - Limit to 10 Contacts  
  - Random Wait (2-5min)  
  - Process One by One

- **Node Details:**  
  - These nodes collectively ensure that no more than 10 emails are sent per execution, each spaced out by randomized delays, and processed one at a time to simulate natural sending behavior.  
  - Edge cases include delays causing workflow overlap or exceeding execution time limits.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                      | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                          |
|-------------------------|----------------------------------|------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Daily Trigger (1 PM)     | Schedule Trigger                 | Workflow daily start trigger       | None                         | Healthcare_Contact_List        | ## Healthcare Email Autoresponder 📧<br>- Sends personalized healthcare jokes daily<br>- Limits to 10 emails per run<br>- Random delays between emails |
| Healthcare_Contact_List  | Google Sheets                   | Fetch contacts not yet emailed     | Daily Trigger (1 PM)          | Limit to 10 Contacts           | See above                                                                                          |
| Limit to 10 Contacts     | Limit                          | Batch size control to max 10       | Healthcare_Contact_List       | Validate Email Address         | ## 🎯 Batch Processing<br>- Limits to 10 emails per run to prevent overwhelming email service      |
| Validate Email Address   | IF                             | Validates presence of email field  | Limit to 10 Contacts          | Process One by One (valid), Skip Invalid Email (invalid) | ## 🔄 Processing Flow<br>- Validates email exists<br>- Processes contacts sequentially            |
| Skip Invalid Email       | No Operation                   | Discards invalid emails            | Validate Email Address (invalid) | None                         | See above                                                                                          |
| Process One by One       | SplitInBatches                 | Processes contacts individually    | Validate Email Address (valid) | AI Email Generator           | See above                                                                                          |
| OpenAI Chat Model        | LangChain LM Chat OpenAI       | Provides GPT-4o-mini model         | AI Email Generator            | AI Email Generator             | ## ✨ AI Email Generation<br>- Customizes emails with personalized jokes and names                  |
| AI Email Generator       | LangChain Agent                | Generates personalized email text | Process One by One            | Prepare Email Data             | See above                                                                                          |
| Prepare Email Data       | Set                           | Structures data for sending        | AI Email Generator            | Send Email                    | See above                                                                                          |
| Send Email               | Gmail                         | Sends the email                    | Prepare Email Data            | Random Wait (2-5min)           | ## ⏰ Smart Timing<br>- Random 2-5 min delays to avoid spam flags and appear natural               |
| Random Wait (2-5min)     | Wait                          | Introduces delay between emails    | Send Email                   | Process One by One, Update Email Status | See above                                                                                          |
| Update Email Status      | Google Sheets Update           | Marks email as sent in sheet       | Random Wait (2-5min)          | None                         | ## 📊 Email Tracking<br>- Updates Google Sheet with timestamp to prevent duplicate sends           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Scheduled Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 1 PM (hour = 13) in your timezone (America/Chicago).  

2. **Add Google Sheets Node to Read Contact List:**  
   - Type: Google Sheets  
   - Operation: Read  
   - Document ID: Your Google Sheet ID containing contacts  
   - Sheet Name: "Sheet1"  
   - Apply filter: Select rows where "Emailed" column is empty (to avoid duplicates)  
   - Connect output from Schedule Trigger to this node.

3. **Add Limit Node:**  
   - Type: Limit  
   - Set maxItems to 10 to restrict batch size.  
   - Connect Google Sheets node output to Limit node.

4. **Add IF Node to Validate Email Address:**  
   - Type: IF  
   - Condition: Check 'Email' field is not empty (string nonEmpty).  
   - Connect Limit node to IF node.

5. **Add No Operation Node for Skipping Invalid Emails:**  
   - Type: NoOp  
   - Connect IF node’s false output (invalid email) to this node.

6. **Add SplitInBatches Node for Sequential Processing:**  
   - Type: SplitInBatches  
   - Default batch size (1) to process one contact at a time.  
   - Connect IF node’s true output (valid emails) to this node.

7. **Add LangChain OpenAI Chat Model Node:**  
   - Type: LangChain LM Chat OpenAI  
   - Model: Set to "gpt-4o-mini"  
   - Connect this node as the AI language model for the next agent node.

8. **Add LangChain Agent Node for AI Email Generation:**  
   - Type: LangChain Agent  
   - Configure prompt with system message:  
     - You are a professional healthcare representative writing personalized outreach emails.  
     - Include recipient's first name and a fixed medical joke.  
     - Request only the full email body text to be returned.  
   - Prompt: "Write a friendly email to {{ $json['First Name'] }}" (use expression).  
   - Connect SplitInBatches node output to this node.  
   - Connect OpenAI Chat Model node as its language model.

9. **Add Set Node to Prepare Email Data:**  
   - Type: Set  
   - Assign fields:  
     - Email: `{{$json.Email}}`  
     - First Name: `{{$json['First Name']}}`  
     - EmailContent: `{{$node["AI Email Generator"].json["output"]}}`  
   - Connect AI Email Generator node output to this node.

10. **Add Gmail Node to Send Email:**  
    - Type: Gmail  
    - Credentials: Setup Gmail OAuth2 with appropriate permissions.  
    - Send To: `={{ $json.Email }}`  
    - Subject: "Medical joke of the day"  
    - Message: `={{ $json.EmailContent }}`  
    - Email type: Text  
    - Turn off attribution footer  
    - Connect Prepare Email Data node to this node.

11. **Add Wait Node to Introduce Random Delay:**  
    - Type: Wait  
    - Unit: minutes  
    - Amount: Expression `={{ Math.floor(Math.random() * 4) + 2 }}` (random 2-5 minutes)  
    - Connect Send Email node to this node.

12. **Add Google Sheets Update Node to Mark Email Sent:**  
    - Type: Google Sheets  
    - Operation: Update  
    - Document ID and Sheet Name same as contact list  
    - Mapping:  
      - Column "Emailed": `={{ $now }}` (current timestamp)  
      - Matching by "row_number" field from original contact list item to update correct row  
    - Connect Wait node output here.

13. **Connect Wait Node output back to SplitInBatches node to continue processing next contact.**

14. **Verify all nodes connected properly:**  
    - Daily Trigger → Google Sheets (Read) → Limit → Validate Email Address →  
      - True → Process One by One → AI Email Generator chain → Prepare Email Data → Send Email → Random Wait → Update Email Status  
      - False → Skip Invalid Email

15. **Set credentials:**  
    - Google Sheets with proper API access to the spreadsheet  
    - Gmail OAuth2 with send email permissions  
    - OpenAI API key configured in LangChain nodes

16. **Test the workflow with a small contact list before activating scheduling.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                         | Context or Link                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Workflow sends personalized healthcare outreach emails with humor, leveraging AI for customization and Google Sheets for contact management.          | Workflow purpose summary                                         |
| Avoid spamming by limiting batch size to 10 and adding random delays between emails (2-5 minutes) to mimic natural sending patterns.                 | Rate limiting and batch control                                  |
| Customize the AI email generator system message to modify the tone, joke, or signature to suit your organization's voice.                            | AI Email Generator customization                                 |
| Requires setup of Gmail OAuth2 credentials, Google Sheets API access, and OpenAI API key for LangChain nodes.                                         | Credential configuration                                         |
| Sticky Notes included in the workflow provide inline documentation and guides; double-click them in n8n to edit and see more info.                    | Inline workflow documentation                                   |
| For detailed usage of sticky notes in n8n workflows, see: https://docs.n8n.io/workflows/sticky-notes/                                                | Sticky notes guide                                               |
| Workflow timezone set to America/Chicago; adjust schedule trigger timezone to your local preference.                                                  | Timezone settings                                                |

---

**Disclaimer:** The provided description and nodes come exclusively from an automated n8n workflow. All data handled is legal and public, and this workflow complies with content policies.