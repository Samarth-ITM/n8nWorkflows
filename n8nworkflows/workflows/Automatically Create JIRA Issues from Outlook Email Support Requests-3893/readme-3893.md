Automatically Create JIRA Issues from Outlook Email Support Requests

https://n8nworkflows.xyz/workflows/automatically-create-jira-issues-from-outlook-email-support-requests-3893


# Automatically Create JIRA Issues from Outlook Email Support Requests

### 1. Workflow Overview

This workflow automates the creation of JIRA issues from support requests received via a shared Outlook inbox. It is designed for organizations that manage support tickets through email and want to streamline issue tracking by automatically generating JIRA tickets enriched with AI-assisted triage.

The workflow is logically divided into three main blocks:

- **1.1 Watch Outlook Inbox for Support Emails**  
  Periodically fetches recent emails from a shared Outlook inbox dedicated to support requests, ensuring each message is processed only once.

- **1.2 AI-Powered Ticket Generation and Triaging**  
  Converts the email body from HTML to markdown, then uses an AI language model to analyze the support request, assigning labels, priority, and generating a concise summary and description.

- **1.3 Create JIRA Issue**  
  Uses the AI-generated metadata to create a new issue in JIRA, populating fields such as summary, description, labels, and priority.

---

### 2. Block-by-Block Analysis

#### 2.1 Watch Outlook Inbox for Support Emails

- **Overview:**  
  This block triggers the workflow on a schedule, fetches recent emails from a shared Outlook inbox, and filters out duplicates to ensure each email is processed exactly once.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Recent Messages (Microsoft Outlook)  
  - Mark as Seen (Remove Duplicates)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates workflow every hour to check for new emails.  
    - *Configuration:* Interval set to 1 hour.  
    - *Input/Output:* No input; outputs trigger event to "Get Recent Messages".  
    - *Edge Cases:* If the workflow is paused or delayed, emails may accumulate; no direct error expected here.

  - **Get Recent Messages**  
    - *Type:* Microsoft Outlook node  
    - *Role:* Retrieves all emails received in the last hour from the shared inbox.  
    - *Configuration:*  
      - Fields fetched include body, categories, conversationId, sender info, subject, attachments, received time, and web link.  
      - Filter applied: receivedAfter = current time minus 1 hour.  
      - Credentials: OAuth2 for Microsoft Outlook account.  
    - *Input/Output:* Trigger input from Schedule Trigger; outputs array of email messages.  
    - *Edge Cases:*  
      - Authentication failures (expired token).  
      - API rate limits or connectivity issues.  
      - Emails outside the expected format or missing fields.

  - **Mark as Seen**  
    - *Type:* Remove Duplicates  
    - *Role:* Filters out emails already processed in previous workflow runs based on unique message ID.  
    - *Configuration:*  
      - Operation: Remove items seen in previous executions.  
      - Deduplication key: email message ID.  
    - *Input/Output:* Input from Get Recent Messages; outputs unique new emails only.  
    - *Edge Cases:*  
      - If message IDs are missing or duplicated, may cause processing errors or duplicates.  
      - Persistence of deduplication data depends on n8n instance storage.

---

#### 2.2 AI-Powered Ticket Generation and Triaging

- **Overview:**  
  Converts the email body from HTML to markdown, then sends the formatted content to an AI language model that triages the ticket by assigning labels, priority, and rewriting the summary and description.

- **Nodes Involved:**  
  - Markdown  
  - OpenAI Chat Model (GPT-4o-mini)  
  - Structured Output Parser  
  - Generate Issue From Support Request (Chain LLM)

- **Node Details:**

  - **Markdown**  
    - *Type:* Markdown node  
    - *Role:* Converts the HTML email body content to markdown for easier parsing by AI.  
    - *Configuration:* Input HTML is taken from the email body content field.  
    - *Input/Output:* Input from Mark as Seen node; outputs markdown text.  
    - *Edge Cases:*  
      - Malformed HTML may cause conversion errors or inaccurate markdown.  
      - Empty or missing body content.

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model node  
    - *Role:* Provides the AI language model interface (GPT-4o-mini) used for triaging.  
    - *Configuration:*  
      - Model: gpt-4o-mini (a compact GPT-4 variant).  
      - No additional options set.  
      - Credentials: OpenAI API key.  
    - *Input/Output:* Receives prompt messages from Generate Issue From Support Request node; outputs AI response.  
    - *Edge Cases:*  
      - API rate limits or quota exceeded.  
      - Network or authentication errors.  
      - Model response delays or failures.

  - **Structured Output Parser**  
    - *Type:* LangChain Structured Output Parser  
    - *Role:* Parses AI output into a structured JSON object with defined schema.  
    - *Configuration:*  
      - Schema expects an object with properties: labels (array of strings), priority (number), summary (string), description (string).  
    - *Input/Output:* Parses output from OpenAI Chat Model; outputs structured data for next node.  
    - *Edge Cases:*  
      - AI output not matching schema causing parse errors.  
      - Missing or malformed fields.

  - **Generate Issue From Support Request**  
    - *Type:* LangChain Chain LLM node  
    - *Role:* Constructs the prompt for AI triage and processes AI output.  
    - *Configuration:*  
      - Prompt includes reporter name/email, timestamp, original subject, and cleaned description.  
      - System prompt instructs AI to classify labels, assign priority, and rewrite summary/description with specific label options and priority scale.  
      - Output parser enabled to enforce structured output.  
    - *Input/Output:* Input from Markdown node (markdown email body); outputs structured AI triage data.  
    - *Edge Cases:*  
      - Expression failures in prompt construction.  
      - AI misclassification or inappropriate labels/priorities.  
      - Unexpected AI output format.

---

#### 2.3 Create JIRA Issue

- **Overview:**  
  Creates a new issue in JIRA using the AI-generated summary, description, labels, and priority.

- **Nodes Involved:**  
  - Create Issue (JIRA)

- **Node Details:**

  - **Create Issue**  
    - *Type:* JIRA node  
    - *Role:* Creates a new issue in a specified JIRA project.  
    - *Configuration:*  
      - Project ID: 10000 (configurable).  
      - Issue Type ID: 10000 (configurable).  
      - Summary, description, labels, and priority dynamically set from AI output fields.  
      - Credentials: JIRA Software Cloud API OAuth2.  
    - *Input/Output:* Input from Generate Issue From Support Request node; outputs created issue metadata.  
    - *Edge Cases:*  
      - Authentication or permission errors with JIRA API.  
      - Invalid field values causing issue creation failure.  
      - Network or API rate limiting issues.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                         | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                          |
|-----------------------------|----------------------------------|---------------------------------------|----------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                  | Periodic trigger to start workflow    | -                          | Get Recent Messages            | ## 1. Watch Outlook Inbox for Support Emails [Learn more about the Outlook node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook/) **This template assumes a shared inbox specifically for support tickets!** If you have a general inbox, you may need to classify and filter each message which might become costly. The "remove duplicates" node (ie. "Mark as seen") ensures we only process each email exactly once. |
| Get Recent Messages         | Microsoft Outlook                | Fetch recent emails from shared inbox | Schedule Trigger            | Mark as Seen                   | See above                                                                                                            |
| Mark as Seen                | Remove Duplicates                | Filter out already processed emails   | Get Recent Messages         | Markdown                      | See above                                                                                                            |
| Markdown                   | Markdown                        | Convert HTML email body to markdown   | Mark as Seen                | Generate Issue From Support Request | ## 2. Automate Generation and Triaging of Ticket [Read more about the Basic LLM node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) New tickets always need to be properly labelled and prioritised but it's not always possible to get to update all incoming tickets if you're light on hands. Using an AI is a great use-case for triaging of tickets as its contextual understanding helps automates this step. |
| OpenAI Chat Model           | LangChain OpenAI Chat Model      | AI language model for triaging        | Generate Issue From Support Request (ai_languageModel) | Structured Output Parser       | See above                                                                                                            |
| Structured Output Parser    | LangChain Structured Output Parser | Parse AI output into structured JSON | OpenAI Chat Model (ai_outputParser) | Generate Issue From Support Request (ai_outputParser) | See above                                                                                                            |
| Generate Issue From Support Request | LangChain Chain LLM             | Build prompt, triage ticket, parse AI output | Markdown, OpenAI Chat Model, Structured Output Parser | Create Issue                  | See above                                                                                                            |
| Create Issue                | JIRA                            | Create JIRA issue from AI output      | Generate Issue From Support Request | -                             | ## 3. Create Issue in JIRA [Read more about the JIRA node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.jira/) This is only a simple example to create an issue in JIRA but easily extendable to add much more! |
| Sticky Note1                | Sticky Note                     | Informative note on AI triage          | -                          | -                             | See Markdown node note above                                                                                         |
| Sticky Note                 | Sticky Note                     | Informative note on Outlook inbox setup | -                          | -                             | See Schedule Trigger note above                                                                                      |
| Sticky Note2                | Sticky Note                     | Informative note on JIRA issue creation | -                          | -                             | See Create Issue note above                                                                                          |
| Sticky Note3                | Sticky Note                     | General workflow overview and usage instructions | -                          | -                             | ## Try It Out! This n8n template watches an outlook shared inbox for support messages and creates an equivalent issue item in JIRA. [Discord](https://discord.com/invite/XPKeKXeB7d) [Forum](https://community.n8n.io/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Set interval to run every 1 hour.  
   - No credentials needed.  

2. **Create "Get Recent Messages" node**  
   - Type: Microsoft Outlook  
   - Operation: getAll  
   - Fields to fetch: body, categories, conversationId, from, hasAttachments, internetMessageId, sender, subject, toRecipients, receivedDateTime, webLink  
   - Filter: receivedAfter = `={{ $now.minus({ "hour": 1 }).toISO() }}`  
   - Credentials: Configure Microsoft Outlook OAuth2 credentials for the shared inbox.  
   - Connect output of "Schedule Trigger" to this node's input.

3. **Create "Mark as Seen" node**  
   - Type: Remove Duplicates  
   - Operation: Remove items seen in previous executions  
   - Deduplication key: `={{ $json.id }}` (unique message ID)  
   - Connect output of "Get Recent Messages" to this node's input.

4. **Create "Markdown" node**  
   - Type: Markdown  
   - Input HTML: `={{ $json.body.content }}` (email body content)  
   - Connect output of "Mark as Seen" to this node's input.

5. **Create "Generate Issue From Support Request" node**  
   - Type: LangChain Chain LLM node  
   - Prompt setup:  
     - Text template:  
       ```
       Reported by {{ $json.from.emailAddress.name }} <{{ $json.from.emailAddress.address }}>
       Reported at: {{ $now.toISO() }}
       Summary: {{ $json.subject }}
       Description:
       {{ $json.data.replaceAll('\n', ' ') }}
       ```  
     - System prompt instructing AI to:  
       1) classify and label the issue using predefined labels (Technical, Account, Access, Billing, Product, Training, Feedback, Complaints, Security, Privacy)  
       2) assign priority (1 highest to 5 lowest)  
       3) rewrite summary and description removing emotional/anecdotal content, focusing on facts and attempted actions.  
   - Enable output parser with schema:  
     ```json
     {
       "type": "object",
       "properties": {
         "labels": { "type": "array", "items": { "type": "string" } },
         "priority": { "type": "number" },
         "summary": { "type": "string" },
         "description": { "type": "string" }
       }
     }
     ```  
   - Connect output of "Markdown" node to this node's main input.

6. **Create "OpenAI Chat Model" node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: gpt-4o-mini  
   - Credentials: Configure OpenAI API key.  
   - Connect this node to the AI language model input of "Generate Issue From Support Request".

7. **Create "Structured Output Parser" node**  
   - Type: LangChain Structured Output Parser  
   - Schema: same as above.  
   - Connect this node to the AI output parser input of "Generate Issue From Support Request".  
   - Connect output of this node back to "Generate Issue From Support Request" for parsing.

8. **Create "Create Issue" node**  
   - Type: JIRA  
   - Project: Set project ID (e.g., 10000)  
   - Issue Type: Set issue type ID (e.g., 10000)  
   - Summary: `={{ $json.output.summary }}` (from AI output)  
   - Description: `={{ $json.output.description }}`  
   - Labels: `={{ $json.output.labels }}`  
   - Priority: `={{ $json.output.priority }}` (mapped to JIRA priority ID)  
   - Credentials: Configure JIRA Software Cloud API OAuth2 credentials.  
   - Connect output of "Generate Issue From Support Request" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow assumes a shared Outlook inbox dedicated to support requests. If your inbox contains mixed emails, consider adding classification/filtering steps before processing to avoid creating irrelevant JIRA issues.                                                                                                                                     | Outlook node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook/ |
| The AI triage step uses a custom system prompt to assign labels and priorities. Adjust the labels and priorities in the prompt to fit your organization's taxonomy and workflow.                                                                                                                                                                              | Basic LLM node docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm |
| The JIRA node configuration here is a simple example. You can extend it to include additional fields such as assignee, components, custom fields, or attachments to better integrate with your issue management process.                                                                                                                                       | JIRA node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.jira/             |
| For community support and questions, join the n8n Discord or Forum.                                                                                                                                                                                                                                                                                            | Discord: https://discord.com/invite/XPKeKXeB7d, Forum: https://community.n8n.io/                     |
| Consider extending this workflow by automating post-creation steps such as issue resolution suggestions, capacity planning, or notifications to relevant teams.                                                                                                                                                                                               | -                                                                                                  |

---

This structured documentation provides a comprehensive understanding of the workflow, enabling reproduction, modification, and troubleshooting by users and automation agents alike.