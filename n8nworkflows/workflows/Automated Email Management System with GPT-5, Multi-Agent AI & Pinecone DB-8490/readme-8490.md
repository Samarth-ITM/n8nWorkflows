Automated Email Management System with GPT-5, Multi-Agent AI & Pinecone DB

https://n8nworkflows.xyz/workflows/automated-email-management-system-with-gpt-5--multi-agent-ai---pinecone-db-8490


# Automated Email Management System with GPT-5, Multi-Agent AI & Pinecone DB

### 1. Workflow Overview

This workflow, titled **"Automated Email Management System with GPT-5, Multi-Agent AI & Pinecone DB"**, automates the classification, routing, and response generation for incoming Gmail emails, leveraging advanced AI models and vector databases. It is designed for business owners or teams handling large volumes of emails, aiming to speed up email triage and handling across different functional areas such as customer support, finance, sales leads, and internal communications.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception and Trigger**: Captures incoming emails via Gmail trigger.
- **1.2 Email Classification**: Uses GPT-5 powered text classification to categorize emails into Customer Support, Finance, Sales Opportunities, Internal, or Others.
- **1.3 Multi-Agent AI Response Generation**: Dispatches emails to specialized AI agents (customer support, finance, sales leads, internal) for tailored processing and reply drafting.
- **1.4 Knowledge Base Integration via Pinecone DB**: Connects to vector databases containing FAQs, policies, and business services to assist AI agents in generating accurate replies.
- **1.5 Email Labeling and Routing**: Applies Gmail labels to emails based on classification results.
- **1.6 Notification and Escalation Handling**: Sends Telegram notifications for escalations, review requests, or financial alerts.
- **1.7 Finance Email Detailed Processing**: Further classifies finance-related emails as customer receipts or vendor invoices and routes them accordingly, including notifications to payment or receivables teams.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

- **Overview:** This block initiates the workflow by detecting new incoming emails in Gmail.
- **Nodes Involved:**  
  - Gmail Trigger  
  - Sticky Note4 (Trigger explanation)
  
- **Node Details:**

  - **Gmail Trigger**  
    - Type: Gmail Trigger  
    - Role: Watches Gmail inbox for new emails.  
    - Configuration: Uses OAuth2 Gmail credentials; polls continuously without filters to capture all incoming emails.  
    - Inputs: None (trigger node)  
    - Outputs: Email JSON data including headers, body, and metadata.  
    - Edge Cases: Gmail API rate limits, OAuth token expiration.  
    - Sticky Note4 content details the setup and trigger role.
    
  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Describes trigger setup and connection to Gmail API.  

---

#### 2.2 Email Classification

- **Overview:** Classifies incoming emails into predefined categories using GPT-5 powered text classification.
- **Nodes Involved:**  
  - GPT 5 mini  
  - Inbox Router  
  - Sticky Note5 (Inbox Router instructions)  
  - Sticky Note6 (General overview)  
  
- **Node Details:**

  - **GPT 5 mini**  
    - Type: Language Model Chat (OpenRouter)  
    - Role: Processes email content for classification.  
    - Configuration: Uses OpenRouter account with "openai/gpt-5-mini" model.  
    - Inputs: Raw email data (from Gmail Trigger).  
    - Outputs: Text classification data to Inbox Router.  
    - Edge Cases: API quota limits, malformed email content.

  - **Inbox Router**  
    - Type: Text Classifier (Langchain)  
    - Role: Classifies emails into categories based on "from", "subject", and body text.  
    - Configuration: Categories defined – Customer Support, Finance, Others, Sales Opportunities, Internal; each category has detailed descriptions and keywords to guide classification.  
    - Inputs: Output from GPT 5 mini.  
    - Outputs: Routed output to respective AI agents or No Operation node for "Others".  
    - Edge Cases: Ambiguous emails might be misclassified; fallback to "Others".  

  - **Sticky Note5**  
    - Details Inbox Router purpose and setup, emphasizing classification using subject, from, and body text.

  - **Sticky Note6**  
    - Provides high-level workflow summary, use cases, setup instructions for Telegram bot, OpenAI, OpenRouter, and Pinecone DB, and customization notes.

---

#### 2.3 Multi-Agent AI Response Generation

- **Overview:** Dedicated GPT-4o-mini agents process emails based on their category to generate tailored responses or actions.
- **Nodes Involved:**  
  - Customer Support Agent  
  - Finance Agent  
  - Leads Agent  
  - Internal Agent  
  - Various GPT 4o mini nodes (GPT 4o mini, GPT 4o mini1, GPT 4o mini2, GPT 4o mini3)  
  - Output Format nodes (OutputFormat, Output Format, Output Format1)  
  - Needs Review? (if escalation needed)  
  - No Operation, do nothing (for uncategorized emails)  
  - Sticky Notes 1, 2, 3 (explanations for each agent)
  
- **Node Details:**

  - **Customer Support Agent**  
    - Type: Langchain Agent  
    - Role: Responds to FAQs, policies, warranty questions using Pinecone DB knowledge.  
    - Configuration: System message instructs use of FAQ database, friendly tone, sign-off as "Customer Support Team", outputs only email body text.  
    - Input: Email text from Inbox Router routed as Customer Support.  
    - Output: Email reply content.  
    - Edge Cases: Missing or incomplete FAQ data; fallback to escalation (not explicitly shown).  
    - Connected Nodes: Labels email as Customer Support, sends reply email.

  - **Finance Agent**  
    - Type: Langchain Agent  
    - Role: Reviews finance-related emails, classifies as payment or receipt, summarizes key details.  
    - Configuration: System message includes classification instructions; uses structured output parser to parse email type, amount, sender, due date.  
    - Input: Email data routed from Inbox Router.  
    - Output: Parsed structured data for further routing.  
    - Edge Cases: Ambiguous email content, missing financial information.  
    - Connected Nodes: Labels email as Finance, routes by email type (receipt or vendor invoice).

  - **Leads Agent**  
    - Type: Langchain Agent  
    - Role: Handles sales inquiries, drafts replies based on business knowledge database, escalates unclear queries.  
    - Configuration: System message instructs identification of sales opportunities, response drafting, escalation.  
    - Input: Email data routed from Inbox Router.  
    - Output: Structured output with email body, reason, escalate boolean, knowledge database summary.  
    - Edge Cases: Insufficient knowledge base info leading to escalation.  
    - Connected Nodes: Needs Review?, Email Draft, Telegram notifications.

  - **Internal Agent**  
    - Type: Langchain Agent  
    - Role: Handles internal team emails, labels them, sends summary notifications.  
    - Configuration: System message instructs labeling and summarizing.  
    - Input: Email data routed from Inbox Router.  
    - Output: Summary for Telegram notification.  
    - Edge Cases: Misclassification of external emails as internal.  
    - Connected Nodes: Labels email as Internal, sends Telegram summary.

  - **GPT 4o mini nodes**  
    - Provide AI language model interface for the respective agents.

  - **Output Format nodes**  
    - Parse AI agent outputs into structured JSON formats for downstream processing.

  - **Needs Review?**  
    - Conditional node checks if Leads Agent output requires escalation.

  - **No Operation, do nothing**  
    - Placeholder node for emails categorized as "Others".

  - **Sticky Notes 1, 2, 3**  
    - Provide detailed explanations of each agent’s role and behavior, including notification and escalation details.

---

#### 2.4 Knowledge Base Integration via Pinecone DB

- **Overview:** Integrates vector databases containing domain-specific knowledge to enhance AI agent responses.
- **Nodes Involved:**  
  - Embeddings OpenAI (2 nodes)  
  - Database of FAQs and Policies  
  - Database of Business Services
  
- **Node Details:**

  - **Embeddings OpenAI**  
    - Type: Langchain OpenAI Embeddings  
    - Role: Generates text embeddings for knowledge base documents (FAQs/policies and business services).  
    - Configuration: Uses OpenAI API credentials; no special options set.  
    - Inputs: Documents (outside scope of current workflow JSON—assumed preloaded).  
    - Outputs: Embeddings passed to respective Pinecone vector stores.  
    - Edge Cases: API quota, embedding generation errors.

  - **Database of FAQs and Policies**  
    - Type: Langchain Pinecone Vector Store  
    - Role: Tool accessible by Customer Support Agent to retrieve relevant policy and FAQ info.  
    - Configuration: Namespace "FAQ", index "chatbot", mode "retrieve-as-tool".  
    - Inputs: Embeddings from Embeddings OpenAI node.  
    - Outputs: Knowledge snippets for agent use.  
    - Edge Cases: Missing or outdated knowledge data.

  - **Database of Business Services**  
    - Type: Langchain Pinecone Vector Store  
    - Role: Tool for Leads Agent to access knowledge about business services, packages, pricing, etc.  
    - Configuration: Index "chatbot", mode "retrieve-as-tool".  
    - Inputs: Embeddings from second Embeddings OpenAI node.  
    - Outputs: Knowledge snippets for Leads Agent.  
    - Edge Cases: Same as above.

---

#### 2.5 Email Labeling and Routing

- **Overview:** Applies Gmail labels to emails based on AI classification results to organize inbox.
- **Nodes Involved:**  
  - Label as Customer Support  
  - Label as Finance  
  - Label as Sales Opportunities  
  - Label as Internal
  
- **Node Details:**

  - **Label as Customer Support**  
    - Type: Gmail Node (addLabels)  
    - Role: Adds label "Customer Support" to relevant emails.  
    - Input: Email ID from Gmail Trigger.  
    - Output: Passes email forward for reply.  
    - Edge Cases: Label not existing in Gmail account, API permission issues.

  - **Label as Finance**  
    - Type: Gmail Node (addLabels)  
    - Role: Adds label "Finance" to finance emails.

  - **Label as Sales Opportunities**  
    - Type: Gmail Tool Node (addLabels)  
    - Role: Adds label "Sales Opportunities".

  - **Label as Internal**  
    - Type: Gmail Tool Node (addLabels)  
    - Role: Adds label "Internal".

---

#### 2.6 Notification and Escalation Handling

- **Overview:** Sends Telegram messages to the user or teams for notifications, escalations, and alerts.
- **Nodes Involved:**  
  - Send a message to user to review drafted reply  
  - Send a message to user to review the email directly  
  - Send a summary of email and from who  
  - Notify Amount Received (Telegram)  
  - Notify Amount to Pay (Telegram)  
  
- **Node Details:**

  - **Send a message to user to review drafted reply**  
    - Type: Telegram Node  
    - Role: Notifies user when an AI draft reply is ready for review, includes rationale and knowledge used.  
    - Inputs: From Leads Agent output.  
    - Edge Cases: Telegram API limits, incorrect chat ID.  
    - Requires user to input personal Telegram chat ID.

  - **Send a message to user to review the email directly**  
    - Type: Telegram Node  
    - Role: Notifies user to manually review email when AI is unsure how to respond.

  - **Send a summary of email and from who**  
    - Type: Telegram Node  
    - Role: Sends summary notification for internal emails.

  - **Notify Amount Received**  
    - Type: Telegram Node  
    - Role: Alerts user of incoming payment amounts from customer receipts.

  - **Notify Amount to Pay**  
    - Type: Telegram Node  
    - Role: Alerts user of payment amounts due to vendors.

---

#### 2.7 Finance Email Detailed Processing

- **Overview:** Further dissects finance emails into customer receipts or vendor invoices and routes notifications and emails accordingly.
- **Nodes Involved:**  
  - OutputFormat (structured output parser for Finance Agent)  
  - Route by Email Type (switch node)  
  - Send a message to Payments Team (email)  
  - Send a message to Receivables Team (email)  
  - Notify Amount Received (Telegram)  
  - Notify Amount to Pay (Telegram)  
  
- **Node Details:**

  - **OutputFormat**  
    - Type: Structured Output Parser  
    - Role: Parses Finance Agent's output into fields: emailType (boolean), emailBody (string), Amount (number), From (string), DueDate (string).  
    - Edge Cases: Parsing errors if AI output malformed.

  - **Route by Email Type**  
    - Type: Switch Node  
    - Role: Routes emails to either "Customer Receipts" or "Vendor Invoice" path based on emailType boolean.  

  - **Send a message to Payments Team**  
    - Type: Gmail Node (send email)  
    - Role: Sends payment request email to payments@payments.com with invoice details.  

  - **Send a message to Receivables Team**  
    - Type: Gmail Node (send email)  
    - Role: Sends receipt notification to receipt@receivables.com.  

  - **Notify Amount Received / Notify Amount to Pay**  
    - Telegram notifications correspond to these finance actions.

---

### 3. Summary Table

| Node Name                          | Node Type                              | Functional Role                           | Input Node(s)                | Output Node(s)                    | Sticky Note                                                                                                                              |
|-----------------------------------|--------------------------------------|-----------------------------------------|-----------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger                     | Gmail Trigger                        | Trigger on incoming emails               | None                        | Inbox Router                    | ## Trigger - Workflow is triggered when an email comes in - Connect to your Gmail account using Gmail API                               |
| Inbox Router                     | Text Classifier (Langchain)          | Classify email into categories           | Gmail Trigger, GPT 5 mini    | Customer Support Agent, Finance Agent, No Operation, Leads Agent, Internal Agent | ## Inbox Router - Based on incoming email's subject, from, and content, GPT routes email to categories                                    |
| GPT 5 mini                      | LM Chat OpenRouter                   | AI model for text classification          | Gmail Trigger               | Inbox Router                    | ## Categorise and route emails with GPT 5 - Multi-agent architecture overview                                                           |
| Customer Support Agent            | Langchain Agent                      | Generate replies for customer support    | Inbox Router, Database of FAQs and Policies | Reply Email, Label as Customer Support | ## Customer Support - Uses Pinecone FAQ DB to answer customer queries                                                                     |
| Finance Agent                    | Langchain Agent                      | Process finance emails, classify receipts/payments | Inbox Router, OutputFormat | Route by Email Type             | ## Finance Agent - Handles payment/receipt emails, notifies user and teams                                                              |
| Leads Agent                      | Langchain Agent                      | Handle sales leads, draft replies         | Inbox Router, Database of Business Services | Needs Review?                  | ## Sales/Leads Agent - Drafts replies or escalates if unsure                                                                            |
| Internal Agent                  | Langchain Agent                      | Handle internal team emails               | Inbox Router, Output Format1 | Label as Internal, Send summary message | ## Internal Agent - Labels internal emails and sends Telegram summary                                                                    |
| No Operation, do nothing         | NoOp                                | Placeholder for uncategorized emails      | Inbox Router                | None                           |                                                                                                                                         |
| Embeddings OpenAI                | Langchain Embeddings (OpenAI)       | Generate embeddings for knowledge bases   | External docs               | Database of FAQs and Policies, Database of Business Services |                                                                                                                                         |
| Database of FAQs and Policies    | Vector Store (Pinecone)              | Provide FAQ/policy knowledge to Customer Support Agent | Embeddings OpenAI           | Customer Support Agent          |                                                                                                                                         |
| Database of Business Services    | Vector Store (Pinecone)              | Provide business info to Leads Agent      | Embeddings OpenAI1          | Leads Agent                    |                                                                                                                                         |
| Reply Email                     | Gmail Reply                         | Send reply email for customer support     | Customer Support Agent      | Label as Customer Support       |                                                                                                                                         |
| Label as Customer Support        | Gmail Add Label                     | Label emails as Customer Support          | Reply Email                 | None                           |                                                                                                                                         |
| Label as Finance                | Gmail Add Label                     | Label finance emails                       | Finance Agent               | None                           |                                                                                                                                         |
| Label as Sales Opportunities     | Gmail Add Label                     | Label sales-related emails                 | Leads Agent                 | None                           |                                                                                                                                         |
| Label as Internal               | Gmail Add Label                     | Label internal emails                      | Internal Agent              | None                           |                                                                                                                                         |
| Needs Review?                   | If Condition                       | Check if Leads Agent output requires review | Leads Agent                 | Send a message to user to review the email directly, Email Draft |                                                                                                                                         |
| Email Draft                    | Gmail Draft                        | Draft email reply for leads                 | Needs Review?               | Send a message to user to review drafted reply |                                                                                                                                         |
| Send a message to user to review drafted reply | Telegram                          | Notify user to review drafted reply        | Email Draft                 | None                           | Note to input your personal chat ID in the telegram node                                                                                 |
| Send a message to user to review the email directly | Telegram                          | Notify user to review email when unsure    | Needs Review?               | None                           | Note to input your personal chat ID in the telegram node                                                                                 |
| Send a summary of email and from who | Telegram                          | Notify user of internal email summary      | Internal Agent              | None                           | Note to input your personal ChatID in the telegram node                                                                                  |
| OutputFormat                    | Structured Output Parser           | Parse Finance Agent output into structured data | Finance Agent               | Route by Email Type             |                                                                                                                                         |
| Route by Email Type             | Switch                            | Route finance emails as receipts or invoices | OutputFormat                | Send a message to Receivables Team, Send a message to Payments Team |                                                                                                                                         |
| Send a message to Payments Team | Gmail Send Email                  | Notify payments team to make a payment     | Route by Email Type         | Notify Amount to Pay            |                                                                                                                                         |
| Send a message to Receivables Team | Gmail Send Email                  | Notify receivables team of incoming payment | Route by Email Type         | Notify Amount Received          |                                                                                                                                         |
| Notify Amount Received          | Telegram                          | Notify user of received payment             | Send a message to Receivables Team | None                           | Note to input your personal chat ID in the telegram node                                                                                 |
| Notify Amount to Pay            | Telegram                          | Notify user of payment to be made           | Send a message to Payments Team | None                           | Note to input your personal chat ID in the telegram node                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Credentials: Connect your Gmail account via OAuth2.  
   - Configuration: No filters, continuous polling for new emails.

2. **Add GPT 5 mini node:**  
   - Type: Langchain LM Chat OpenRouter  
   - Credentials: OpenRouter API account configured.  
   - Model: "openai/gpt-5-mini".

3. **Add Inbox Router node:**  
   - Type: Langchain Text Classifier  
   - Input: Use the email’s "From", "Subject", and "Body" fields from Gmail Trigger and GPT 5 mini output.  
   - Categories: Configure with categories and descriptions: Customer Support, Finance, Sales Opportunities, Internal, Others.

4. **Connect Inbox Router outputs to four Langchain agents:**  
   - Customer Support Agent  
   - Finance Agent  
   - Leads Agent  
   - Internal Agent  
   
   Each agent uses GPT 4o mini model via separate Langchain LM Chat OpenRouter nodes:
   - GPT 4o mini for Customer Support  
   - GPT 4o mini1 for Finance  
   - GPT 4o mini2 for Leads  
   - GPT 4o mini3 for Internal

5. **Configure Customer Support Agent:**  
   - System message instructing friendly replies using Pinecone FAQ DB tool.  
   - Connect agent’s tool to "Database of FAQs and Policies" Pinecone vector store.  
   - Connect output to Gmail Reply node and Label as Customer Support node.

6. **Configure Finance Agent:**  
   - System message for finance email classification and summary.  
   - Connect to OutputFormat node (structured output parser) with schema for emailType (boolean), Amount, From, DueDate.  
   - Connect OutputFormat to switch node "Route by Email Type" splitting into Customer Receipts and Vendor Invoice.  
   - Connect switch outputs to send email nodes to Receivables Team and Payments Team respectively.  
   - Connect these send email nodes to Telegram notification nodes notifying user of amounts.

7. **Configure Leads Agent:**  
   - System message instructing sales inquiry handling and escalation.  
   - Connect to "Database of Business Services" Pinecone vector store.  
   - Connect output to structured output parser "Output Format".  
   - Connect Output Format to "Needs Review?" if node checking "escalate" boolean.  
   - If escalation needed, send Telegram notification to user to review email directly.  
   - If no escalation, draft email in Gmail Draft node and notify user via Telegram to review drafted reply.

8. **Configure Internal Agent:**  
   - System message for internal emails.  
   - Connect to structured output parser "Output Format1" to extract summary.  
   - Connect to Label as Internal node and Telegram node sending summary message.

9. **Set up Pinecone vector stores:**  
   - Create two Pinecone indexes/namespaces: one for FAQs and Policies ("FAQ"), one for Business Services.  
   - Use Embeddings OpenAI nodes to generate embeddings for documents related to FAQs/policies and business services respectively.

10. **Configure Gmail nodes for labeling:**  
    - Label as Customer Support (label ID for "Customer Support")  
    - Label as Finance (label ID for "Finance")  
    - Label as Sales Opportunities (label ID for "Sales Opportunities")  
    - Label as Internal (label ID for "Internal")  
    - Use messageId from Gmail Trigger for label operations.

11. **Configure Telegram nodes:**  
    - Create Telegram bot and obtain API token.  
    - Set user chat IDs in text fields for personal notifications (review requests, summaries, financial alerts).  
    - Use Telegram nodes for notifications to the user and for finance-related notifications.

12. **Connect nodes according to the workflow’s connection logic:**  
    - Gmail Trigger → GPT 5 mini → Inbox Router  
    - Inbox Router → respective AI agents  
    - AI agents → appropriate output parsers → logical next steps (labeling, replies, notifications)  
    - Finance Agent outputs → OutputFormat → Route by Email Type → team emails + Telegram notifications  
    - Leads Agent output → Needs Review? → Email Draft + Telegram notifications or direct Telegram notification for review  
    - Internal Agent → Label as Internal + summary Telegram notification  
    - Customer Support Agent → Reply Email + Label as Customer Support

13. **Validate all credentials:**  
    - Gmail OAuth2 with permissions to read, add labels, send emails, and create drafts.  
    - OpenAI API for embeddings.  
    - OpenRouter API for GPT models.  
    - Pinecone API v2 credentials for vector database access.  
    - Telegram API credentials with bot token.

14. **Test the workflow:**  
    - Send test emails representing each category and verify classification, labeling, replies, and notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                            | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Set up Telegram bot via Botfather. Instructions: https://core.telegram.org/bots/tutorial                                                                                                                                                                | Telegram bot setup instructions                                                                            |
| Setup OpenAI API for embeddings and language models. Details: https://openai.com/index/openai-api/                                                                                                                                                        | OpenAI API documentation                                                                                   |
| Set up OpenRouter account for GPT model access: https://openrouter.ai/docs/api-reference/authentication                                                                                                                                                   | OpenRouter API authentication guide                                                                       |
| Set up Pinecone vector database for knowledge storage: https://docs.pinecone.io/guides/get-started/quickstart                                                                                                                                             | Pinecone DB quickstart guide                                                                                |
| The workflow supports extension to other email providers such as Outlook and alternative vector databases like Supabase, Qdrant, Weaviate.                                                                                                               | Customization notes                                                                                        |
| Personal Telegram chat ID must be input into all Telegram nodes to enable notifications to the user.                                                                                                                                                       | Critical configuration note                                                                                 |
| Gmail labels used must exist in your Gmail account prior to configuring label nodes; create them manually if necessary.                                                                                                                                    | Gmail label management                                                                                      |
| The multi-agent architecture allows for modular handling of diverse email types, improving scalability and maintainability.                                                                                                                               | Architectural note                                                                                          |
| This workflow does not handle email attachments or embedded images; additional nodes and logic are required for such cases.                                                                                                                               | Limitation                                                                                                 |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow built with n8n, complying fully with content policies and containing no illegal, offensive, or protected elements. All data handled is legal and public.