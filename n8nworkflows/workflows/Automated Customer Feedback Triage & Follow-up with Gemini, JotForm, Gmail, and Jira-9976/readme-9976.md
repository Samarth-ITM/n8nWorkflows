Automated Customer Feedback Triage & Follow-up with Gemini, JotForm, Gmail, and Jira

https://n8nworkflows.xyz/workflows/automated-customer-feedback-triage---follow-up-with-gemini--jotform--gmail--and-jira-9976


# Automated Customer Feedback Triage & Follow-up with Gemini, JotForm, Gmail, and Jira

### 1. Workflow Overview

This workflow automates customer feedback triage and follow-up by integrating JotForm (for receiving feedback), AI-powered analysis and response generation (via Langchain agents using Google Gemini chat model), Gmail (for email communication), and Jira (for issue tracking). It targets businesses seeking to efficiently handle customer feedback by automatically categorizing feedback as positive or negative, generating appropriate email replies, and creating Jira issues for negative feedback to facilitate resolution.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Captures feedback submissions from customers via JotForm and incoming emails from Gmail.
- **1.2 AI Feedback Analysis & Response Generation:** Uses AI agents to classify feedback, generate email replies, and determine subsequent actions.
- **1.3 Email Response Handling:** Sends automated email replies to customers based on AI-generated messages and manages email threads.
- **1.4 Issue Tracking:** Creates Jira issues automatically for negative feedback to ensure tracking and resolution.
- **1.5 Follow-up Interaction:** Engages customers further via email to clarify issues and collect more information when negative feedback is detected.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block receives customer feedback submitted through JotForm and also listens to unread incoming emails in Gmail to trigger further processing.

**Nodes Involved:**  
- JotForm Trigger  
- Gmail Trigger

**Node Details:**

- **JotForm Trigger**  
  - *Type:* Trigger  
  - *Role:* Captures new form submissions from a specific JotForm form (ID: 252856264643060).  
  - *Configuration:* Webhook-based trigger linked to the form; `resolveData` is false (raw form data).  
  - *Input:* Incoming form submission webhook.  
  - *Output:* JSON data containing user feedback fields, including the user's name and feedback text.  
  - *Edge Cases:* Possible webhook setup failure, form field changes could break expressions.  
  - *Sticky Note:* "Get the feedback from users" with a JotForm signup link.

- **Gmail Trigger**  
  - *Type:* Trigger  
  - *Role:* Watches for unread emails in Gmail inbox to handle ongoing email conversations.  
  - *Configuration:* Polls every minute for unread messages; no simple mode to retain full metadata.  
  - *Input:* Incoming emails.  
  - *Output:* Email details including thread ID and message content.  
  - *Edge Cases:* Gmail API rate limits, authentication errors, or message format changes.

---

#### 2.2 AI Feedback Analysis & Response Generation

**Overview:**  
This block uses AI agents powered by Langchain and Google Gemini to analyze feedback, classify it as positive or negative, and generate appropriate response messages and action instructions.

**Nodes Involved:**  
- AI Agent (for JotForm feedback)  
- Google Gemini Chat Model (language model for AI Agents)  
- Structured Output Parser  
- Edit Fields

**Node Details:**

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Processes JotForm customer feedback; determines if feedback is positive or negative.  
  - *Configuration:* Prompt instructs the agent to write a concise reply mentioning the user’s first name and feedback. If positive, express appreciation and no Jira issue; if negative, acknowledge feedback and instruct Jira issue creation.  
  - *Key Expressions:* Uses expressions like `{{ $json.q3_name.first }}` (user's first name) and `{{ $json.q6_typeA6 }}` (feedback text).  
  - *Input:* JotForm Trigger data.  
  - *Output:* Structured data including reply message, Gmail thread ID, and feedback classification.  
  - *Edge Cases:* Failure to parse user name or feedback text, AI misclassification, or API errors.  
  - *Connected To:* Edit Fields node.

- **Google Gemini Chat Model**  
  - *Type:* Language Model  
  - *Role:* Provides the AI language understanding and generation capabilities for agents.  
  - *Configuration:* Default settings, no custom options.  
  - *Input:* From AI Agents.  
  - *Output:* Text completion for AI Agents.  
  - *Edge Cases:* API limits, latency, or unavailability.

- **Structured Output Parser**  
  - *Type:* Langchain Output Parser  
  - *Role:* Parses AI agent output to structured JSON format for downstream processing.  
  - *Configuration:* Example JSON schema specifies expected fields like `threadId` and `output`.  
  - *Input:* AI Agent raw output.  
  - *Output:* Structured JSON with key reply components.  
  - *Edge Cases:* Parsing errors if AI output deviates from expected format.

- **Edit Fields**  
  - *Type:* Set Node  
  - *Role:* Extracts and reshapes data, creating variables `text` (reply message) and `threadId` for further nodes.  
  - *Configuration:* Sets `text` to the feedback text from JotForm and `threadId` to the AI agent’s output thread ID.  
  - *Input:* AI Agent output.  
  - *Output:* Processed JSON for subsequent AI Agent (Chat).  
  - *Edge Cases:* Missing or null fields from prior nodes.

---

#### 2.3 Email Response Handling

**Overview:**  
This block sends email replies to customers, either as initial thank-you messages or as follow-up questions, depending on feedback sentiment.

**Nodes Involved:**  
- Send a message in Gmail  
- Reply to a message in Gmail

**Node Details:**

- **Send a message in Gmail**  
  - *Type:* Gmail Tool (Send)  
  - *Role:* Sends an initial thank-you email to the customer using the email address from JotForm submission.  
  - *Configuration:* Sends to `q4_email` field from JotForm; message content generated by AI Agent; subject fixed as "Thank you for your Response".  
  - *Input:* AI Agent output with message text.  
  - *Output:* Email sent confirmation, includes webhook ID for response tracking.  
  - *Edge Cases:* Email delivery failure, invalid email addresses, Gmail API quota limits.

- **Reply to a message in Gmail**  
  - *Type:* Gmail Tool (Reply)  
  - *Role:* Replies to existing Gmail threads with AI-generated messages, used during follow-up question phase.  
  - *Configuration:* Replies to message ID from Gmail Trigger; message text from AI Agent (Chat); text email type; no attribution appended.  
  - *Input:* AI Agent (Chat) output and Gmail Trigger data.  
  - *Output:* Email reply sent confirmation.  
  - *Edge Cases:* Message ID not found, thread closed, or API errors.

---

#### 2.4 Issue Tracking

**Overview:**  
Automatically creates Jira issues for negative feedback to log problems and trigger resolution processes.

**Nodes Involved:**  
- Create an issue in Jira Software

**Node Details:**

- **Create an issue in Jira Software**  
  - *Type:* Jira Tool  
  - *Role:* Creates a new Jira issue with summary and priority generated by AI agent for negative feedback.  
  - *Configuration:* Project and issue type selected from a list (configuration placeholders present, values expected to be set in credentials or environment); summary and priority dynamically generated by AI output.  
  - *Input:* AI Agent output with issue details.  
  - *Output:* Jira issue creation confirmation.  
  - *Edge Cases:* Authentication errors, missing project or issue type configuration, API limits, malformed summaries.

---

#### 2.5 Follow-up Interaction

**Overview:**  
Engages customers who provide negative feedback by asking clarifying questions through email, collecting more details about the issue, and assuring further assistance including a coupon offer.

**Nodes Involved:**  
- AI Agent (Chat)  
- Simple Memory  
- Gmail Trigger  
- Reply to a message in Gmail

**Node Details:**

- **AI Agent (Chat)**  
  - *Type:* Langchain Agent  
  - *Role:* Handles negative feedback emails, generating questions to clarify issues and inviting customers to stay in touch. If sufficient info is already available, sends summary reply and creates Jira issue.  
  - *Configuration:* Prompt instructs agent to ask about device used, reproducible steps, and keep customer engaged with promise of a free coupon.  
  - *Key Expressions:* Uses `{{ $json.text }}` for email content.  
  - *Input:* Emails from Gmail Trigger and edited fields.  
  - *Output:* Email reply messages and Jira issue details.  
  - *Edge Cases:* AI misinterpretation of email content, missing data, API errors.

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains conversation context in the email thread using Gmail thread ID as session key, preserving last 10 interactions.  
  - *Configuration:* Session key derived dynamically from Gmail Trigger thread ID.  
  - *Input:* AI Agent (Chat) context and emails.  
  - *Output:* Context-aware AI responses.  
  - *Edge Cases:* Memory overflow, session key mismatch.

---

### 3. Summary Table

| Node Name                   | Node Type                       | Functional Role                                  | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                 |
|-----------------------------|--------------------------------|-------------------------------------------------|-------------------------|-------------------------|---------------------------------------------------------------------------------------------|
| JotForm Trigger             | JotForm Trigger                | Capture customer feedback from JotForm form     |                         | AI Agent                | Get the feedback from users [https://www.jotform.com/?partner=zainurrehman](https://www.jotform.com/?partner=zainurrehman) |
| AI Agent                   | Langchain Agent                | Analyze feedback, classify, generate reply      | JotForm Trigger         | Edit Fields             | Identify feedback if positive/negative and create Jira ticket/send initial response        |
| Edit Fields                | Set                           | Prepare reply text and thread ID for next steps | AI Agent                | AI Agent (Chat)         | Identify feedback if positive/negative and create Jira ticket/send initial response        |
| AI Agent (Chat)            | Langchain Agent                | Handle negative feedback via email follow-up    | Edit Fields, Gmail Trigger | Reply to a message in Gmail | This agent asks appropriate questions via email to get more insights                       |
| Gmail Trigger              | Gmail Trigger                 | Detect unread incoming emails                    |                         | AI Agent (Chat)         | This agent asks appropriate questions via email to get more insights                       |
| Google Gemini Chat Model   | Langchain Language Model       | Provide AI language processing for agents       |                         | AI Agent, AI Agent (Chat) | Identify feedback if positive/negative and create Jira ticket/send initial response        |
| Structured Output Parser   | Langchain Output Parser        | Parse AI output to structured JSON               | AI Agent                | AI Agent                | Identify feedback if positive/negative and create Jira ticket/send initial response        |
| Send a message in Gmail    | Gmail Tool (Send)              | Send thank-you email to customer                  | AI Agent                |                         | Identify feedback if positive/negative and create Jira ticket/send initial response        |
| Reply to a message in Gmail | Gmail Tool (Reply)             | Reply to existing Gmail threads                   | AI Agent (Chat), Gmail Trigger |                         | This agent asks appropriate questions via email to get more insights                       |
| Create an issue in Jira Software | Jira Tool                    | Create Jira issues for negative feedback          | AI Agent                |                         | Identify feedback if positive/negative and create Jira ticket/send initial response        |
| Simple Memory              | Langchain Memory Buffer Window | Maintain conversation context for AI Agent (Chat) | Gmail Trigger           | AI Agent (Chat)         | This agent asks appropriate questions via email to get more insights                       |
| Sticky Note                | Sticky Note                   | Informational/Instructional                       |                         |                         | Get the feedback from users [https://www.jotform.com/?partner=zainurrehman](https://www.jotform.com/?partner=zainurrehman) |
| Sticky Note1               | Sticky Note                   | Informational/Instructional                       |                         |                         | Identify the feedback request if positive or negative. Based on that create ticket on Jira and send an initial response |
| Sticky Note2               | Sticky Note                   | Informational/Instructional                       |                         |                         | This agent asks appropriate question from the user through email to get more insights       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Type: JotForm Trigger  
   - Configure webhook for form ID `252856264643060` (customer feedback form).  
   - Set `Resolve Data` to false (send raw data).  
   - Position appropriately.

2. **Add Google Gemini Chat Model Node**  
   - Type: Langchain Google Gemini Chat Model  
   - Use default settings.  
   - This node will serve as the language model for AI agents.

3. **Create AI Agent Node (Feedback Classification & Reply Generation)**  
   - Type: Langchain Agent  
   - Configure prompt to:  
     - Address business owner replying to customer feedback.  
     - Use expressions for user name: `{{ $json.q3_name.first }}` and feedback text: `{{ $json.q6_typeA6 }}`.  
     - Logic: If feedback positive, express appreciation and no Jira issue; if negative, acknowledge and plan Jira creation.  
   - Connect its language model input to the Google Gemini Chat Model node.  
   - Connect its output to a Structured Output Parser node.

4. **Create Structured Output Parser Node**  
   - Type: Langchain Structured Output Parser  
   - Provide a JSON schema example with fields `threadId` and `output`.  
   - Connect AI Agent output to this node.

5. **Add Edit Fields Node**  
   - Type: Set  
   - Create fields:  
     - `text` assigned as JotForm feedback field `q6_typeA6`.  
     - `threadId` assigned from AI Agent output property `threadId`.  
   - Connect Structured Output Parser output here.

6. **Create AI Agent (Chat) Node (Negative Feedback Follow-up)**  
   - Type: Langchain Agent  
   - Configure prompt to:  
     - Act as feedback assistant handling negative feedback.  
     - Ask for device info, reproducibility steps, and keep customer engaged.  
     - If info already provided, send thank you and summarize to Jira.  
     - Use email text input: `{{ $json.text }}`.  
   - Connect language model input to Google Gemini Chat Model node.  
   - Connect input from Edit Fields and Gmail Trigger nodes.

7. **Add Simple Memory Node**  
   - Type: Langchain Memory Buffer Window  
   - Configure session key to Gmail thread ID: `={{ $('Gmail Trigger').item.json.threadId }}`  
   - Context window length: 10 messages.  
   - Connect between Gmail Trigger and AI Agent (Chat).

8. **Add Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure to poll unread emails every minute.  
   - Disable simple mode for full email data.

9. **Add Send a message in Gmail Node**  
   - Type: Gmail Tool (Send)  
   - Set `sendTo` as email from JotForm field `q4_email`.  
   - Message content set from AI Agent output.  
   - Subject: "Thank you for your Response".  
   - Connect AI Agent output to this node as an AI tool.

10. **Add Reply to a message in Gmail Node**  
    - Type: Gmail Tool (Reply)  
    - Set `messageId` from Gmail Trigger’s message ID.  
    - Message content from AI Agent (Chat) output.  
    - Connect AI Agent (Chat) output and Gmail Trigger node here.

11. **Add Create an issue in Jira Software Node**  
    - Type: Jira Tool  
    - Configure project and issue type (select from your Jira instance).  
    - Summary and priority assigned dynamically from AI output.  
    - Connect AI Agent output here as an AI tool.

12. **Connect the nodes in this order:**  
    - JotForm Trigger → AI Agent → Structured Output Parser → Edit Fields → AI Agent (Chat)  
    - Gmail Trigger → Simple Memory → AI Agent (Chat) → Reply to a message in Gmail  
    - AI Agent → Send a message in Gmail  
    - AI Agent → Create an issue in Jira Software  

13. **Credential Setup:**  
    - Configure JotForm credentials for form webhook.  
    - Configure Google Gemini Chat Model API credentials.  
    - Connect Gmail OAuth2 credentials with appropriate scopes for reading, sending, and replying to emails.  
    - Configure Jira credentials with permissions to create issues.

14. **Test each step carefully:**  
    - Submit test feedback through JotForm.  
    - Confirm AI classification and email response.  
    - Send negative feedback and verify Jira issue creation and email follow-up.  
    - Monitor Gmail inbox for replies and AI follow-up questions.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                          |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Get the feedback from users via JotForm. Sign up link: [https://www.jotform.com/?partner=zainurrehman](https://www.jotform.com/?partner=zainurrehman) | Sticky Note on JotForm Trigger node.                                    |
| Identify feedback as positive or negative to create Jira tickets and send initial response automatically.       | Sticky Note spanning AI Agent and related nodes.                        |
| AI agent asks appropriate questions via email to collect more insights on negative feedback.                    | Sticky Note covering AI Agent (Chat), Gmail Trigger, and related nodes. |

---

This documentation provides a complete, structured reference for understanding, reproducing, and modifying the Automated Customer Feedback Triage & Follow-up workflow using n8n, leveraging JotForm, Google Gemini, Gmail, and Jira integrations with AI-powered decision making.