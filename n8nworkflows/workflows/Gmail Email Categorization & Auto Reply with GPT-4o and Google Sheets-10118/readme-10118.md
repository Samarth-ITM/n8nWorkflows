Gmail Email Categorization & Auto Reply with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/gmail-email-categorization---auto-reply-with-gpt-4o-and-google-sheets-10118


# Gmail Email Categorization & Auto Reply with GPT-4o and Google Sheets

### 1. Workflow Overview

This workflow automates the processing, categorization, and response to incoming Gmail messages using AI-powered classification and generation. It targets users who receive diverse types of emails and want to streamline their inbox management with minimal manual intervention. The system classifies emails into four categories—High Priority, Advertisement, Inquiry, and Finance/Billing—and performs tailored actions for each category, including labeling, drafting replies, summarization, logging to Google Sheets, and sending automated replies.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receiving new Gmail emails in real time.
- **1.2 AI Classification:** Categorizing emails into four predefined classes using LangChain’s Text Classifier.
- **1.3 Email Labeling:** Adding Gmail labels according to the classification result.
- **1.4 AI Processing & Response Generation:** Using OpenAI GPT-4o models to generate replies, summaries, or recommendations depending on the category.
- **1.5 Output Actions:** Creating Gmail drafts, sending replies, and appending rows to Google Sheets for record keeping.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block continuously monitors the Gmail inbox and triggers the workflow for every new incoming email.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**

  - **Gmail Trigger**  
    - Type: `n8n-nodes-base.gmailTrigger`  
    - Role: Monitors Gmail account for new incoming emails.  
    - Configuration: Polls every minute for new messages, uses OAuth2 credentials for Gmail account.  
    - Key expressions: None; outputs full email JSON including `text` and `id`.  
    - Input: None (trigger node).  
    - Output: Passes email JSON to Text Classifier.  
    - Potential failures: OAuth token expiration, Gmail API rate limits, connectivity issues.

---

#### 1.2 AI Classification

- **Overview:**  
  This block analyzes the email text and categorizes the email into one of four categories using a LangChain Text Classifier node.

- **Nodes Involved:**  
  - Text Classifier

- **Node Details:**

  - **Text Classifier**  
    - Type: `@n8n/n8n-nodes-langchain.textClassifier`  
    - Role: Categorizes email content into High Priority, Advertisement, Inquiry, or Finance/Billing.  
    - Configuration:  
      - Input text extracted from `{{$json.text}}` (email body).  
      - Categories defined with detailed descriptions and examples to guide classification.  
    - Input: Email JSON from Gmail Trigger.  
    - Output: Routes classified email to one of four Gmail label nodes based on category.  
    - Potential failures: Text extraction errors if email content is missing or malformed; classification miscategorization; API limits or errors from LangChain/OpenAI.

---

#### 1.3 Email Labeling

- **Overview:**  
  Depending on the classification, the workflow applies the appropriate Gmail label to the email to organize the inbox.

- **Nodes Involved:**  
  - Save Priority Mail  
  - Save Advertisement Mail  
  - Save Inquiry Mail  
  - Save Finance Mail

- **Node Details:**

  - **Save Priority Mail**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Adds “High Priority” Gmail label to the email.  
    - Configuration: Uses label ID `Label_6395372519299544115` and email `id` from incoming JSON.  
    - Input: From Text Classifier output for High Priority emails.  
    - Output: Triggers AI draft generation for High Priority.  
    - Failures: Label ID mismatch, Gmail API failures.

  - **Save Advertisement Mail**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Adds “Advertisement” Gmail label.  
    - Configuration: Label ID `Label_2877175876700118240`.  
    - Input: From Text Classifier Advertisement branch.  
    - Output: Triggers summary generation for Advertisement emails.  
    - Failures: Same as above.

  - **Save Inquiry Mail**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Adds “Inquiry” Gmail label.  
    - Configuration: Label ID `Label_8971387974199578288`.  
    - Input: From Text Classifier Inquiry branch.  
    - Output: Triggers reply generation for Inquiry emails.  
    - Failures: Same as above.

  - **Save Finance Mail**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Adds “Finance/Billing” Gmail label.  
    - Configuration: Label ID `Label_7818902445747119871`.  
    - Input: From Text Classifier Finance/Billing branch.  
    - Output: Triggers finance summary generation.  
    - Failures: Same as above.

---

#### 1.4 AI Processing & Response Generation

- **Overview:**  
  This block uses OpenAI GPT-4o models to generate category-specific content: draft replies, summaries, or recommendations.

- **Nodes Involved:**  
  - Generate Priority Draft  
  - Generate Ad Summary  
  - Generate Inquiry Reply  
  - Generate Finance Summary

- **Node Details:**

  - **Generate Priority Draft**  
    - Type: `@n8n/n8n-nodes-langchain.openAi`  
    - Role: Creates an executive assistant style reply draft for High Priority emails.  
    - Configuration: GPT-4o model; prompt asks to output subject and message based on email text.  
    - Input: Email text from Gmail Trigger.  
    - Output: JSON with subject and message for draft creation.  
    - Failures: Model API errors, prompt interpretation issues.

  - **Generate Ad Summary**  
    - Type: `@n8n/n8n-nodes-langchain.openAi`  
    - Role: Summarizes Advertisement emails concisely for spreadsheet logging.  
    - Configuration: GPT-4o model; prompt requests sender, subject, summary, and recommendation fields.  
    - Input: Email text from Gmail Trigger.  
    - Output: JSON with summary fields for Google Sheets.  
    - Failures: Same as above.

  - **Generate Inquiry Reply**  
    - Type: `@n8n/n8n-nodes-langchain.openAi`  
    - Role: Drafts reply for customer inquiries; fallback includes contact number if unknown.  
    - Configuration: GPT-4o model; prompt requests subject and message for reply.  
    - Input: Email text from Gmail Trigger.  
    - Output: JSON with subject and message for sending reply.  
    - Failures: Same as above.

  - **Generate Finance Summary**  
    - Type: `@n8n/n8n-nodes-langchain.openAi`  
    - Role: Summarizes financial/billing emails briefly.  
    - Configuration: GPT-4o model; prompt asks for subject and message summary.  
    - Input: Email text from Gmail Trigger.  
    - Output: JSON for sending reply mail.  
    - Failures: Same as above.

---

#### 1.5 Output Actions

- **Overview:**  
  This block executes final actions such as creating Gmail drafts, sending replies, and appending summarized data to Google Sheets.

- **Nodes Involved:**  
  - Create Draft  
  - Append Row to Sheet  
  - Send Finance Mail  
  - Reply to Inquiry

- **Node Details:**

  - **Create Draft**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Creates a Gmail draft reply for High Priority emails.  
    - Configuration: Uses generated subject and message from AI output; uses Gmail OAuth2 credentials.  
    - Input: Output JSON from Generate Priority Draft.  
    - Output: None (final action).  
    - Failures: Gmail API quota, draft creation errors.

  - **Append Row to Sheet**  
    - Type: `n8n-nodes-base.googleSheets`  
    - Role: Appends summarized Advertisement email data to Google Sheets for record keeping.  
    - Configuration: Maps AI-generated fields (Sender, Subject, Summary, Recommendation) to sheet columns; uses Google Sheets OAuth2 credentials; targets specific spreadsheet and sheet (gid=0).  
    - Input: Output JSON from Generate Ad Summary.  
    - Output: None (final action).  
    - Failures: Google Sheets API limits, invalid spreadsheet ID or permissions.

  - **Send Finance Mail**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Sends a reply email within the Gmail thread for Finance/Billing emails.  
    - Configuration: Uses threadId and messageId from labeled Finance email node; message content from AI output.  
    - Input: Output JSON from Generate Finance Summary.  
    - Output: None (final action).  
    - Failures: Gmail API errors, thread/message ID mismatch.

  - **Reply to Inquiry**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Sends an auto-reply email for Inquiry emails to a configured recipient.  
    - Configuration: Uses AI-generated subject and message; sendTo is a placeholder `{{YOUR_EMAIL_ADDRESS}}` requiring replacement; Gmail OAuth2 credentials.  
    - Input: Output JSON from Generate Inquiry Reply.  
    - Output: None (final action).  
    - Failures: Placeholder not replaced, Gmail API errors.

---

### 3. Summary Table

| Node Name            | Node Type                                | Functional Role                             | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                          |
|----------------------|----------------------------------------|---------------------------------------------|-------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger        | n8n-nodes-base.gmailTrigger             | Triggers workflow on new Gmail emails       | None                    | Text Classifier             | ## ①Receive Gmail → AI understands the email content Incoming emails are analyzed by AI and categorized automatically. |
| Text Classifier      | @n8n/n8n-nodes-langchain.textClassifier | Categorizes email into 4 categories          | Gmail Trigger           | Save Priority Mail, Save Advertisement Mail, Save Inquiry Mail, Save Finance Mail | Same as above                                                                                                        |
| Save Priority Mail   | n8n-nodes-base.gmail                    | Adds High Priority label to email            | Text Classifier         | Generate Priority Draft      | ## ②Email Processing Section Describes AI classification and tailored actions for each category.                     |
| Save Advertisement Mail | n8n-nodes-base.gmail                  | Adds Advertisement label                     | Text Classifier         | Generate Ad Summary          | Same as above                                                                                                        |
| Save Inquiry Mail    | n8n-nodes-base.gmail                    | Adds Inquiry label                           | Text Classifier         | Generate Finance Summary     | Same as above                                                                                                        |
| Save Finance Mail    | n8n-nodes-base.gmail                    | Adds Finance/Billing label                   | Text Classifier         | Generate Inquiry Reply       | Same as above                                                                                                        |
| Generate Priority Draft | @n8n/n8n-nodes-langchain.openAi      | Generates draft reply for high priority mail | Save Priority Mail      | Create Draft                | Same as above                                                                                                        |
| Generate Ad Summary  | @n8n/n8n-nodes-langchain.openAi        | Summarizes advertisement emails             | Save Advertisement Mail | Append Row to Sheet          | Same as above                                                                                                        |
| Generate Finance Summary | @n8n/n8n-nodes-langchain.openAi      | Summarizes finance/billing emails            | Save Inquiry Mail       | Send Finance Mail           | Same as above                                                                                                        |
| Generate Inquiry Reply | @n8n/n8n-nodes-langchain.openAi       | Generates reply for inquiries                 | Save Finance Mail       | Reply to Inquiry            | Same as above                                                                                                        |
| Create Draft         | n8n-nodes-base.gmail                    | Creates Gmail draft reply                     | Generate Priority Draft | None                       | Same as above                                                                                                        |
| Append Row to Sheet  | n8n-nodes-base.googleSheets             | Logs advertisement summary to Google Sheets | Generate Ad Summary     | None                       | Same as above                                                                                                        |
| Send Finance Mail    | n8n-nodes-base.gmail                    | Sends reply mail in finance thread           | Generate Finance Summary| None                       | Same as above                                                                                                        |
| Reply to Inquiry     | n8n-nodes-base.gmail                    | Sends inquiry auto-reply                      | Generate Inquiry Reply  | None                       | Same as above                                                                                                        |
| Sticky Note          | n8n-nodes-base.stickyNote               | Documentation and overview                    | None                    | None                       | ## 📩Gmail Auto Reply & Categorization with AI (LangChain + OpenAI + Google Sheets) Detailed description and setup instructions |
| Sticky Note1         | n8n-nodes-base.stickyNote               | Documentation: Input reception overview      | None                    | None                       | ## ①Receive Gmail → AI understands the email content Incoming emails are analyzed by AI and automatically categorized into four sections |
| Sticky Note2         | n8n-nodes-base.stickyNote               | Documentation: Email processing overview     | None                    | None                       | ## ②Email Processing Section Detailed explanation of categorization and actions for each email type                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: `Gmail Trigger`  
   - Poll interval: Every minute  
   - Credentials: Connect Gmail OAuth2 account  
   - Output: Pass full email JSON including `text` and `id`.

2. **Add Text Classifier Node:**  
   - Type: `Text Classifier (LangChain)`  
   - Input text: `={{ $json.text }}` from Gmail Trigger  
   - Define categories with detailed descriptions for:  
     - High Priority  
     - Advertisement  
     - Inquiry  
     - Finance/Billing  
   - Connect Gmail Trigger main output to this node input.

3. **Add Gmail Labeling Nodes (4 total):**  
   - For each category, create a `Gmail` node to add the corresponding label:  
     - High Priority → Label ID: `Label_6395372519299544115`  
     - Advertisement → Label ID: `Label_2877175876700118240`  
     - Inquiry → Label ID: `Label_8971387974199578288`  
     - Finance/Billing → Label ID: `Label_7818902445747119871`  
   - Use message ID from input: `={{ $json.id }}`  
   - Connect each classification output from Text Classifier to respective labeling node.

4. **Add AI Processing Nodes (4 OpenAI nodes):**  
   - For each category, add an OpenAI node configured with GPT-4o:  
     - **High Priority:** Generate Priority Draft  
       - Prompt: Executive assistant style reply with subject and message.  
       - Input: Email text from Gmail Trigger.  
       - Connect output of Save Priority Mail to this node.  
     - **Advertisement:** Generate Ad Summary  
       - Prompt: Summarize sender, subject, summary, recommendation.  
       - Input: Email text.  
       - Connect Save Advertisement Mail output here.  
     - **Inquiry:** Generate Inquiry Reply  
       - Prompt: Generate reply with fallback instructions.  
       - Input: Email text.  
       - Connect Save Finance Mail output here (note: seems swapped in original; confirm and adjust accordingly).  
     - **Finance/Billing:** Generate Finance Summary  
       - Prompt: Summarize financial email and generate subject/message.  
       - Input: Email text.  
       - Connect Save Inquiry Mail output here (note: original connections swapped - verify and correct).

5. **Add Output Action Nodes:**  
   - **Create Draft (High Priority):**  
     - Type: Gmail node with resource set to Draft  
     - Subject: `={{ $json.message.content['send'] }}` (correct key for subject)  
     - Message: `={{ $json.message.content['message'] }}`  
     - Connect from Generate Priority Draft node.  
   - **Append Row to Google Sheets (Advertisement):**  
     - Map columns: Sender, Subject, Summary, Recommendation from AI output JSON fields.  
     - Document ID and Sheet Name: Use your Google Sheets’ spreadsheet ID and sheet gid.  
     - Connect from Generate Ad Summary node.  
   - **Send Finance Mail (Finance/Billing):**  
     - Type: Gmail node, operation reply in thread  
     - Use threadId and messageId from Save Inquiry Mail node output (confirm correct connection).  
     - Message content from Generate Finance Summary output.  
     - Connect from Generate Finance Summary node.  
   - **Reply to Inquiry (Inquiry):**  
     - Type: Gmail send mail node  
     - SendTo: Replace placeholder `{{YOUR_EMAIL_ADDRESS}}` with actual email address.  
     - Subject and Message from Generate Inquiry Reply output.  
     - Connect from Generate Inquiry Reply node.

6. **Setup Credentials:**  
   - Gmail OAuth2 for all Gmail nodes.  
   - OpenAI API credentials for all LangChain OpenAI nodes.  
   - Google Sheets OAuth2 credentials for Google Sheets node.

7. **Replace Placeholders:**  
   - Gmail label IDs must correspond to your actual Gmail labels.  
   - Spreadsheet ID and sheet gid in Google Sheets node must be your own.  
   - Replace `{{YOUR_EMAIL_ADDRESS}}` in Reply to Inquiry node with a real email address.

8. **Add Sticky Notes for Documentation (Optional):**  
   - Create sticky notes summarizing workflow purpose, setup instructions, and flow overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                               | Context or Link                                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| This workflow combines Gmail, LangChain text classification, OpenAI GPT-4o models, and Google Sheets to automate email management, reducing manual sorting and reply times. Replace placeholders carefully and create necessary Gmail labels before use.                    | Workflow overview sticky note content                                                                                       |
| Setup requires valid OAuth2 credentials for Gmail and Google Sheets, as well as an OpenAI API key configured in n8n.                                                                                                                                                       | Credential setup reminder                                                                                                   |
| Placeholder `{{YOUR_EMAIL_ADDRESS}}` in the Inquiry reply node must be replaced with the actual recipient email to ensure replies are sent correctly.                                                                                                                       | Important to avoid silent failure in sending Inquiry replies                                                                |
| Gmail label IDs in the workflow must be created in Gmail beforehand, and the IDs updated in the workflow nodes accordingly.                                                                                                                                               | Gmail label management                                                                                                       |
| Google Sheets document ID and sheet gid must reflect the target spreadsheet for Advertisement summaries.                                                                                                                                                                   | Google Sheets integration                                                                                                   |
| The workflow uses LangChain Text Classifier and OpenAI GPT-4o models; ensure your API quotas support expected volumes.                                                                                                                                                    | AI model usage considerations                                                                                                |
| The original workflow shows what appear to be swapped connections for Finance and Inquiry AI generation nodes; verify and test these to ensure replies and summaries are routed correctly.                                                                                  | Quality assurance and testing note                                                                                          |
| For detailed conceptual guidance, see the sticky notes embedded in the workflow which explain the classification categories and processing logic.                                                                                                                          | Internal documentation sticky notes                                                                                         |

---

**Disclaimer:** The content above is derived exclusively from an automated n8n workflow, fully compliant with content policies, and contains no illegal or protected material. All data handled are lawful and public.