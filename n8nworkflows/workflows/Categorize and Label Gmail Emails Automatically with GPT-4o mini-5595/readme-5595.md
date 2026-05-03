Categorize and Label Gmail Emails Automatically with GPT-4o mini

https://n8nworkflows.xyz/workflows/categorize-and-label-gmail-emails-automatically-with-gpt-4o-mini-5595


# Categorize and Label Gmail Emails Automatically with GPT-4o mini

### 1. Workflow Overview

This workflow automates the categorization and labeling of incoming Gmail emails using AI (GPT-4o mini). It targets users who receive many emails and want to organize them automatically into predefined Gmail labels without manual sorting or coding.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Filtering:** Detects new incoming Gmail messages and filters out emails that are already labeled, ensuring only unlabeled emails are processed.

- **1.2 AI Processing and Classification:** Sends the email content to an AI agent powered by GPT-4o mini, which analyzes the subject, body, and sender to determine the most appropriate category label. The AI output is structured into JSON for further processing.

- **1.3 Label Application:** Uses a Switch node to route the email to the correct Gmail node that applies the label returned by the AI. Each Gmail node adds a specific label to the email.

---

### 2. Block-by-Block Analysis

#### Block 1: Input Reception and Filtering

- **Overview:**  
  This block triggers the workflow on new incoming Gmail emails and filters out those that have existing labels to avoid redundant processing.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Label Checker Filter  
  - Sticky Note3 (documentation)

- **Node Details:**

  1. **Gmail Trigger**  
     - *Type:* Trigger node for Gmail  
     - *Role:* Listens for new incoming emails in Gmail.  
     - *Configuration:* Polls every minute to check new emails. Processes all emails (simple mode off).  
     - *Credentials:* Gmail OAuth2 account connected.  
     - *Input:* None (trigger)  
     - *Output:* Emits new email data with full metadata including subject, body, sender, labels, and ID.  
     - *Edge Cases:* Gmail API quota limits; connection/authentication failures; delayed polling interval may miss emails in real time.

  2. **Label Checker Filter**  
     - *Type:* Filter node  
     - *Role:* Filters out emails that already have labels applied.  
     - *Configuration:* Checks if the `labelIds` array on the email contains any label; if yes, filters out (only passes emails with no labels).  
     - *Input:* Incoming emails from Gmail Trigger  
     - *Output:* Only unlabeled emails proceed.  
     - *Edge Cases:* Emails without `labelIds` field could cause expression errors; ensure correct JSON path and case sensitivity; if Gmail label structure changes, this node may need updates.

  3. **Sticky Note3**  
     - *Type:* Documentation  
     - *Role:* Explains the purpose and usage of the Gmail Trigger and Filter nodes, including links to official docs and tips for adjusting polling frequency.

---

#### Block 2: AI Processing and Classification

- **Overview:**  
  Takes the filtered unlabeled email content and sends it to an AI agent (GPT-4o mini) to classify the email into predefined categories. The AI output is parsed into structured JSON.

- **Nodes Involved:**  
  - Give a Label AI Agent  
  - OpenAI Chat Model2  
  - Structured Output Parser1  
  - Sticky Note12 (documentation)

- **Node Details:**

  1. **Give a Label AI Agent**  
     - *Type:* Langchain agent node (AI agent)  
     - *Role:* Analyzes email subject, content, and sender using GPT-4o mini and outputs a JSON with the target label category.  
     - *Configuration:*  
       - Prompt includes email fields: subject, text, sender.  
       - System message defines role and categories: Work, Personal, Finance, Shopping, Travel, Newsletters, Others.  
       - Instructions specify JSON output with exact label names and to skip labeling if unsure.  
       - Output parser enabled for structured response.  
     - *Input:* Email data from Label Checker Filter (with subject, body, sender)  
     - *Output:* JSON object `{ "email_label": "Category" }`  
     - *Credentials:* OpenAI API with GPT-4o mini access  
     - *Edge Cases:* AI may produce unexpected or malformed output; ensure prompt clarity; API rate limits or errors; empty or ambiguous emails may cause incorrect or null labels; network issues.

  2. **OpenAI Chat Model2**  
     - *Type:* Langchain OpenAI Chat Model node  
     - *Role:* Provides GPT-4o mini model backend for the AI agent.  
     - *Configuration:* Uses GPT-4o mini model, no additional options set.  
     - *Input:* AI agent requests  
     - *Output:* AI completions sent back to agent node  
     - *Credentials:* OpenAI API key  
     - *Edge Cases:* API quota, latency, or downtime may cause failures.

  3. **Structured Output Parser1**  
     - *Type:* Langchain output parser node  
     - *Role:* Parses AI text output into structured JSON format, validating against example schema.  
     - *Configuration:* Uses JSON schema example `{"email_label": "business"}` for validation.  
     - *Input:* AI agent raw output  
     - *Output:* Parsed JSON to downstream nodes  
     - *Edge Cases:* Parsing errors if AI output is malformed; invalid JSON; unexpected response format.

  4. **Sticky Note12**  
     - *Type:* Documentation  
     - *Role:* Explains how the AI agent and output parser nodes work together, with links to official docs.

---

#### Block 3: Label Application Based on AI Output

- **Overview:**  
  This block routes the classified email to the correct Gmail node according to the AI-determined label and applies the corresponding Gmail label.

- **Nodes Involved:**  
  - Switch  
  - Gmail nodes for each category: Work, Personal, Finance, Shopping, travel, Newsletters, Others  
  - Sticky Note13 (documentation)

- **Node Details:**

  1. **Switch**  
     - *Type:* Switch node  
     - *Role:* Routes workflow execution based on the `email_label` property from AI output.  
     - *Configuration:* Has branches for each category label (Work, Personal, Finance, Shopping, Travel, Newsletters, Others). Matches exact string equality, case sensitive.  
     - *Input:* Parsed AI output JSON `{ "email_label": "Category" }`  
     - *Output:* One branch per category leading to corresponding Gmail node.  
     - *Edge Cases:* If AI label does not match any branch, email will not be labeled; case sensitivity can cause misrouting; missing labels in Gmail cause errors downstream.

  2. **Gmail Nodes (Work, Personal, Finance, Shopping, travel, Newsletters, Others)**  
     - *Type:* Gmail message operation node  
     - *Role:* Adds a Gmail label to the email identified by message ID.  
     - *Configuration:*  
       - Operation: addLabels  
       - Message ID: taken from the original Gmail Trigger node’s item JSON id  
       - Each node applies a different label matching the AI categories.  
     - *Input:* From Switch node  
     - *Output:* Confirmation of label application or error  
     - *Credentials:* Same Gmail OAuth2 account as trigger  
     - *Edge Cases:* Labels must exist in Gmail beforehand; applying non-existent labels will cause API errors; message ID must be valid and accessible; Gmail API quota or errors.

  3. **Sticky Note13**  
     - *Type:* Documentation  
     - *Role:* Explains the use of the Switch and Gmail nodes for label application, with official documentation links.

---

### 3. Summary Table

| Node Name             | Node Type                              | Functional Role                     | Input Node(s)          | Output Node(s)              | Sticky Note                                                                                           |
|-----------------------|--------------------------------------|-----------------------------------|-----------------------|----------------------------|-----------------------------------------------------------------------------------------------------|
| Gmail Trigger         | n8n-nodes-base.gmailTrigger           | Trigger on new incoming emails    | None                  | Label Checker Filter         | See Sticky Note3 (Input reception and filtering docs)                                               |
| Label Checker Filter  | n8n-nodes-base.filter                 | Filter out emails with existing labels | Gmail Trigger         | Give a Label AI Agent        | See Sticky Note3                                                                                     |
| Give a Label AI Agent | @n8n/n8n-nodes-langchain.agent       | AI classification of email        | Label Checker Filter   | Switch                      | See Sticky Note12 (AI agent and output parser docs)                                                 |
| OpenAI Chat Model2    | @n8n/n8n-nodes-langchain.lmChatOpenAi| GPT-4o mini language model backend | Give a Label AI Agent  | Give a Label AI Agent (ai_languageModel) | See Sticky Note12                                                                                     |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI output into structured JSON | Give a Label AI Agent | Give a Label AI Agent (ai_outputParser) | See Sticky Note12                                                                                     |
| Switch                | n8n-nodes-base.switch                 | Routes emails based on AI label   | Give a Label AI Agent  | Work, Personal, Finance, Shopping, travel, Newsletters, Others | See Sticky Note13 (Routing and label application docs)                                              |
| Work                  | n8n-nodes-base.gmail                  | Apply "Work" label                | Switch                 | None                       | See Sticky Note13                                                                                     |
| Personal              | n8n-nodes-base.gmail                  | Apply "Personal" label            | Switch                 | None                       | See Sticky Note13                                                                                     |
| Finance               | n8n-nodes-base.gmail                  | Apply "Finance" label             | Switch                 | None                       | See Sticky Note13                                                                                     |
| Shopping              | n8n-nodes-base.gmail                  | Apply "Shopping" label            | Switch                 | None                       | See Sticky Note13                                                                                     |
| travel                | n8n-nodes-base.gmail                  | Apply "Travel" label              | Switch                 | None                       | See Sticky Note13                                                                                     |
| Newsletters           | n8n-nodes-base.gmail                  | Apply "Newsletters" label         | Switch                 | None                       | See Sticky Note13                                                                                     |
| Others                | n8n-nodes-base.gmail                  | Apply "Others" label              | Switch                 | None                       | See Sticky Note13                                                                                     |
| Sticky Note3          | n8n-nodes-base.stickyNote             | Documentation                    | None                  | None                       | Explains Gmail Trigger and Filter nodes                                                             |
| Sticky Note11         | n8n-nodes-base.stickyNote             | Documentation                    | None                  | None                       | Overview and setup instructions for entire workflow                                                  |
| Sticky Note12         | n8n-nodes-base.stickyNote             | Documentation                    | None                  | None                       | Explains AI Agent and Output Parser nodes                                                           |
| Sticky Note13         | n8n-nodes-base.stickyNote             | Documentation                    | None                  | None                       | Explains Switch node and Gmail label application                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**  
   - Type: `Gmail Trigger`  
   - Set to watch for new emails (poll every minute or desired frequency)  
   - Uncheck "Simple" mode to get full email data  
   - Connect Gmail OAuth2 credentials  
   - Position: starting node  

2. **Add Filter node (Label Checker Filter):**  
   - Type: `Filter`  
   - Condition: Check that the first label ID (`$json.labelIds[0]`) does **not** contain any label (i.e., the email is unlabeled)  
   - Connect input from Gmail Trigger’s main output  

3. **Add Langchain Agent node (Give a Label AI Agent):**  
   - Type: `Langchain Agent`  
   - Set prompt text to include email subject, text body, and sender fields:  
     ```
     Topic: {{ $json.subject }}
     Description: {{ $json.text }}
     Sender: {{ $json.from.text }}
     ```  
   - System message defines role and categories exactly as:  
     - Work, Personal, Finance, Shopping, Travel, Newsletters, Others  
   - Include instructions for JSON output format:  
     ```json
     {
       "email_label": "Category"
     }
     ```  
   - Enable output parser with Structured Output Parser node  
   - Connect input from Label Checker Filter main output  

4. **Add Langchain OpenAI Chat Model node (OpenAI Chat Model2):**  
   - Model: `gpt-4o-mini`  
   - No special options needed  
   - Connect to AI agent node’s language model input  
   - Connect OpenAI API credentials  

5. **Add Structured Output Parser node (Structured Output Parser1):**  
   - JSON schema example:  
     ```json
     {
       "email_label": "business"
     }
     ```  
   - Connect output parser to AI agent’s output parser input  

6. **Add Switch node:**  
   - Add branches for each category string exactly matching AI labels:  
     - Work  
     - Personal  
     - Finance  
     - Shopping  
     - Travel  
     - Newsletters  
     - Others  
   - Condition for each branch: equals `$json.output.email_label`  
   - Connect input from AI agent’s main output  

7. **For each category, add Gmail node to apply label:**  
   For each category branch in the Switch node:  
   - Type: `Gmail` node  
   - Operation: `addLabels`  
   - Message ID: `={{ $('Gmail Trigger').item.json.id }}` (reference original email)  
   - Connect Gmail OAuth2 credentials  
   - Label name must exactly exist in Gmail prior to use  
   - Connect each Gmail node input from corresponding Switch output branch  

8. **Documentation and Notes:**  
   - Optionally add Sticky Note nodes with instructions and links at various positions for user clarity.  

9. **Save and activate workflow.**  
   - Ensure scheduling and polling frequency are adjusted as needed.  
   - Test on unlabeled incoming emails to verify correct labeling.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is perfect for individuals or teams receiving high volumes of email and wanting automatic categorization without coding. All Gmail labels must be pre-created to match AI categories. Adjust the prompt and Switch node branches to fit your specific labeling scheme.                                                                                                  | See Sticky Note11 content in the workflow JSON                                                                                                           |
| Gmail Trigger node official docs: https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger/                                                                                                                                                                                                                                                                    | Gmail Trigger documentation                                                                                                                               |
| Filter node official docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.filter/                                                                                                                                                                                                                                                                                   | Filter node documentation                                                                                                                                  |
| AI Agent node docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/                                                                                                                                                                                                                                                                         | AI Agent node documentation                                                                                                                               |
| Structured Output Parser node docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.outputparserstructured/                                                                                                                                                                                                                                        | Structured Output Parser documentation                                                                                                                    |
| Switch node docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/                                                                                                                                                                                                                                                                                             | Switch node documentation                                                                                                                                 |
| Gmail node label operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/message-operations/#add-label-to-a-message                                                                                                                                                                                                                                         | Gmail node documentation                                                                                                                                  |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.