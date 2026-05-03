Smart Email Classifier & Auto-Responder with AI

https://n8nworkflows.xyz/workflows/smart-email-classifier---auto-responder-with-ai-3242


# Smart Email Classifier & Auto-Responder with AI

### 1. Workflow Overview

This workflow automates the management of incoming emails by leveraging AI to classify messages, generate context-aware replies, and trigger notifications and calendar events. It is designed for professionals, customer support, and sales teams who want to streamline email handling and improve response efficiency.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Preprocessing**: Captures incoming emails via Gmail trigger and prepares data for AI processing.
- **1.2 AI-Powered Email Classification**: Uses Azure OpenAI and LangChain sentiment analysis nodes to categorize emails into Spam, Important, Promotion, Notification, Personal, Call Request, and Needs Reply.
- **1.3 Decision Logic & Filtering**: Determines if emails are important or require replies, routing them accordingly.
- **1.4 AI-Generated Draft Replies**: Generates smart, context-aware email drafts using AI agents.
- **1.5 Email Drafting & Sending**: Creates Gmail drafts and sends Telegram notifications for important emails.
- **1.6 Google Calendar Integration**: Optionally schedules follow-ups based on email content.
- **1.7 No-Operation Handling**: Gracefully handles emails that do not require action.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preprocessing

- **Overview:**  
  This block captures incoming emails via Gmail trigger and sets up the initial data structure for further processing.

- **Nodes Involved:**  
  - Gmail Trigger  
  - $INPUTS$

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Role: Listens for new incoming emails in Gmail inbox  
    - Configuration: Uses OAuth2 credentials for Gmail; triggers on new emails  
    - Inputs: None (trigger)  
    - Outputs: Email data to $INPUTS$ node  
    - Edge Cases: Gmail API rate limits, OAuth token expiration, webhook failures  

  - **$INPUTS$**  
    - Type: Set node  
    - Role: Prepares and normalizes incoming email data for AI processing  
    - Configuration: No parameters set, acts as a pass-through or placeholder for data shaping  
    - Inputs: Gmail Trigger output  
    - Outputs: Data forwarded to Sentiment Analysis node  
    - Edge Cases: Empty or malformed email data  

#### 1.2 AI-Powered Email Classification

- **Overview:**  
  Classifies incoming emails into predefined categories using Azure OpenAI chat model and LangChain sentiment analysis nodes.

- **Nodes Involved:**  
  - Azure OpenAI Chat Model  
  - Sentiment Analysis  
  - Spam  
  - Important  
  - Promotion  
  - Notification  
  - Personal  
  - Call request  
  - Needs Reply

- **Node Details:**

  - **Azure OpenAI Chat Model**  
    - Type: AI language model node (Azure OpenAI GPT)  
    - Role: Processes email content to assist classification and response generation  
    - Configuration: Connected to Azure OpenAI with GPT-4o model; used as a language model backend for classification and reply drafting  
    - Inputs: None directly; used as a resource by other AI nodes  
    - Outputs: Feeds into Sentiment Analysis, important?, Draft Reply, needs reply? nodes  
    - Edge Cases: API quota limits, network timeouts, invalid prompts  

  - **Sentiment Analysis**  
    - Type: LangChain sentiment analysis node  
    - Role: Analyzes email sentiment and content to classify into categories  
    - Configuration: Uses AI model to detect email category  
    - Inputs: $INPUTS$ node output (email data)  
    - Outputs: Routes emails to category-specific Gmail nodes (Spam, Important, Promotion, Notification, Personal, Call request, Needs Reply)  
    - Edge Cases: Ambiguous content, misclassification, API errors  

  - **Spam, Important, Promotion, Notification, Personal, Call request, Needs Reply**  
    - Type: Gmail nodes (webhook triggers)  
    - Role: Act as category-specific entry points for emails classified into respective categories  
    - Configuration: Each node listens for emails tagged with its category via webhookId  
    - Inputs: Sentiment Analysis output  
    - Outputs: Connected to further processing nodes (e.g., important? or Draft Reply)  
    - Edge Cases: Webhook failures, missing emails, duplicate triggers  

#### 1.3 Decision Logic & Filtering

- **Overview:**  
  Determines if emails are important or require replies, guiding the workflow to generate responses or ignore.

- **Nodes Involved:**  
  - important? (Sentiment Analysis)  
  - needs reply? (Sentiment Analysis)  
  - No Operation, do nothing

- **Node Details:**

  - **important?**  
    - Type: Sentiment Analysis node  
    - Role: Checks if an email is important enough to warrant notification or further action  
    - Inputs: Promotion, Notification nodes output  
    - Outputs: Routes to Gmail and Important nodes for further processing  
    - Edge Cases: False positives/negatives in importance detection  

  - **needs reply?**  
    - Type: Sentiment Analysis node  
    - Role: Determines if an email requires a reply  
    - Inputs: Important node output  
    - Outputs: Routes to Draft Reply node if reply needed, or No Operation node if not  
    - Edge Cases: Misjudging reply necessity, AI errors  

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Role: Terminates processing for emails that do not require replies  
    - Inputs: needs reply? output  
    - Outputs: None  
    - Edge Cases: None (safe termination)  

#### 1.4 AI-Generated Draft Replies

- **Overview:**  
  Generates context-aware draft replies using AI agents, leveraging structured output parsing for consistency.

- **Nodes Involved:**  
  - Draft Reply (LangChain agent)  
  - json schema (Output Parser Structured)

- **Node Details:**

  - **Draft Reply**  
    - Type: LangChain agent node  
    - Role: Generates AI-powered email reply drafts based on email content and classification  
    - Configuration: Uses Azure OpenAI Chat Model as language model backend; connected to Gmail Draft and Telegram nodes downstream  
    - Inputs: From Personal, Needs Reply, Call request, Draft, Google Calendar, Get emails nodes  
    - Outputs: To Write draft node  
    - Edge Cases: AI generation errors, incomplete replies, API limits  

  - **json schema**  
    - Type: Output Parser Structured node  
    - Role: Parses AI output into structured JSON format for consistent reply generation  
    - Inputs: Connected to Draft Reply ai_outputParser input  
    - Outputs: Feeds parsed data back to Draft Reply for final processing  
    - Edge Cases: Parsing failures, malformed AI output  

#### 1.5 Email Drafting & Sending

- **Overview:**  
  Creates Gmail drafts for generated replies and sends Telegram notifications for important emails.

- **Nodes Involved:**  
  - Write draft (Gmail)  
  - Telegram

- **Node Details:**

  - **Write draft**  
    - Type: Gmail node  
    - Role: Creates draft emails in Gmail based on AI-generated replies  
    - Configuration: Uses Gmail OAuth2 credentials; drafts are saved but not sent automatically  
    - Inputs: Draft Reply output  
    - Outputs: Telegram node for notifications  
    - Edge Cases: Gmail API errors, draft creation failures  

  - **Telegram**  
    - Type: Telegram node  
    - Role: Sends real-time alerts for important emails to configured Telegram chat  
    - Configuration: Uses Telegram Bot credentials; sends message with email summary or alert  
    - Inputs: Write draft output  
    - Outputs: None  
    - Edge Cases: Telegram API rate limits, bot permission errors  

#### 1.6 Google Calendar Integration

- **Overview:**  
  Optionally schedules calendar events or follow-ups based on email content and AI analysis.

- **Nodes Involved:**  
  - Google Calendar  
  - Draft Reply (ai_tool input)

- **Node Details:**

  - **Google Calendar**  
    - Type: Google Calendar Tool node  
    - Role: Creates calendar events or reminders for follow-ups extracted from email content  
    - Configuration: Uses Google Calendar OAuth2 credentials; event details generated by AI  
    - Inputs: Draft Reply ai_tool input  
    - Outputs: Draft Reply node for further processing  
    - Edge Cases: Calendar API errors, invalid event data  

#### 1.7 No-Operation Handling

- **Overview:**  
  Handles emails that do not require any action, ensuring workflow stability.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Role: Terminates processing for emails that do not require replies or actions  
    - Inputs: needs reply? output (when no reply needed)  
    - Outputs: None  
    - Edge Cases: None  

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                          | Input Node(s)                  | Output Node(s)                     | Sticky Note |
|---------------------|----------------------------------|----------------------------------------|-------------------------------|----------------------------------|-------------|
| Gmail Trigger       | Gmail Trigger                    | Captures incoming emails                | None                          | $INPUTS$                         |             |
| $INPUTS$            | Set                             | Prepares email data                     | Gmail Trigger                 | Sentiment Analysis               |             |
| Azure OpenAI Chat Model | LangChain LM Chat Azure OpenAI | AI language model backend               | None                         | Sentiment Analysis, important?, Draft Reply, needs reply? |             |
| Sentiment Analysis  | LangChain Sentiment Analysis     | Classifies emails into categories       | $INPUTS$                     | Spam, Important, Promotion, Notification, Personal, Call request, Needs Reply |             |
| Spam                | Gmail                           | Handles Spam category emails             | Sentiment Analysis           | Gmail                           |             |
| Important           | Gmail                           | Handles Important category emails        | Sentiment Analysis           | needs reply?                    |             |
| Promotion           | Gmail                           | Handles Promotion category emails        | Sentiment Analysis           | important?                     |             |
| Notification        | Gmail                           | Handles Notification category emails     | Sentiment Analysis           | important?                     |             |
| Personal            | Gmail                           | Handles Personal category emails         | Sentiment Analysis           | Draft Reply                    |             |
| Call request        | Gmail                           | Handles Call Request category emails     | Sentiment Analysis           | Draft Reply                    |             |
| Needs Reply         | Gmail                           | Handles Needs Reply category emails      | Sentiment Analysis           | Draft Reply                    |             |
| important?          | LangChain Sentiment Analysis     | Determines if email is important         | Promotion, Notification      | Gmail, Important               |             |
| needs reply?        | LangChain Sentiment Analysis     | Determines if email needs reply          | Important                   | Draft Reply, No Operation       |             |
| Draft Reply         | LangChain Agent                 | Generates AI draft replies                | Personal, Needs Reply, Call request, Draft, Google Calendar, Get emails | Write draft                    |             |
| json schema         | LangChain Output Parser Structured | Parses AI output into structured format | Draft Reply (ai_outputParser) | Draft Reply                    |             |
| Write draft         | Gmail                           | Creates Gmail draft emails                | Draft Reply                  | Telegram                      |             |
| Telegram            | Telegram                        | Sends Telegram notifications              | Write draft                  | None                         |             |
| Google Calendar     | Google Calendar Tool            | Schedules calendar events                  | Draft Reply (ai_tool)        | Draft Reply                   |             |
| No Operation, do nothing | NoOp                         | Terminates processing for no-action emails | needs reply?                | None                         |             |
| Get emails          | Gmail Tool                     | Retrieves emails (used for AI reply generation) | None                      | Draft Reply (ai_tool)          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Credentials: Connect Gmail OAuth2 credentials  
   - Trigger on new incoming emails  
   - Position: Start of workflow  

2. **Add Set Node named `$INPUTS$`**  
   - Purpose: Prepare incoming email data for AI processing  
   - Connect Gmail Trigger output to this node  

3. **Add Azure OpenAI Chat Model Node**  
   - Type: LangChain LM Chat Azure OpenAI  
   - Credentials: Connect Azure OpenAI with GPT-4o model  
   - No direct input; acts as AI backend for classification and reply generation  

4. **Add Sentiment Analysis Node**  
   - Type: LangChain Sentiment Analysis  
   - Connect `$INPUTS$` output to this node  
   - Configure to classify emails into categories: Spam, Important, Promotion, Notification, Personal, Call Request, Needs Reply  
   - Connect Azure OpenAI Chat Model as language model backend  

5. **Add Gmail Nodes for Each Category**  
   - Spam, Important, Promotion, Notification, Personal, Call request, Needs Reply  
   - Each configured with Gmail OAuth2 credentials  
   - Connect Sentiment Analysis outputs to respective category nodes  

6. **Add important? Sentiment Analysis Node**  
   - Connect Promotion and Notification nodes to this node  
   - Configure to determine if email is important  
   - Output to Gmail and Important nodes  

7. **Add needs reply? Sentiment Analysis Node**  
   - Connect Important node output to this node  
   - Configure to determine if email requires reply  
   - Output to Draft Reply node if yes, No Operation node if no  

8. **Add No Operation Node**  
   - Connect needs reply? output for no-reply emails  
   - Purpose: End processing for these emails  

9. **Add Draft Reply Node (LangChain Agent)**  
   - Connect Personal, Needs Reply, Call request, Draft, Google Calendar, and Get emails nodes to this node  
   - Use Azure OpenAI Chat Model as backend  
   - Configure for context-aware reply generation  

10. **Add json schema Node (Output Parser Structured)**  
    - Connect Draft Reply ai_outputParser input to this node  
    - Purpose: Parse AI output into structured JSON for consistent replies  

11. **Add Write draft Gmail Node**  
    - Connect Draft Reply output to this node  
    - Configure to create Gmail drafts (OAuth2 credentials required)  

12. **Add Telegram Node**  
    - Connect Write draft output to this node  
    - Configure Telegram Bot credentials for sending notifications  

13. **Add Google Calendar Node**  
    - Connect Draft Reply ai_tool input to this node  
    - Configure Google Calendar OAuth2 credentials  
    - Purpose: Schedule follow-ups or events extracted from emails  

14. **Add Get emails Gmail Tool Node**  
    - Optional: Used for fetching emails for AI reply generation  
    - Connect output to Draft Reply ai_tool input  

15. **Connect all nodes according to logical flow:**  
    - Gmail Trigger → $INPUTS$ → Sentiment Analysis → Category Gmail nodes → important? / needs reply? → Draft Reply → Write draft → Telegram  
    - Google Calendar connected to Draft Reply for scheduling  
    - No Operation node connected for emails not needing replies  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Setup Instructions: Connect Gmail, OpenAI, Telegram, and Google Calendar credentials.         | Workflow description section                                                                    |
| AI model used: Azure OpenAI GPT-4o for classification and reply generation.                    | Workflow overview                                                                               |
| Supports email threading and context-aware replies for seamless conversations.                 | Features section                                                                               |
| Real-time Telegram alerts notify users of important emails.                                   | Features section                                                                               |
| Google Calendar integration schedules follow-ups based on email content.                      | Features section                                                                               |
| Recommended for busy professionals, customer support, and sales teams to manage inboxes.      | Workflow description                                                                           |
| For detailed node configuration, ensure OAuth2 credentials are valid and API quotas are sufficient. | General operational advice                                                                     |

---

This document provides a comprehensive understanding of the "Smart Email Classifier & Auto-Responder with AI" workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.