📨 Categorize and Label Existing Gmail Emails Automatically with GPT-4o mini

https://n8nworkflows.xyz/workflows/---categorize-and-label-existing-gmail-emails-automatically-with-gpt-4o-mini-6031


# 📨 Categorize and Label Existing Gmail Emails Automatically with GPT-4o mini

---
### 1. Workflow Overview

This workflow automates the categorization and labeling of existing emails in a Gmail inbox using AI powered by GPT-4o mini. It is designed for users who want to organize a batch of already received emails by applying relevant Gmail labels based on the email’s content, subject, and sender.

The workflow is logically divided into three main blocks:

- **1.1 Manual Trigger + Gmail Fetch + Label Checker:**  
  Starts the workflow manually, fetches a batch of existing Gmail messages, and filters out emails that already have labels to avoid reprocessing.

- **1.2 AI Categorization + Structured Output:**  
  Sends the subject, content, and sender of each unlabeled email to an AI Agent (GPT-4o mini) which analyzes and assigns a category. The AI’s output is parsed into structured JSON indicating the label.

- **1.3 Apply Labels Based on AI Output:**  
  Routes each email based on the AI-determined label using a Switch node and applies the corresponding Gmail label to the email.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger + Gmail Fetch + Label Checker

- **Overview:**  
  This block initiates the workflow manually, retrieves a specified number of existing Gmail emails, and filters out emails that already have labels to focus only on unlabeled messages.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get many messages (Gmail)  
  - Label Checker Filter (Filter)  
  - Sticky Note3 (Documentation)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on a user click.  
    - Configuration: Default, no parameters needed.  
    - Input: None  
    - Output: Triggers “Get many messages” node.  
    - Edge cases: None inherent; workflow runs only when manually triggered.

  - **Get many messages**  
    - Type: Gmail node (Get All Messages)  
    - Role: Retrieves a batch of emails from the Gmail inbox.  
    - Configuration: Operation “getAll” with default filters; fetch limit can be adjusted in node settings (default 50).  
    - Credentials: Requires Gmail OAuth2 credentials.  
    - Input: Triggered by Manual Trigger.  
    - Output: Sends data to “Label Checker Filter.”  
    - Edge Cases: API rate limits, authentication errors, network issues.  
    - Notes: Fetches all messages regardless of labels.

  - **Label Checker Filter**  
    - Type: Filter node  
    - Role: Removes emails that already have one or more labels applied.  
    - Configuration: Checks if the first label ID string does NOT contain “Label” (indicating no label assigned).  
    - Input: Messages from Gmail node.  
    - Output: Only unlabeled emails proceed to AI Agent.  
    - Edge Cases: Emails with multiple labels, no label field present, JSON path failures.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Provides contextual documentation about this block’s function and links to relevant n8n docs.

---

#### 2.2 AI Categorization + Structured Output

- **Overview:**  
  This block uses GPT-4o mini to analyze each email’s subject, content, and sender, categorizes the email, and parses the AI output into structured JSON for downstream routing.

- **Nodes Involved:**  
  - Give a Label AI Agent (AI Agent)  
  - Structured Output Parser (Output Parser)  
  - OpenAI Chat Model (Language Model)  
  - Sticky Note12 (Documentation)

- **Node Details:**

  - **Give a Label AI Agent**  
    - Type: AI Agent node (n8n-nodes-langchain.agent)  
    - Role: Takes email data and prompts GPT-4o mini to classify the email into predefined categories.  
    - Configuration:  
      - Prompt includes email subject, body, sender details.  
      - System message defines roles, categories (Work, Personal, Finance, Shopping, Travel, Newsletters, Others), instructions, and requires exact label formatting in JSON output.  
      - Output Parser enabled to parse AI response.  
    - Input: Emails filtered as unlabeled.  
    - Output: JSON with field `email_label` for the category.  
    - Edge Cases: AI misclassification, prompt timeout, invalid JSON response, category mismatch with Gmail labels.

  - **Structured Output Parser**  
    - Type: Structured Output Parser node  
    - Role: Parses the AI’s text response into structured JSON format.  
    - Configuration: Uses a JSON schema example: `{ "email_label": "business" }`  
    - Input: AI Agent text output.  
    - Output: Parsed JSON with category label.  
    - Edge Cases: Parsing errors on unexpected AI responses.

  - **OpenAI Chat Model**  
    - Type: Language Model node (OpenAI Chat)  
    - Role: Provides GPT-4o mini model for the AI Agent node.  
    - Configuration: Model set to “gpt-4o-mini”.  
    - Input: Prompt text from AI Agent node.  
    - Output: AI-generated classification text.  
    - Credentials: Requires OpenAI API key with GPT-4o mini access.  
    - Edge Cases: API quota limits, network errors, authentication errors.

  - **Sticky Note12**  
    - Type: Sticky Note  
    - Role: Describes AI categorization process and links to AI node documentation.

---

#### 2.3 Apply Labels Based on AI Output

- **Overview:**  
  Routes each email according to the AI-determined label and applies the corresponding Gmail label using Gmail nodes.

- **Nodes Involved:**  
  - Switch  
  - Gmail nodes: Work, Personal, Finance, Shopping, travel, Newsletters, Others  
  - Sticky Note13 (Documentation)

- **Node Details:**

  - **Switch**  
    - Type: Switch node  
    - Role: Routes emails based on `email_label` field from AI output.  
    - Configuration: One condition per category (Work, Personal, Finance, Shopping, Travel, Newsletters, Others).  
    - Input: Parsed JSON from AI Agent.  
    - Output: Routes to corresponding Gmail node.  
    - Edge Cases: Label mismatch, missing `email_label`, case sensitivity issues.

  - **Gmail Nodes (Work, Personal, Finance, Shopping, travel, Newsletters, Others)**  
    - Type: Gmail node (Add Labels to a Message)  
    - Role: Adds the Gmail label matching the routed category to the email message.  
    - Configuration:  
      - Operation: addLabels  
      - Message ID input references the email id from the trigger (note: the workflow uses a manual trigger, so message id sourcing must be consistent).  
      - Label names must exactly match Gmail existing labels.  
    - Credentials: Gmail OAuth2 required.  
    - Input: Routed emails from Switch node.  
    - Output: None (terminal nodes).  
    - Edge Cases: Gmail label missing, API auth errors, message id mismatch.

  - **Sticky Note13**  
    - Type: Sticky Note  
    - Role: Explains the routing and labeling process with references to Switch and Gmail node docs.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                             | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                                               |
|---------------------------|-------------------------------|---------------------------------------------|-----------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                | Starts workflow manually                     | None                        | Get many messages             |                                                                                                                                           |
| Get many messages          | Gmail                         | Fetches batch of emails from Gmail inbox    | When clicking ‘Execute workflow’ | Label Checker Filter          | [Gmail Get node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/draft-operations/#get-many-drafts) |
| Label Checker Filter       | Filter                        | Filters out emails that already have labels | Get many messages            | Give a Label AI Agent         | [Filter node documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.filter/)                                   |
| Give a Label AI Agent      | AI Agent (LangChain)          | Analyzes email and assigns category label  | Label Checker Filter         | Switch                       | [AI Agent node documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)                |
| Structured Output Parser   | Structured Output Parser      | Parses AI response into JSON                 | Give a Label AI Agent (ai_outputParser) | Give a Label AI Agent (ai_languageModel) | [Structured Output Parser docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.outputparserstructured/) |
| OpenAI Chat Model          | Language Model (OpenAI GPT)   | Provides GPT-4o mini model for AI Agent     | Give a Label AI Agent (ai_languageModel) | Structured Output Parser      |                                                                                                                                           |
| Switch                    | Switch                        | Routes emails to correct label branch       | Give a Label AI Agent        | Work, Personal, Finance, Shopping, travel, Newsletters, Others | [Switch node documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/)                                   |
| Work                      | Gmail                         | Applies "Work" label to email                | Switch                      | None                         | [Gmail label application docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/message-operations/#add-label-to-a-message) |
| Personal                  | Gmail                         | Applies "Personal" label                      | Switch                      | None                         | Same as above                                                                                                                             |
| Finance                   | Gmail                         | Applies "Finance" label                       | Switch                      | None                         | Same as above                                                                                                                             |
| Shopping                  | Gmail                         | Applies "Shopping" label                      | Switch                      | None                         | Same as above                                                                                                                             |
| travel                    | Gmail                         | Applies "Travel" label                        | Switch                      | None                         | Same as above                                                                                                                             |
| Newsletters               | Gmail                         | Applies "Newsletters" label                   | Switch                      | None                         | Same as above                                                                                                                             |
| Others                    | Gmail                         | Applies "Others" label                        | Switch                      | None                         | Same as above                                                                                                                             |
| Sticky Note3              | Sticky Note                   | Documentation for block 1                     | None                        | None                         | Covers Manual Trigger + Gmail Fetch + Label Checker block with relevant links                                                              |
| Sticky Note11             | Sticky Note                   | Workflow overview and usage instructions      | None                        | None                         | Covers entire workflow overview and setup instructions                                                                                     |
| Sticky Note12             | Sticky Note                   | Documentation for AI Categorization block     | None                        | None                         | Covers AI Categorization + Structured Output block with relevant links                                                                     |
| Sticky Note13             | Sticky Note                   | Documentation for Apply Labels block          | None                        | None                         | Covers Apply Labels based on AI output block with relevant links                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named “When clicking ‘Execute workflow’” to start the workflow manually.

2. **Add Gmail Node to Fetch Messages:**  
   - Add a Gmail node named “Get many messages”.  
   - Set operation to “getAll” to fetch multiple messages.  
   - Configure credentials with Gmail OAuth2.  
   - Optionally, set a limit (default 50) to control batch size.

3. **Add Filter Node to Check Labels:**  
   - Add a Filter node named “Label Checker Filter”.  
   - Set condition to exclude emails where `labelIds` field contains any label (i.e., only pass emails without labels).  
   - Use expression: check if `{{$json.labelIds[0]}}` does **not** contain “Label”.  
   - Connect “Get many messages” output to this filter.

4. **Add AI Agent Node:**  
   - Add an AI Agent node named “Give a Label AI Agent”.  
   - Configure to use OpenAI as LLM provider with “gpt-4o-mini” model.  
   - Set prompt to include email subject, content, and sender:  
     ```
     Topic: {{ $json.subject }}
     Description: {{ $json.text }}
     Sender: {{ $json.from.text }}
     ```  
   - Provide system message defining the categories exactly: Work, Personal, Finance, Shopping, Travel, Newsletters, Others, and instruct to output JSON with `email_label`.  
   - Enable output parser.

5. **Add OpenAI Chat Model Node:**  
   - Add “OpenAI Chat Model” node.  
   - Set model to “gpt-4o-mini”.  
   - Connect as language model provider for AI Agent node.  
   - Configure OpenAI credentials.

6. **Add Structured Output Parser Node:**  
   - Add a “Structured Output Parser” node.  
   - Provide JSON schema example: `{ "email_label": "business" }`.  
   - Connect AI Agent node’s output parser to this node.

7. **Add Switch Node:**  
   - Add a Switch node named “Switch”.  
   - Add one condition per category exactly matching the labels:  
     - Work  
     - Personal  
     - Finance  
     - Shopping  
     - Travel  
     - Newsletters  
     - Others  
   - Use expression: `{{$json.output.email_label}}` equals each category string.  
   - Connect AI Agent node output to Switch node.

8. **Add Gmail Nodes to Apply Labels:**  
   - For each category, add a Gmail node named after the category (e.g., “Work”).  
   - Set operation to “addLabels”.  
   - Set the messageId to the original email’s id (ensure the correct expression is used to get message id from incoming data).  
   - Configure credentials for Gmail OAuth2.  
   - Each node applies the corresponding Gmail label (labels must be pre-created in Gmail).

9. **Connect Switch Outputs to Corresponding Gmail Nodes:**  
   - Connect each Switch output branch to its respective Gmail node.

10. **Verify and Adjust:**  
    - Ensure all Gmail labels exist in the Gmail inbox and match exactly the categories defined.  
    - Test the workflow manually by clicking “Execute Workflow”.  
    - Check logs and Gmail inbox to verify labels applied correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                                |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for individuals or teams wanting to clean up unlabeled emails in bulk using AI without coding.                                                                                                             | Workflow Overview Sticky Note                                                                                                  |
| For sorting incoming Gmail messages automatically, use the complementary workflow: [Categorize and Label Incoming Gmail Emails Automatically with GPT-4o mini](https://n8n.io/workflows/5595-categorize-and-label-gmail-emails-automatically-with-gpt-4o-mini/) | Workflow Overview Sticky Note                                                                                                  |
| Gmail labels must be manually created in Gmail beforehand and must exactly match the categories defined in the AI prompt and Switch node to ensure correct labeling.                                                               | Workflow Overview Sticky Note                                                                                                  |
| The AI prompt and category list can be customized to fit specific needs or mailbox types (work, personal, client support).                                                                                                         | Workflow Overview Sticky Note                                                                                                  |
| Make sure OpenAI credentials have access to GPT-4o mini model for this workflow to function.                                                                                                                                       | AI Categorization Sticky Note                                                                                                  |
| When adjusting batch size in Gmail node, be aware of Gmail API rate limits and n8n execution time constraints.                                                                                                                    | Manual Trigger + Gmail Fetch Sticky Note                                                                                       |
| Switch node routing and Gmail label nodes must be kept in sync to prevent mislabeling.                                                                                                                                             | Apply Labels Sticky Note                                                                                                       |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.