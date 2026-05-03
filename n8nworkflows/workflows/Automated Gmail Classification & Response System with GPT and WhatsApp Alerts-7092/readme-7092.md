Automated Gmail Classification & Response System with GPT and WhatsApp Alerts

https://n8nworkflows.xyz/workflows/automated-gmail-classification---response-system-with-gpt-and-whatsapp-alerts-7092


# Automated Gmail Classification & Response System with GPT and WhatsApp Alerts

### 1. Workflow Overview

This workflow automates the classification and response process for incoming Gmail emails using OpenAI GPT models and sends alerts via WhatsApp. It is designed for efficient email management by categorizing emails into defined classes, applying appropriate labels in Gmail, generating tailored replies or summaries, and notifying stakeholders for urgent or important communications.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Detect new incoming emails in Gmail via a trigger.
- **1.2 Email Classification:** Use OpenAI to classify emails into five categories.
- **1.3 Gmail Labeling:** Apply Gmail labels corresponding to each classification category.
- **1.4 AI-Generated Responses/Summaries:** Generate emails drafts, replies, or summaries tailored to the category.
- **1.5 Email Drafting and Sending:** Create drafts or send replies in Gmail.
- **1.6 WhatsApp Alerts:** Notify via WhatsApp for high priority, customer support, promotional, and finance emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow by detecting new emails arriving in a Gmail inbox, polling every minute.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Sticky Note (Instruction)

- **Node Details:**  
  - **Gmail Trigger**  
    - Type: Gmail Trigger node  
    - Role: Watches Gmail inbox for new emails, triggers workflow per new email  
    - Configuration: Polls every minute, no specific filters applied (all new emails captured)  
    - Inputs: None (start node)  
    - Outputs: Email metadata and snippet to downstream nodes  
    - Edge Cases: OAuth token expiry, Gmail API rate limits, network issues  
    - Credentials: OAuth2 Gmail account configured  
  - **Sticky Note**  
    - Content: Reminder to set up Gmail credentials and filters if needed

---

#### 2.2 Email Classification

- **Overview:**  
  Using OpenAI GPT-4.1-mini via Langchain, the workflow classifies the incoming email snippet into one of five categories: High Priority, Customer Support, Promotion, Finance/Billing, or Random/General.

- **Nodes Involved:**  
  - OpenAI Chat Model (Langchain)  
  - Email Classifier (Langchain Text Classifier)  
  - Sticky Note (Instruction)

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node  
    - Role: Acts as a language model interface required by the text classifier  
    - Configuration: Model set to GPT-4.1-mini  
    - Inputs: None directly; used as a resource for Email Classifier node  
    - Outputs: Model instance passed to classifier  
    - Credentials: OpenAI API key  
  - **Email Classifier**  
    - Type: Langchain Text Classifier node  
    - Role: Classifies email snippet into predefined categories with detailed descriptions for each  
    - Configuration: Takes snippet from Gmail Trigger, classifies into 5 categories, each with description and examples  
    - Inputs: Email snippet JSON field from Gmail Trigger  
    - Outputs: Routes workflow into one of five paths based on classification  
    - Edge Cases: Misclassification, model API timeouts, empty or malformed email snippets  
    - Depends on: OpenAI Chat Model node  
  - **Sticky Note1**  
    - Content: Explains the five email categories used for classification

---

#### 2.3 Gmail Labeling

- **Overview:**  
  Based on classification, the workflow applies corresponding Gmail labels to the incoming email for organizational purposes.

- **Nodes Involved:**  
  - High Priority Email  
  - Customer Support Email  
  - Promotion Email  
  - Finance/Billing Email  
  - Random (General/Other) Email  
  - Sticky Notes (Instruction)

- **Node Details:**  
  Each node:  
  - Type: Gmail node (to modify labels)  
  - Role: Adds Gmail label to the email identified by messageId from the Gmail Trigger  
  - Configuration:  
    - Label IDs are hardcoded for each category (e.g., High Priority label ID for high priority emails)  
    - Operation: Add label to message  
  - Inputs: Message ID and email data from Gmail Trigger  
  - Outputs: Trigger subsequent processing nodes for each category  
  - Credentials: Same Gmail OAuth2 account  
  - Edge Cases: Label ID invalid or deleted, Gmail API errors, message ID missing  
  - Sticky Notes attached:  
    - High Priority Email node accompanied by Sticky Note2: describes labeling, draft creation, WhatsApp alert  
    - Customer Support Email with Sticky Note3: describes label, auto-reply, WhatsApp confirmation  
    - Promotion Email with Sticky Note4: label, summary, WhatsApp digest  
    - Finance/Billing Email with Sticky Note5: label, summary, notification via Gmail and WhatsApp  
    - Random (General/Other) Email with Sticky Note6: label only, no further action

---

#### 2.4 AI-Generated Responses and Summaries

- **Overview:**  
  For each category (except Random), the workflow uses OpenAI GPT models to generate appropriate email drafts, replies, or summaries customized to the email's nature.

- **Nodes Involved:**  
  - Creating a Draft (High Priority)  
  - Create Email (Customer Support)  
  - Summarize Promotions (Promotion)  
  - Finance Department (Finance/Billing)  
  - Sticky Notes (Instruction)

- **Node Details:**  
  - **Creating a Draft**  
    - Type: Langchain OpenAI node  
    - Role: Generates a professional draft reply for high priority emails  
    - Configuration: Uses GPT-4.1-mini with system prompt instructing to produce subject and message in a strict output format  
    - Inputs: Email snippet from Gmail Trigger  
    - Outputs: JSON containing subject and message for draft creation  
    - Edge Cases: Model API failure, prompt misunderstanding  
  - **Create Email**  
    - Type: Langchain OpenAI node  
    - Role: Generates auto-reply emails for customer support inquiries, addressing customer by name, polite and professional  
    - Configuration: GPT-4.1-mini, detailed prompt with required tone and closing signature  
    - Inputs: Email snippet from Gmail Trigger  
    - Outputs: JSON with reply message content  
  - **Summarize Promotions**  
    - Type: Langchain OpenAI node  
    - Role: Summarizes promotional emails into concise actionable summaries  
    - Configuration: GPT-4.1-mini with prompt focusing on key offers, expiry, conditions  
    - Inputs: Email snippet  
  - **Finance Department**  
    - Type: Langchain OpenAI node  
    - Role: Summarizes finance/billing emails for forwarding to finance department  
    - Configuration: GPT-4.1-mini, formal summary format with subject and text  
    - Inputs: Email snippet  
  - **Sticky Notes**  
    Provide instructions for each AI generation node’s role and expected output

---

#### 2.5 Email Drafting and Sending

- **Overview:**  
  This block creates drafts or sends replies in Gmail based on AI-generated content.

- **Nodes Involved:**  
  - Draft email (create draft for high priority)  
  - Auto Reply (send reply for customer support)  
  - Send to Finance Department (send summarized finance email)  
  - Sticky Notes (related to these actions)

- **Node Details:**  
  - **Draft email**  
    - Type: Gmail node (draft creation)  
    - Role: Creates a draft email in Gmail with subject and body generated by AI for high priority emails  
    - Inputs: Draft content and subject from Creating a Draft node  
    - Outputs: Triggers WhatsApp alert  
  - **Auto Reply**  
    - Type: Gmail node (reply to email)  
    - Role: Sends reply email to original sender for customer support emails  
    - Inputs: Reply message content from Create Email node, message ID from Gmail Trigger  
    - Outputs: Triggers WhatsApp confirmation  
  - **Send to Finance Department**  
    - Type: Gmail node (send email)  
    - Role: Sends summarized finance email to designated finance email address  
    - Inputs: Subject and text summary from Finance Department node  
    - Outputs: Triggers WhatsApp finance alert  
  - Edge Cases: Gmail sending limits, draft save errors, invalid message IDs

---

#### 2.6 WhatsApp Alerts

- **Overview:**  
  Sends WhatsApp messages notifying relevant parties about email processing results, focusing on high priority, customer support, promotion, and finance emails.

- **Nodes Involved:**  
  - High Priority Alert  
  - Confirmation (Customer Support)  
  - Promotional Alert  
  - Finance Alert  
  - Sticky Notes (related to alerts)

- **Node Details:**  
  - All are WhatsApp nodes sending text messages  
  - Messages include email snippet and AI-generated replies or summaries  
  - Phone number ID and recipient phone number must be configured (placeholders present)  
  - Credentials: WhatsApp API credentials configured  
  - Edge Cases: WhatsApp API rate limits, invalid phone numbers, message failures  
  - Sticky Notes describe the purpose of each alert node

---

### 3. Summary Table

| Node Name              | Node Type                           | Functional Role                                 | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                  |
|------------------------|-----------------------------------|------------------------------------------------|-------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------|
| Gmail Trigger          | Gmail Trigger                     | Detect new incoming emails                      | None                    | Email Classifier          | Starts the workflow when a new email is received. Ensure Gmail credentials and filters are configured.        |
| OpenAI Chat Model      | Langchain OpenAI Chat Model       | Provide language model for classification      | None                    | Email Classifier          |                                                                                                              |
| Email Classifier       | Langchain Text Classifier         | Classify email content into 5 categories       | Gmail Trigger, OpenAI Chat Model | High Priority Email, Customer Support Email, Promotion Email, Finance/Billing Email, Random Email | Classifies emails into High Priority, Customer Support, Promotion, Finance/Billing, Random/General categories. |
| High Priority Email    | Gmail                            | Add High Priority Gmail label                   | Email Classifier        | Creating a Draft          | Adds "High Priority" label, drafts reply, and sends WhatsApp alert.                                           |
| Creating a Draft       | Langchain OpenAI                  | Generate draft reply for high priority emails  | High Priority Email     | Draft email               |                                                                                                              |
| Draft email            | Gmail                            | Create draft email in Gmail                      | Creating a Draft        | High Priority Alert       |                                                                                                              |
| High Priority Alert    | WhatsApp                         | Send WhatsApp alert for high priority emails   | Draft email             | None                      |                                                                                                              |
| Customer Support Email | Gmail                            | Add Customer Support Gmail label                | Email Classifier        | Create Email              | Adds "Customer Support" label, generates reply, sends WhatsApp confirmation.                                 |
| Create Email           | Langchain OpenAI                  | Generate auto-reply for customer support emails| Customer Support Email  | Auto Reply                |                                                                                                              |
| Auto Reply             | Gmail                            | Send reply email to customer                    | Create Email            | Confirmation              |                                                                                                              |
| Confirmation           | WhatsApp                         | Send WhatsApp confirmation for support replies | Auto Reply              | None                      |                                                                                                              |
| Promotion Email        | Gmail                            | Add Promotion Gmail label                       | Email Classifier        | Summarize Promotions      | Labels email as "Promotion," summarizes content, sends WhatsApp alert.                                       |
| Summarize Promotions   | Langchain OpenAI                  | Summarize promotional email content             | Promotion Email         | Promotional Alert         |                                                                                                              |
| Promotional Alert      | WhatsApp                         | Send WhatsApp alert for promotions             | Summarize Promotions    | None                      |                                                                                                              |
| Finance/Billing Email  | Gmail                            | Add Finance/Billing Gmail label                 | Email Classifier        | Finance Department        | Labels as "Finance/Billing," prepares summary, sends email and WhatsApp alert.                               |
| Finance Department     | Langchain OpenAI                  | Summarize finance email content                 | Finance/Billing Email   | Send to Finance Department |                                                                                                              |
| Send to Finance Department | Gmail                        | Send summarized finance email                    | Finance Department      | Finance Alert             |                                                                                                              |
| Finance Alert          | WhatsApp                         | Send WhatsApp alert about finance email         | Send to Finance Department | None                    |                                                                                                              |
| Random (General/Other) Email | Gmail                      | Add Random/General label, no further action     | Email Classifier        | None                      | Labels unclassified emails as “Random.” No further automation.                                               |
| Sticky Notes (Multiple)| Sticky Note                     | Instructions and explanations                    | None                    | None                      | Various notes explaining workflow sections and nodes.                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Set to poll every minute (default)  
   - Configure with Gmail OAuth2 credentials  
   - No filters (captures all new emails)  

2. **Add OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Select model: GPT-4.1-mini  
   - Configure OpenAI API credentials  

3. **Add Email Classifier Node**  
   - Type: Langchain Text Classifier  
   - Input text: Expression `{{$json.snippet}}` from Gmail Trigger  
   - Categories and descriptions:  
     - High Priority  
     - Customer Support  
     - Promotion  
     - Finance/Billing  
     - Random (General/Other)  
   - Connect OpenAI Chat Model node as the language model resource  
   - Connect Gmail Trigger output to Email Classifier input  

4. **Add Gmail Nodes for Labeling (5 nodes)**  
   - For each category add a Gmail node:  
     - Operation: Add label  
     - Message ID: `={{$('Gmail Trigger').item.json.id}}`  
     - Label IDs: Use appropriate Gmail label IDs for each category  
   - Connect each Email Classifier output to corresponding Gmail node  

5. **High Priority Email Branch**  
   - Add Langchain OpenAI node “Creating a Draft”  
     - Model: GPT-4.1-mini  
     - Prompt: Executive Assistant drafting professional responses  
     - Input: Email snippet from Gmail Trigger  
   - Add Gmail node “Draft email”  
     - Operation: Create draft  
     - Subject: `={{$json.message.content.Subject}}` from Creating a Draft  
     - Message: `={{$json.message.content.Message}}`  
   - Add WhatsApp node “High Priority Alert”  
     - Text message includes snippet and draft message  
     - Configure WhatsApp credentials and phone numbers  
   - Connect nodes sequentially: High Priority Email → Creating a Draft → Draft email → High Priority Alert  

6. **Customer Support Branch**  
   - Add Langchain OpenAI node “Create Email”  
     - Model: GPT-4.1-mini  
     - Prompt: Polite, professional customer support reply with customer name  
     - Input: Email snippet  
   - Add Gmail node “Auto Reply”  
     - Operation: Reply to original email  
     - Message: `={{$json.message.content}}` from Create Email  
   - Add WhatsApp node “Confirmation”  
     - Text with snippet and reply message  
   - Connect nodes: Customer Support Email → Create Email → Auto Reply → Confirmation  

7. **Promotion Branch**  
   - Add Langchain OpenAI node “Summarize Promotions”  
     - Model: GPT-4.1-mini  
     - Prompt: Summarize key offers, expiry, conditions  
   - Add WhatsApp node “Promotional Alert”  
     - Text with snippet and summary  
   - Connect: Promotion Email → Summarize Promotions → Promotional Alert  

8. **Finance/Billing Branch**  
   - Add Langchain OpenAI node “Finance Department”  
     - Model: GPT-4.1-mini  
     - Prompt: Summarize finance-related emails professionally  
   - Add Gmail node “Send to Finance Department”  
     - Send email to configured finance email address  
     - Subject and message from Finance Department node output  
   - Add WhatsApp node “Finance Alert”  
     - Text includes snippet and confirmation of notification sent  
   - Connect: Finance/Billing Email → Finance Department → Send to Finance Department → Finance Alert  

9. **Random/General Branch**  
   - Add Gmail node “Random (General/Other) Email”  
     - Label email as Random, no further action  

10. **Set Up Credentials**  
    - Gmail OAuth2 for all Gmail nodes  
    - OpenAI API key for Langchain OpenAI nodes  
    - WhatsApp API credentials for WhatsApp nodes  
    - Replace placeholder phone numbers (`[YOUR_WHATSAPP_NUMBER]`) and finance email (`[YOUR_GMAIL_ADDRESS]`) with actual values  

11. **Add Sticky Notes for Documentation** (optional but recommended)  
    - Document each block with relevant notes as per the original workflow  

12. **Test Workflow**  
    - Send test emails matching each category  
    - Verify labels, draft/reply generation, email sending, and WhatsApp alerts  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Ensure Gmail labels exist and label IDs are correctly configured in Gmail before running workflow.                                                     | Gmail label management UI                                                                       |
| OpenAI API usage requires appropriate billing and token limits; monitor usage to avoid interruptions.                                                  | https://platform.openai.com/account/usage                                                      |
| WhatsApp API requires business account and phone number registration; configure phoneNumberId and recipientPhoneNumber correctly.                    | https://developers.facebook.com/docs/whatsapp/                                                 |
| This workflow uses GPT-4.1-mini model via Langchain integration in n8n; ensure your n8n instance supports these node types and versions.              | n8n documentation on Langchain nodes                                                           |
| Workflow designed for business email automation; customize prompts and labels as per your organization's policies and style guides.                  |                                                                                                 |
| Sticky notes in workflow provide inline documentation; maintain them updated when modifying workflow.                                                  |                                                                                                 |

---

This completes the detailed technical reference for the "Automated Gmail Classification & Response System with GPT and WhatsApp Alerts" workflow, enabling thorough understanding, reproduction, and modification.