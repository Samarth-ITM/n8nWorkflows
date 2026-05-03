Automatically Create Linear Issues from Gmail Support Request Messages

https://n8nworkflows.xyz/workflows/automatically-create-linear-issues-from-gmail-support-request-messages-3899


# Automatically Create Linear Issues from Gmail Support Request Messages

### 1. Workflow Overview

This workflow automates the creation of Linear issues from support request emails received in a Gmail inbox. It is designed for teams managing support tickets via email and issue tracking in Linear. The workflow is structured into three main logical blocks:

- **1.1 Input Reception:** Periodically fetches recent Gmail messages filtered for support requests and ensures each message is processed only once.
- **1.2 Content Preparation:** Converts the HTML email body to Markdown format for easier parsing and downstream processing.
- **1.3 AI Processing and Issue Creation:** Uses an AI agent to triage the support request by generating labels, priority, summary, and description, then creates a corresponding issue in Linear.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow on a schedule, fetches recent Gmail messages sent to a dedicated support email address, and filters out any messages that have already been processed to avoid duplication.

**Nodes Involved:**  
- Schedule Trigger  
- Get Recent Messages  
- Mark as Seen  
- Sticky Note (Input context)  
- Sticky Note4 (Gmail filter explanation)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow on an hourly interval to check for new emails.  
  - Configuration: Interval set to every 1 hour.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "Get Recent Messages"  
  - Edge Cases: If the trigger interval is too long, support requests may be delayed; too short may cause API rate limits.

- **Get Recent Messages**  
  - Type: Gmail node  
  - Role: Retrieves recent emails from Gmail inbox filtered by recipient address.  
  - Configuration:  
    - Operation: Get All  
    - Limit: 1 (fetches one message per execution; can be adjusted)  
    - Filter Query: `to:support@example.com` (captures emails sent to support)  
    - Simple: false (fetches full message data including HTML body)  
  - Credentials: Gmail OAuth2 account required  
  - Inputs: From Schedule Trigger  
  - Outputs: Connects to "Mark as Seen"  
  - Edge Cases:  
    - Gmail API quota limits  
    - Emails not matching filter are ignored  
    - If multiple messages arrive quickly, limit may need increasing  
  - Sticky Note4 explains the Gmail filter usage.

- **Mark as Seen**  
  - Type: Remove Duplicates  
  - Role: Ensures each email message is processed only once by removing duplicates based on message ID.  
  - Configuration:  
    - Operation: Remove items seen in previous executions  
    - Deduplication value: message ID (`{{$json.id}}`)  
  - Inputs: From "Get Recent Messages"  
  - Outputs: Connects to "Markdown"  
  - Edge Cases: If message IDs are not unique or change, duplicates may be processed multiple times.

- **Sticky Note (Input context)**  
  - Provides contextual information about watching the Gmail inbox for support emails, emphasizing the assumption of a dedicated support inbox and the importance of deduplication.

- **Sticky Note4 (Gmail Filters)**  
  - Explains the Gmail filter used to capture support requests.

---

#### 1.2 Content Preparation

**Overview:**  
This block converts the HTML content of the email body into Markdown format, which is easier for the AI agent to process and parse.

**Nodes Involved:**  
- Markdown

**Node Details:**

- **Markdown**  
  - Type: Markdown node  
  - Role: Converts HTML email body to Markdown text.  
  - Configuration:  
    - HTML input: `{{$json.html}}` (the HTML body of the email)  
    - Options: Default (no special options)  
  - Inputs: From "Mark as Seen"  
  - Outputs: Connects to "Generate Issue From Support Request"  
  - Edge Cases:  
    - Malformed HTML may cause conversion issues  
    - Emails without HTML body may produce empty or invalid Markdown

---

#### 1.3 AI Processing and Issue Creation

**Overview:**  
This block uses an AI language model to triage the support request by generating labels, priority, summary, and description. The output is parsed into structured data and then used to create a new issue in Linear.

**Nodes Involved:**  
- Generate Issue From Support Request  
- OpenAI Chat Model  
- Structured Output Parser  
- Create Issue in Linear.App  
- Sticky Note1 (AI triage explanation)  
- Sticky Note2 (Linear issue creation explanation)

**Node Details:**

- **Generate Issue From Support Request**  
  - Type: Chain LLM (Langchain)  
  - Role: Sends a prompt to the AI to classify, label, prioritize, and rewrite the issue summary and description.  
  - Configuration:  
    - Prompt includes: reporter name and email, report time, original subject, and Markdown description.  
    - System prompt instructs AI to:  
      - Assign one or more labels from a predefined list (Technical, Account, Access, Billing, Product, Training, Feedback, Complaints, Security, Privacy)  
      - Assign priority on a scale from 1 (highest) to 5 (lowest)  
      - Rewrite summary and description removing emotional content and focusing on facts and attempted solutions  
    - Output parser enabled to enforce structured JSON output.  
  - Inputs: From "Markdown" (Markdown email body) and from "OpenAI Chat Model" (AI model node)  
  - Outputs: Connects to "Create Issue in Linear.App"  
  - Edge Cases:  
    - AI model may produce unexpected or malformed output if prompt is ambiguous  
    - OpenAI API rate limits or errors  
    - Missing or incomplete email data may reduce AI accuracy

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model (Langchain)  
  - Role: Provides the AI language model backend for the chain LLM node.  
  - Configuration:  
    - Model: GPT-4o-mini (a GPT-4 variant optimized for cost/performance)  
    - No special options set  
  - Credentials: OpenAI API key required  
  - Inputs: From "Generate Issue From Support Request" (as AI language model)  
  - Outputs: To "Generate Issue From Support Request" (as AI output parser)  
  - Edge Cases: API key invalid or expired, network issues, model unavailability

- **Structured Output Parser**  
  - Type: Structured Output Parser (Langchain)  
  - Role: Parses AI output into a structured JSON object with properties: labels (array of strings), priority (number), summary (string), description (string).  
  - Configuration:  
    - JSON schema manually defined to enforce output format  
  - Inputs: From "OpenAI Chat Model" (AI raw output)  
  - Outputs: To "Generate Issue From Support Request" (as AI output parser)  
  - Edge Cases: Parsing errors if AI output does not conform to schema

- **Create Issue in Linear.App**  
  - Type: Linear node  
  - Role: Creates a new issue in Linear using AI-generated data.  
  - Configuration:  
    - Title: AI-generated summary  
    - Team ID: Fixed UUID identifying the Linear team  
    - Additional fields:  
      - State ID: Fixed UUID for issue state  
      - Priority ID: AI-generated priority or default 3 (medium)  
      - Description: AI-generated description appended with labels formatted as hashtags  
  - Credentials: Linear API key required  
  - Inputs: From "Generate Issue From Support Request"  
  - Outputs: None (end of workflow)  
  - Edge Cases:  
    - Linear API errors (auth, rate limits)  
    - Missing or invalid team or state IDs  
    - AI output missing required fields

- **Sticky Note1 (AI triage explanation)**  
  - Explains the AI triage step, its purpose, and provides a link to the n8n documentation for the Basic LLM node.

- **Sticky Note2 (Linear issue creation explanation)**  
  - Describes the Linear issue creation node and notes it is a simple example easily extendable.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                          | Input Node(s)                | Output Node(s)                      | Sticky Note                                                                                                         |
|-------------------------------|----------------------------------|----------------------------------------|-----------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                 | Initiates workflow on schedule         | None                        | Get Recent Messages               | ## 1. Watch Gmail Inbox for Support Emails<br>[Learn more about the Gmail node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) |
| Get Recent Messages            | Gmail                           | Fetches recent support emails          | Schedule Trigger            | Mark as Seen                     | ### Gmail Filters<br>Here we're using the filter `to:support@example.com` to capture support requests.             |
| Mark as Seen                  | Remove Duplicates               | Filters out already processed emails   | Get Recent Messages         | Markdown                        | ## 1. Watch Gmail Inbox for Support Emails<br>**This template assumes a group email specifically for support tickets!** The "remove duplicates" node ensures each email is processed once. |
| Markdown                      | Markdown                       | Converts HTML email body to Markdown   | Mark as Seen                | Generate Issue From Support Request |                                                                                                                     |
| Generate Issue From Support Request | Chain LLM (Langchain)           | AI triage: labels, priority, summary  | Markdown, OpenAI Chat Model | Create Issue in Linear.App       | ## 2. Automate Generation and Triaging of Ticket<br>[Read more about the Basic LLM node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) |
| OpenAI Chat Model             | OpenAI Chat Model (Langchain)  | Provides AI language model backend     | Generate Issue From Support Request (ai_languageModel) | Structured Output Parser         |                                                                                                                     |
| Structured Output Parser      | Structured Output Parser (Langchain) | Parses AI output into structured JSON  | OpenAI Chat Model (ai_outputParser) | Generate Issue From Support Request (ai_outputParser) |                                                                                                                     |
| Create Issue in Linear.App    | Linear                         | Creates issue in Linear from AI output | Generate Issue From Support Request | None                            | ## 3. Create Issue in Linear.App<br>[Read more about the Linear.App node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.linear) |
| Sticky Note                   | Sticky Note                   | Provides context on Gmail inbox watch  | None                        | None                            | ## 1. Watch Gmail Inbox for Support Emails<br>**This template assumes a group email specifically for support tickets!** The "remove duplicates" node ensures each email is processed once. |
| Sticky Note1                  | Sticky Note                   | Provides context on AI triage step     | None                        | None                            | ## 2. Automate Generation and Triaging of Ticket<br>[Read more about the Basic LLM node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) |
| Sticky Note2                  | Sticky Note                   | Provides context on Linear issue creation | None                        | None                            | ## 3. Create Issue in Linear.App<br>[Read more about the Linear.App node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.linear) |
| Sticky Note4                  | Sticky Note                   | Explains Gmail filter usage            | None                        | None                            | ### Gmail Filters<br>Here we're using the filter `to:support@example.com` to capture support requests.             |
| Sticky Note3                  | Sticky Note                   | Overview and instructions for the workflow | None                        | None                            | ## Try It Out!<br>This n8n template watches a Gmail inbox for support messages and creates an equivalent issue item in Linear.<br>... (full content in workflow overview) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to every 1 hour.  
   - This node triggers the workflow periodically.

2. **Add a Gmail node named "Get Recent Messages"**  
   - Operation: Get All  
   - Limit: 1 (adjust as needed)  
   - Filters: Set query to `to:support@example.com` to capture support emails.  
   - Credentials: Connect with a Gmail OAuth2 credential authorized to access the support inbox.  
   - Connect Schedule Trigger output to this node.

3. **Add a Remove Duplicates node named "Mark as Seen"**  
   - Operation: Remove items seen in previous executions  
   - Deduplication value: Use expression `{{$json.id}}` to deduplicate by Gmail message ID.  
   - Connect output of "Get Recent Messages" to this node.

4. **Add a Markdown node named "Markdown"**  
   - Set HTML input to `{{$json.html}}` to convert the email's HTML body to Markdown.  
   - Connect output of "Mark as Seen" to this node.

5. **Add an OpenAI Chat Model node named "OpenAI Chat Model"**  
   - Select model: GPT-4o-mini (or equivalent GPT-4 variant).  
   - Credentials: Use OpenAI API key credential.  
   - No special options needed.  
   - This node will be connected as the AI language model backend for the Chain LLM node.

6. **Add a Chain LLM node named "Generate Issue From Support Request"**  
   - Prompt Type: Define prompt  
   - Text prompt:  
     ```
     Reported by {{ $json.from.value[0].name }} <{{ $json.from.value[0].address }}>
     Reported at: {{ $now.toISO() }}
     Summary: {{ $json.subject }}
     Description:
     {{ $json.data.replaceAll('\n', ' ') }}
     ```
   - System prompt instructing AI to:  
     - Classify and label the issue using predefined labels: Technical, Account, Access, Billing, Product, Training, Feedback, Complaints, Security, Privacy  
     - Prioritize issue on scale 1 (highest) to 5 (lowest)  
     - Rewrite summary and description removing emotional content and focusing on facts and attempted solutions  
   - Enable output parser with manual JSON schema:  
     ```json
     {
       "type": "object",
       "properties": {
         "labels": {
           "type": "array",
           "items": { "type": "string" }
         },
         "priority": { "type": "number" },
         "summary": { "type": "string" },
         "description": { "type": "string" }
       }
     }
     ```
   - Connect input from "Markdown" node.  
   - Set AI language model to "OpenAI Chat Model" node.  
   - Set AI output parser to "Structured Output Parser" node (next step).

7. **Add a Structured Output Parser node named "Structured Output Parser"**  
   - Schema Type: Manual  
   - Input Schema: Use the same JSON schema as above.  
   - Connect AI output from "OpenAI Chat Model" to this node.  
   - Connect output of this node back to "Generate Issue From Support Request" as AI output parser.

8. **Add a Linear node named "Create Issue in Linear.App"**  
   - Operation: Create Issue  
   - Title: Set to `{{$json.output.summary}}` (AI-generated summary)  
   - Team ID: Use your Linear team UUID (e.g., `1c721608-321d-4132-ac32-6e92d04bb487`)  
   - Additional Fields:  
     - State ID: Set to your desired issue state UUID (e.g., `92962324-3d1f-4cf8-993b-0c982cc95245`)  
     - Priority ID: Use expression `{{$json.output.priority ?? 3}}` (default to 3 if missing)  
     - Description: Combine AI-generated description and labels as hashtags, e.g.:  
       ```
       {{$json.output.description}}

       {{$json.output.labels.map(label => `#${label}`).join(' ')}}
       ```  
   - Credentials: Connect with Linear API key credential.  
   - Connect output of "Generate Issue From Support Request" to this node.

9. **Connect nodes in the following order:**  
   Schedule Trigger → Get Recent Messages → Mark as Seen → Markdown → Generate Issue From Support Request → Create Issue in Linear.App  
   Also, connect "Generate Issue From Support Request" AI language model input to "OpenAI Chat Model" and AI output parser input to "Structured Output Parser".

10. **Add Sticky Notes for documentation (optional):**  
    - Add notes explaining Gmail inbox watching, AI triage, and Linear issue creation with relevant links as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow assumes a dedicated support email inbox to avoid processing irrelevant emails. If using a general inbox, additional classification and filtering may be required, which could increase costs due to AI usage.                                                                                                                                                          | Workflow Overview                                                                                               |
| The AI triage step uses a predefined set of labels and priority levels; customize these in the system prompt to fit your organization's taxonomy.                                                                                                                                                                                                                                | Workflow Overview / Node "Generate Issue From Support Request"                                                  |
| The Linear issue creation node uses fixed Team and State IDs; replace these with your own Linear workspace values.                                                                                                                                                                                                                                                             | Node "Create Issue in Linear.App"                                                                               |
| Consider extending the workflow to automate further steps after issue creation, such as automated resolution attempts or capacity planning.                                                                                                                                                                                                                                     | Workflow Overview                                                                                                |
| For detailed node documentation, refer to: <br> - Gmail Node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ <br> - Linear Node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.linear <br> - Langchain Nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm | Sticky Notes in workflow                                                                                         |
| Join the n8n community for support and discussion: <br> Discord: https://discord.com/invite/XPKeKXeB7d <br> Forum: https://community.n8n.io/                                                                                                                                                                                                                                      | Sticky Note3                                                                                                    |

---

This document provides a complete, structured reference to understand, reproduce, and customize the "Automatically Create Linear Issues from Gmail Support Request Messages" workflow in n8n. It covers all nodes, their configurations, logical flow, and potential edge cases to facilitate robust usage and extension.