Classify and Auto-Reply to Emails with Outlook and GPT-4

https://n8nworkflows.xyz/workflows/classify-and-auto-reply-to-emails-with-outlook-and-gpt-4-10583


# Classify and Auto-Reply to Emails with Outlook and GPT-4

### 1. Workflow Overview

This workflow is designed to automate the classification and response process for incoming emails in Microsoft Outlook using AI (GPT-4). It targets professionals and teams managing high email volumes who require efficient triage and personalized auto-responses based on email urgency.

The workflow is structured into four main logical blocks:

- **1.1 Email Input Reception:** Watches the Outlook inbox for new emails, downloads attachments, and captures relevant metadata.
- **1.2 AI Email Classification:** Uses AI to classify the email into three categories based on urgency: Urgent, Important, or Not Important, considering context such as deadlines and email content.
- **1.3 AI-Driven Reply Generation:** For each classification, a tailored AI-generated email reply is created with a specific tone and style appropriate to the category.
- **1.4 Email Organization and Dispatch:** Sends the AI-generated reply back through Outlook, then moves the original email to the appropriate Outlook folder (Urgent, Important, or Not Important) for easy inbox management.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Input Reception

- **Overview:**  
  This block triggers the workflow when a new email arrives in the Outlook inbox. It downloads any attachments and reads all email metadata, preparing the email data for classification.

- **Nodes Involved:**  
  - Microsoft Outlook Trigger

- **Node Details:**

  - **Microsoft Outlook Trigger**  
    - Type: Trigger node that monitors Outlook inbox  
    - Configuration:  
      - Polls every minute for new emails.  
      - Downloads attachments automatically.  
      - No filter on incoming emails (processes all).  
    - Key Expressions/Variables: None (raw email data output)  
    - Input Connections: None (trigger node)  
    - Output Connections: Connected to AI Email Classifier node  
    - Edge Cases:  
      - Outlook API rate limits or connectivity issues.  
      - Large attachments causing timeouts.  
      - Emails with unusual encoding or malformed data.  

---

#### 2.2 AI Email Classification

- **Overview:**  
  This block analyzes the email body to classify the email into one of three categories: Urgent, Important, or Not Important, based on urgency, deadlines, and content indicators. It uses GPT-4 mini for fast and accurate classification.

- **Nodes Involved:**  
  - OpenAI Chat Model - Classifier  
  - AI Email Classifier

- **Node Details:**

  - **OpenAI Chat Model - Classifier**  
    - Type: AI Language Model node (GPT-4 mini)  
    - Configuration: Uses GPT-4 mini model for classification prompt processing  
    - Input: Receives email body preview text  
    - Output: Passes response to AI Email Classifier node for final category determination  
    - Edge Cases: API errors, slow response, or incomplete classification due to ambiguous email content

  - **AI Email Classifier**  
    - Type: Text classification node using Langchain AI  
    - Configuration:  
      - System prompt defines three categories with detailed instructions on urgency and importance.  
      - Uses today's date dynamically (`{{ $today }}`) for contextual deadline evaluation.  
      - Input text is the email body preview.  
      - Categories defined explicitly: Urgent, Important, Not Important.  
    - Input: Receives email body from Outlook Trigger  
    - Output: Routes to three distinct paths based on classification result: Urgent, Important, Not Important  
    - Edge Cases:  
      - Misclassification due to ambiguous wording.  
      - Failure to detect urgency or importance when metadata is lacking or vague.  
      - API or prompt errors.

---

#### 2.3 AI-Driven Reply Generation

- **Overview:**  
  Based on the classification, this block generates a personalized email reply tailored to each category’s tone and urgency level using GPT-4 agents and sends the reply back via Outlook.

- **Nodes Involved:**  
  - Generate Urgent Reply  
  - OpenAI Chat Model - Urgent  
  - Reply to Urgent Email  
  - Generate Important Reply  
  - OpenAI Chat Model - Important  
  - Reply to Important Email  
  - Generate Standard Reply  
  - OpenAI Chat Model - Standard  
  - Reply to Not Important Email

- **Node Details:**

  - **Generate Urgent Reply**  
    - Type: Langchain AI Agent node generating reply text  
    - Configuration:  
      - Prompt instructs to write a brief, formal, first-person reply emphasizing urgency and rapid handling.  
      - Uses email body preview as input for context.  
    - Input: Email content from AI Email Classifier (Urgent path)  
    - Output: Reply text to be sent via Outlook  
    - Edge Cases: AI generating repetitive or inappropriate responses; API timeouts

  - **OpenAI Chat Model - Urgent**  
    - Type: GPT-4 mini language model node supporting Generate Urgent Reply  
    - Input/Output: AI language model invoked by Generate Urgent Reply  
    - Edge Cases: API failures or response delays

  - **Reply to Urgent Email**  
    - Type: Microsoft Outlook node to send reply  
    - Configuration:  
      - Replies to the original email ID.  
      - Message content is the AI-generated reply.  
      - Reply sent only to original sender.  
      - Sets "replyTo" field to original sender’s email.  
    - Input: Generated reply text from Generate Urgent Reply  
    - Output: Passes email message for folder movement  
    - Edge Cases: Outlook API errors, invalid message ID, permission issues

  - **Generate Important Reply**  
    - Type: Langchain AI Agent node generating reply text  
    - Configuration:  
      - Prompt instructs a balanced, formal reply indicating timely processing without urgency.  
      - Uses email body preview for context.  
    - Input: Email content from AI Email Classifier (Important path)  
    - Output: Reply text for sending  
    - Edge Cases: Similar to Urgent reply generation

  - **OpenAI Chat Model - Important**  
    - Type: GPT-4 mini language model node supporting Generate Important Reply  
    - Edge Cases: Same as above

  - **Reply to Important Email**  
    - Type: Microsoft Outlook node to send reply  
    - Configuration similar to Reply to Urgent Email, adjusted for Important category  
    - Edge Cases: Same as above

  - **Generate Standard Reply**  
    - Type: Langchain AI Agent node generating reply text  
    - Configuration:  
      - Prompt instructs polite, brief response indicating processing "in due time" without implying low priority.  
      - Uses email body preview for context.  
    - Input: Email content from AI Email Classifier (Not Important path)  
    - Output: Reply text for sending  
    - Edge Cases: Similar to other reply generations

  - **OpenAI Chat Model - Standard**  
    - Type: GPT-4 mini model supporting Generate Standard Reply  
    - Edge Cases: Same as above

  - **Reply to Not Important Email**  
    - Type: Outlook node to send reply  
    - Configuration similar to other reply nodes  
    - Edge Cases: Same as above

---

#### 2.4 Email Organization and Dispatch

- **Overview:**  
  After replies are sent, emails are moved to corresponding folders (Urgent, Important, Not Important) for efficient inbox organization.

- **Nodes Involved:**  
  - Move to Urgent Folder  
  - Move to Important Folder  
  - Move to Not Important Folder

- **Node Details:**

  - **Move to Urgent Folder**  
    - Type: Microsoft Outlook operation node  
    - Configuration: Moves email to the configured "URGENT" folder using folder ID.  
    - Input: Email ID from AI Email Classifier and reply nodes  
    - Edge Cases: Folder ID invalid or missing, permission errors, API call failures

  - **Move to Important Folder**  
    - Type: Outlook node moving email to "Important" folder  
    - Edge Cases: Same as above

  - **Move to Not Important Folder**  
    - Type: Outlook node moving email to "Not Important" folder  
    - Edge Cases: Same as above

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                     | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                                     |
|----------------------------|--------------------------------------|-----------------------------------|-----------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Microsoft Outlook Trigger   | microsoftOutlookTrigger               | Email input reception trigger     | None                        | AI Email Classifier             | ## 1. Email Trigger: Monitors inbox every minute, downloads attachments, passes email data to classifier.      |
| OpenAI Chat Model - Classifier | lmChatOpenAi (GPT-4 mini)            | AI model for email classification | Microsoft Outlook Trigger    | AI Email Classifier             | ## 2. AI Email Classification: Classifies email urgency using GPT-4 mini with today's date context.             |
| AI Email Classifier         | textClassifier                       | Classifies emails into categories | Microsoft Outlook Trigger, OpenAI Chat Model - Classifier | Generate Urgent Reply, Generate Important Reply, Generate Standard Reply | ## 2. AI Email Classification (continued): Routes emails based on classification to appropriate reply generation. |
| Generate Urgent Reply       | agent                               | Generates urgent email reply      | AI Email Classifier (Urgent path) | Reply to Urgent Email           | ## 3A. Urgent Email Path: Formal, brief, urgent tone reply emphasizing rapid processing.                        |
| OpenAI Chat Model - Urgent  | lmChatOpenAi (GPT-4 mini)            | AI model supporting urgent reply  | Generate Urgent Reply        | Generate Urgent Reply           | ## 3A. Urgent Email Path (continued)                                                                           |
| Reply to Urgent Email       | microsoftOutlook                    | Sends urgent reply email          | Generate Urgent Reply        | Move to Urgent Folder           | ## 3A. Urgent Email Path (continued)                                                                           |
| Move to Urgent Folder       | microsoftOutlook                    | Moves email to URGENT folder      | Reply to Urgent Email        | None                          | ## 4. Email Organization: Routes emails to dedicated folders for easy management.                              |
| Generate Important Reply    | agent                               | Generates important email reply   | AI Email Classifier (Important path) | Reply to Important Email        | ## 3B. Important Email Path: Balanced, professional reply with neutral timeline commitment.                    |
| OpenAI Chat Model - Important | lmChatOpenAi (GPT-4 mini)            | AI model supporting important reply | Generate Important Reply     | Generate Important Reply        | ## 3B. Important Email Path (continued)                                                                        |
| Reply to Important Email    | microsoftOutlook                    | Sends important reply email       | Generate Important Reply     | Move to Important Folder         | ## 3B. Important Email Path (continued)                                                                        |
| Move to Important Folder    | microsoftOutlook                    | Moves email to Important folder   | Reply to Important Email     | None                          | ## 4. Email Organization (continued)                                                                            |
| Generate Standard Reply     | agent                               | Generates not important email reply | AI Email Classifier (Not Important path) | Reply to Not Important Email    | ## 3C. Not Important Email Path: Polite, brief reply with “in due time” messaging.                              |
| OpenAI Chat Model - Standard | lmChatOpenAi (GPT-4 mini)            | AI model supporting standard reply | Generate Standard Reply      | Generate Standard Reply         | ## 3C. Not Important Email Path (continued)                                                                    |
| Reply to Not Important Email | microsoftOutlook                    | Sends not important reply email   | Generate Standard Reply      | Move to Not Important Folder     | ## 3C. Not Important Email Path (continued)                                                                    |
| Move to Not Important Folder | microsoftOutlook                    | Moves email to Not Important folder | Reply to Not Important Email | None                          | ## 4. Email Organization (continued)                                                                            |
| Sticky Note                 | stickyNote                         | Provides overview and context     | None                        | None                          | ## AI-Powered Email Classification and Auto-Reply System: Overview, use cases, setup, and customization tips.  |
| Sticky Note1                | stickyNote                         | Describes Email Trigger block     | None                        | None                          | ## 1. Email Trigger explanations                                                                                 |
| Sticky Note2                | stickyNote                         | Describes AI Classification block | None                        | None                          | ## 2. AI Email Classification explanations                                                                     |
| Sticky Note3                | stickyNote                         | Describes Urgent Email Path       | None                        | None                          | ## 3A. Urgent Email Path explanations                                                                            |
| Sticky Note4                | stickyNote                         | Describes Important Email Path    | None                        | None                          | ## 3B. Important Email Path explanations                                                                         |
| Sticky Note5                | stickyNote                         | Describes Not Important Email Path| None                        | None                          | ## 3C. Not Important Email Path explanations                                                                     |
| Sticky Note6                | stickyNote                         | Describes Email Organization block| None                        | None                          | ## 4. Email Organization explanations                                                                            |
| Sticky Note7                | stickyNote                         | Important configuration notes     | None                        | None                          | ## ⚠️ Important Configuration Notes: Credentials, folders, testing, security                                    |
| Sticky Note8                | stickyNote                         | Customization and extension tips  | None                        | None                          | ## 💡 Customization Tips: Adjust classification, personalize replies, extend functionality                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Microsoft Outlook Trigger Node:**  
   - Type: Microsoft Outlook Trigger  
   - Configure: Poll every minute, no filters, enable attachment download  
   - Connect output to AI classification nodes

2. **Create OpenAI Chat Model - Classifier Node:**  
   - Type: GPT-4 mini (lmChatOpenAi)  
   - No extra options needed  
   - Connect input from Outlook Trigger  
   - Connect output to AI Email Classifier node  
   - Set OpenAI credentials in n8n credentials manager

3. **Create AI Email Classifier Node:**  
   - Type: Langchain Text Classifier  
   - Configure system prompt with detailed category definitions ("Urgent," "Important," "Not Important") including today's date variable (`{{ $today }}`)  
   - Input text from Outlook Trigger email body preview  
   - Output connections: three separate outputs corresponding to each category (Urgent, Important, Not Important)

4. **For Each Email Category:**

   - **Urgent Path:**  
     a. Create OpenAI Chat Model - Urgent Node (GPT-4 mini)  
     b. Create Generate Urgent Reply Node (Langchain Agent) with prompt instructing brief, formal, urgent reply, referencing email body preview.  
     c. Connect OpenAI Chat Model - Urgent to Generate Urgent Reply (set as AI language model provider).  
     d. Create Reply to Urgent Email Node (Microsoft Outlook) configured to:  
        - Reply to original email ID  
        - Message content from Generate Urgent Reply  
        - ReplyTo sender only  
     e. Create Move to Urgent Folder Node (Microsoft Outlook) configured with folder ID for “URGENT” folder  
     f. Connect nodes in order: AI Email Classifier (Urgent) → Generate Urgent Reply → Reply to Urgent Email → Move to Urgent Folder

   - **Important Path:**  
     a. Create OpenAI Chat Model - Important Node (GPT-4 mini)  
     b. Create Generate Important Reply Node (Langchain Agent) with prompt for balanced, professional reply without urgency implication  
     c. Connect OpenAI Chat Model - Important to Generate Important Reply  
     d. Create Reply to Important Email Node (Microsoft Outlook)  
     e. Create Move to Important Folder Node with folder ID for “Important” folder  
     f. Connect: AI Email Classifier (Important) → Generate Important Reply → Reply to Important Email → Move to Important Folder

   - **Not Important Path:**  
     a. Create OpenAI Chat Model - Standard Node (GPT-4 mini)  
     b. Create Generate Standard Reply Node (Langchain Agent) with prompt for polite, brief reply indicating processing “in due time”  
     c. Connect OpenAI Chat Model - Standard to Generate Standard Reply  
     d. Create Reply to Not Important Email Node (Microsoft Outlook)  
     e. Create Move to Not Important Folder Node with folder ID for “Not Important” folder  
     f. Connect: AI Email Classifier (Not Important) → Generate Standard Reply → Reply to Not Important Email → Move to Not Important Folder

5. **Credential Configuration:**  
   - Set up Microsoft Outlook OAuth2 credentials with API access  
   - Configure OpenAI API key with GPT-4 access in n8n credential system  
   - Ensure all Chat Model and Outlook nodes reference these credentials

6. **Folder Setup:**  
   - Create three folders in Outlook named exactly: “URGENT”, “Important”, and “Not Important”  
   - Obtain folder IDs for each and configure Move nodes accordingly

7. **Testing and Adjustments:**  
   - Use test emails to verify correct classification, reply generation, and folder movement  
   - Adjust AI prompts in Generate Reply nodes to customize tone and style as needed  
   - Monitor logs for any API or connection errors and handle accordingly

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Workflow designed for professionals needing automatic email triage and reply generation using Outlook and GPT-4 | Overview provided in initial sticky note in workflow                                                                 |
| Requires Microsoft Outlook account with API access and OpenAI API key with GPT-4 access                          | Configuration notes in sticky notes and setup instructions                                                          |
| Create three folders in Outlook before running: URGENT, Important, Not Important                                 | Folder creation required for correct email routing                                                                   |
| Customize AI prompts in Langchain Agent nodes to adjust reply tone and style                                    | Sticky Note8 customization tips                                                                                      |
| Avoid hardcoding API keys; use n8n credential system for security                                               | Security best practices emphasized in sticky note7                                                                   |
| Potential extensions include Slack notifications, CRM integration, and reporting dashboards                     | Customization and extension tips in sticky note8                                                                     |
| Video or blog resources for AI email automation workflows can be found on n8n community and official documentation | External resources not included here but recommended for extended learning                                            |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. The processing complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.