Automated Inbox Organization for Outlook using GPT-4.1 Agent

https://n8nworkflows.xyz/workflows/automated-inbox-organization-for-outlook-using-gpt-4-1-agent-6454


# Automated Inbox Organization for Outlook using GPT-4.1 Agent

### 1. Workflow Overview

This workflow automates the organization of incoming Outlook emails by leveraging GPT-4.1 AI to categorize messages intelligently. Its primary purpose is to maintain a clean Inbox by moving only non-essential emails to appropriate folders while preserving actionable client communications and ongoing conversations in place.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Extraction:** Poll Outlook Inbox every minute to detect new emails and extract key fields.
- **1.2 Email Content Preparation:** Convert email body from HTML to Markdown and sanitize the content for AI processing.
- **1.3 AI-Based Categorization:** Use a LangChain AI Agent powered by GPT-4.1 to analyze the email and determine the best folder category based on strict rules.
- **1.4 Outlook Folder and Contact Retrieval:** Fetch all Outlook folders and contacts to provide the AI with contextual data and support decision-making.
- **1.5 Message Movement:** Move the email to the selected folder if applicable, or leave it in place with an explanation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Extraction

- **Overview:**  
  This block triggers the workflow whenever a new email arrives in a specified Outlook folder (Inbox). It extracts relevant fields for downstream processing.

- **Nodes Involved:**  
  - Microsoft Outlook Trigger

- **Node Details:**  
  - **Microsoft Outlook Trigger**  
    - Type: Trigger node for Outlook  
    - Configuration:  
      - Polls every minute for new emails.  
      - Extracted fields: `from`, `subject`, `isRead`, `body`.  
      - Monitors a specific folder ID (Inbox).  
    - Inputs: None (trigger)  
    - Outputs: JSON containing email metadata and body content.  
    - Potential Failures: Authentication errors with Outlook OAuth2, folder ID misconfiguration, API rate limits.  
    - Notes: Requires valid Microsoft Outlook OAuth2 credentials.

#### 2.2 Email Content Preparation

- **Overview:**  
  Converts the HTML email body into Markdown format and sanitizes the content by removing images, links, HTML tags, and table formatting. The sanitized plain text is truncated to 4000 characters for optimal AI input.

- **Nodes Involved:**  
  - Markdown  
  - Sanitize Email Body (Code Node)

- **Node Details:**  
  - **Markdown**  
    - Type: Conversion node  
    - Configuration: Converts `body.content` (HTML) to Markdown, output key named `Email Body Markdown`.  
    - Inputs: Output of Outlook Trigger node (email body).  
    - Outputs: Markdown version of the email body.  
    - Failures: Invalid HTML input or empty email body may cause conversion issues.  
  - **Sanitize Email Body**  
    - Type: Code node  
    - Configuration: JavaScript code that:  
      - Removes Markdown images completely.  
      - Simplifies or removes Markdown links.  
      - Strips residual HTML tags.  
      - Removes markdown table formatting.  
      - Normalizes whitespace and truncates to 4000 characters.  
    - Inputs: Markdown content from the previous node.  
    - Outputs: `sanitizedBody` containing clean plain text for AI.  
    - Failures: Potential for unexpected text structure causing regex failures, large emails truncated.

#### 2.3 AI-Based Categorization

- **Overview:**  
  Uses a GPT-4.1 LangChain AI Agent to analyze the sanitized email content and metadata. The agent decides whether to move the message and to which folder based on detailed instructions and contextual data.

- **Nodes Involved:**  
  - AI Agent - Determine Category  
  - OpenRouter Chat Model (GPT-4.1)

- **Node Details:**  
  - **AI Agent - Determine Category**  
    - Type: LangChain agent node  
    - Configuration:  
      - System message defines assistant role: managing Inbox to keep only actionable items.  
      - Input prompt includes email subject, sender email and name, sanitized body, email ID.  
      - Instructions:  
        - Determine if email requires action (client message, inquiry) → move to ‘Action’ folder.  
        - Otherwise, categorize into Junk, Receipt, SaaS, etc., or ‘Other’.  
        - Do not move emails from saved contacts or real people (recognized by presence of personal name/email).  
        - Explain brief reasons if no move is done.  
        - Use Outlook folder IDs only for moves.  
      - Uses external tools: Get Folders, Get Contacts, Move Message.  
      - Output parser enabled for structured response.  
    - Inputs: Sanitized email data, folders, contacts, AI language model.  
    - Outputs: Decision on folder ID and message ID, or explanation.  
    - Failures: AI model latency or downtime, parsing errors, incomplete data, misclassification.  
  - **OpenRouter Chat Model**  
    - Type: Large Language Model node (OpenRouter GPT-4.1)  
    - Configuration: Uses GPT-4.1 model via OpenRouter API.  
    - Credentials: OpenRouter API key.  
    - Inputs: From AI Agent node as language model backend.  
    - Outputs: AI-generated response for categorization.  
    - Failures: API key invalid/expired, rate limits, network issues.

#### 2.4 Outlook Folder and Contact Retrieval

- **Overview:**  
  Retrieves all available Outlook folders and saved contacts to provide the AI with current folder options and identify contacts to avoid moving their emails.

- **Nodes Involved:**  
  - Get Folders  
  - Get Contacts

- **Node Details:**  
  - **Get Folders**  
    - Type: Microsoft Outlook Tool node  
    - Configuration: Retrieves all folders (Inbox, Junk, Action, etc.) with their IDs.  
    - Inputs: None (invoked by AI Agent as tool).  
    - Outputs: Folder list JSON.  
    - Failures: API errors, permission issues.  
  - **Get Contacts**  
    - Type: Microsoft Outlook Tool node  
    - Configuration: Retrieves all saved contacts.  
    - Inputs: None (invoked by AI Agent as tool).  
    - Outputs: Contacts list JSON.  
    - Failures: API errors, permission issues.

#### 2.5 Message Movement

- **Overview:**  
  Moves the email message to a designated folder based on AI decision or leaves it in place with explanation for no move.

- **Nodes Involved:**  
  - Move Message

- **Node Details:**  
  - **Move Message**  
    - Type: Microsoft Outlook Tool node  
    - Configuration:  
      - Operation: Move message.  
      - Inputs: Folder ID and message ID received dynamically from AI Agent output.  
      - Uses Outlook OAuth2 credentials.  
    - Inputs: Folder ID and message ID (from AI Agent).  
    - Outputs: Confirmation of move or failure.  
    - Failures: Invalid folder/message ID, permissions, API errors, concurrency issues.

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                         | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                   |
|-------------------------|--------------------------------------------|---------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Microsoft Outlook Trigger| n8n-nodes-base.microsoftOutlookTrigger     | Trigger on new Outlook email arrival  | None                        | Markdown                    |                                                                                                                               |
| Markdown                | n8n-nodes-base.markdown                     | Convert HTML email body to Markdown   | Microsoft Outlook Trigger   | Sanitize Email Body         |                                                                                                                               |
| Sanitize Email Body     | n8n-nodes-base.code                         | Clean and truncate email body text    | Markdown                    | AI Agent - Determine Category|                                                                                                                               |
| AI Agent - Determine Category | @n8n/n8n-nodes-langchain.agent         | Categorize emails with GPT-4.1 agent  | Sanitize Email Body, Get Folders, Get Contacts, Move Message, OpenRouter Chat Model | None                        |                                                                                                                               |
| OpenRouter Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenRouter  | LLM backend for AI agent (GPT-4.1)    | AI Agent - Determine Category| AI Agent - Determine Category|                                                                                                                               |
| Get Folders             | n8n-nodes-base.microsoftOutlookTool         | Retrieve all Outlook folders           | AI Agent - Determine Category | AI Agent - Determine Category|                                                                                                                               |
| Get Contacts            | n8n-nodes-base.microsoftOutlookTool         | Retrieve saved Outlook contacts        | AI Agent - Determine Category | AI Agent - Determine Category|                                                                                                                               |
| Move Message            | n8n-nodes-base.microsoftOutlookTool         | Move email to selected folder          | AI Agent - Determine Category | AI Agent - Determine Category|                                                                                                                               |
| Sticky Note             | n8n-nodes-base.stickyNote                    | Documentation and explanation          | None                        | None                        | # Auto-Categorize Outlook Emails with AI in n8n  \n\n## **How It Works**  \n\n1. Workflow triggers every minute for new emails.  \n2. Converts and sanitizes email body for AI analysis.  \n3. AI agent decides email category and moves messages accordingly.  \n\n## **Best Practices & Notes**  \n- AI is conservative: never moves emails from real contacts.  \n- Designed to maintain Inbox Zero.  \n- Customizable folder and prompt logic.  \n- Privacy ensured; minimal sanitized data sent to AI. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Microsoft Outlook Trigger Node**  
   - Type: Microsoft Outlook Trigger  
   - Set resource to Message, operation to trigger on new email.  
   - Configure to extract fields: `from`, `subject`, `isRead`, `body`.  
   - Set folder filter to your Inbox folder ID.  
   - Poll interval: every 1 minute.  
   - Connect Microsoft Outlook OAuth2 credentials.  

2. **Add Markdown Node**  
   - Type: Markdown  
   - Input HTML: `{{$json["body"]["content"]}}` from Outlook Trigger node.  
   - Output key: `Email Body Markdown`.  
   - Connect output of Outlook Trigger to this node.  

3. **Add Code Node to Sanitize Email Body**  
   - Type: Code  
   - Paste JavaScript code that removes images, links, HTML tags, tables, and truncates to 4000 characters.  
   - Input: `Email Body Markdown` from Markdown node.  
   - Output: `sanitizedBody`.  
   - Connect Markdown node output to this node.  

4. **Add OpenRouter Chat Model Node**  
   - Type: LangChain LLM Chat OpenRouter  
   - Model: `openai/gpt-4.1`  
   - Connect your OpenRouter API credentials.  

5. **Add AI Agent - Determine Category Node**  
   - Type: LangChain AI Agent  
   - Configure prompt with instructions to analyze email fields (`subject`, `from.emailAddress.address`, `from.emailAddress.name`), and sanitized body (`sanitizedBody`).  
   - System message: Define role as Inbox manager prioritizing actionable emails.  
   - Enable output parser for structured responses.  
   - Set tools: link to Get Folders, Get Contacts, Move Message nodes.  
   - Connect input from Sanitize Email Body node.  
   - Use OpenRouter Chat Model node as language model backend.  

6. **Add Microsoft Outlook Tool Node - Get Folders**  
   - Type: Microsoft Outlook Tool  
   - Resource: Folder  
   - Operation: Get All  
   - Return All: true  
   - Connect to AI Agent as a tool node.  
   - Use Microsoft Outlook OAuth2 credentials.  

7. **Add Microsoft Outlook Tool Node - Get Contacts**  
   - Type: Microsoft Outlook Tool  
   - Resource: Contact  
   - Operation: Get All  
   - Return All: true  
   - Connect to AI Agent as a tool node.  
   - Use Microsoft Outlook OAuth2 credentials.  

8. **Add Microsoft Outlook Tool Node - Move Message**  
   - Type: Microsoft Outlook Tool  
   - Operation: Move  
   - Inputs for folderId and messageId set dynamically from AI Agent outputs via expressions.  
   - Connect to AI Agent as a tool node.  
   - Use Microsoft Outlook OAuth2 credentials.  

9. **Connect Workflow**  
   - Microsoft Outlook Trigger → Markdown → Sanitize Email Body → AI Agent - Determine Category  
   - AI Agent uses OpenRouter Chat Model as LLM backend.  
   - AI Agent uses Get Folders, Get Contacts, and Move Message as tools.  

10. **Add Sticky Note for Documentation (Optional)**  
    - Add a Sticky Note node with detailed workflow description and best practices as provided in the original workflow.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The AI agent is explicitly instructed to never move emails from known personal contacts or real people, ensuring important communications remain in the Inbox. This conservative approach helps maintain actionable focus without losing emails that require attention.                                                                                                                                                                                                                                                                                                          | Workflow best practice and prompt engineering.                                                 |
| The workflow is designed to maintain Inbox Zero by sorting non-essential emails into appropriate folders automatically, reducing manual email triage effort.                                                                                                                                                                                                                                                                                                                                                                                                              | Workflow purpose and design philosophy.                                                       |
| All email data processing happens within the n8n environment, with minimal and sanitized content sent to the AI service (OpenRouter GPT-4.1). This approach protects privacy and data security.                                                                                                                                                                                                                                                                                                                                                                              | Privacy and security notes.                                                                    |
| The AI prompt uses an advanced LangChain agent with tool integration, allowing dynamic retrieval of folders and contacts, and executing move operations securely. This modular design enables easy customization or extension to other email providers or categories.                                                                                                                                                                                                                                                                                                       | Architecture and extensibility notes.                                                         |
| For further customization, users can adjust folder IDs, add more categories, or refine AI instructions to better match their email handling policies.                                                                                                                                                                                                                                                                                                                                                                                                                      | Customization advice.                                                                          |
| The workflow uses Microsoft Outlook OAuth2 credentials, which must be correctly configured with appropriate permissions for reading emails, folders, contacts, and moving messages.                                                                                                                                                                                                                                                                                                                                                                                         | Credential setup requirement.                                                                 |
| OpenRouter API key is required to access GPT-4.1 model; ensure the key is valid and has sufficient quota to handle the workflow frequency.                                                                                                                                                                                                                                                                                                                                                                                                                                 | Credential setup and usage note.                                                              |
| Sticky note node in the workflow contains detailed documentation and best practices which is recommended to keep for operational clarity.                                                                                                                                                                                                                                                                                                                                                                                                                                   | Internal documentation best practice.                                                        |

---

**Disclaimer:** The provided description and analysis are derived exclusively from the n8n workflow automation JSON. This processing respects all relevant content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.