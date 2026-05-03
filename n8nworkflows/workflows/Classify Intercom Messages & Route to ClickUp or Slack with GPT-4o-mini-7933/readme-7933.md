Classify Intercom Messages & Route to ClickUp or Slack with GPT-4o-mini

https://n8nworkflows.xyz/workflows/classify-intercom-messages---route-to-clickup-or-slack-with-gpt-4o-mini-7933


# Classify Intercom Messages & Route to ClickUp or Slack with GPT-4o-mini

### 1. Workflow Overview

This workflow automates the classification and routing of incoming customer messages from Intercom using AI-powered analysis with GPT-4o-mini. It is designed for customer support, product management, and sales teams to streamline issue handling by automatically categorizing conversations into Support, Product, Sales, or Other. Based on the classification, it either creates tasks in ClickUp or sends notifications to Slack, ensuring timely and relevant follow-up actions.

The workflow is logically divided into four main blocks:

- **1.1 Webhook Intake**: Receives incoming Intercom conversation data via webhook.
- **1.2 AI Classification Engine**: Sends the conversation to an AI prompt using GPT-4o-mini to classify and analyze sentiment, urgency, and tags.
- **1.3 Conditional Routing**: Uses conditional logic to route conversations based on AI-determined categories.
- **1.4 Action Handling**: Creates ClickUp tasks for Support and Product inquiries or sends Slack notifications for Sales inquiries.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Intake

- **Overview**:  
  This block receives new conversation data from Intercom via a webhook HTTP POST request. It acts as the entry point for the workflow, triggering downstream processing.

- **Nodes Involved**:  
  - 📨 Intercom Webhook  
  - Sticky Note (📝 descriptive)

- **Node Details**:

  - **📨 Intercom Webhook**  
    - *Type*: Webhook  
    - *Technical Role*: Entry node that listens for incoming HTTP POST requests carrying Intercom conversation JSON data.  
    - *Configuration*:  
      - HTTP Method: POST  
      - Path: Custom webhook path, e.g., "your-webhook-path-here" (to be set per environment)  
    - *Input/Output*: No input; outputs raw JSON payload from Intercom webhook.  
    - *Version*: 1  
    - *Failure Cases*: Invalid or missing webhook data, incorrect HTTP method, network issues.  
    - *Notes*: Must be publicly accessible to Intercom for webhook triggering.

  - **Sticky Note**  
    - Content summarizes purpose: receives conversation JSON with customer name, email, and message content.

#### 1.2 AI Classification Engine

- **Overview**:  
  This block sends the raw conversation data to an AI model (GPT-4o-mini) with a detailed prompt instructing it to classify the conversation into categories, sentiment, urgency, reasoning, and tags. The AI response is then parsed and combined with original conversation details for downstream use.

- **Nodes Involved**:  
  - 🧠 Classifier – AI prompt  
  - GPT model (gpt-4o-mini)  
  - 🧮 Process Classification  
  - Sticky Note (📝 descriptive)

- **Node Details**:

  - **🧠 Classifier – AI prompt**  
    - *Type*: LangChain Agent Node  
    - *Technical Role*: Sends a custom prompt to the AI model to classify the Intercom conversation.  
    - *Configuration*:  
      - Prompt includes instructions to output ONLY a valid JSON with fields: category, sentiment, urgency, reasoning, tags.  
      - Input expression: Injects entire conversation JSON from webhook node `{{ $json["data"]["item"] }}` into the prompt.  
    - *Input*: Output of 📨 Intercom Webhook  
    - *Output*: AI-generated JSON classification (in string format)  
    - *Version*: 2  
    - *Failure Cases*: AI returns invalid JSON, timeout, API errors, prompt misconfiguration.

  - **GPT model (gpt-4o-mini)**  
    - *Type*: LangChain OpenAI Chat Model  
    - *Technical Role*: Provides LLM backend for AI prompt node.  
    - *Configuration*: Model set to `gpt-4o-mini`.  
    - *Credentials*: Requires OpenAI API key (OAuth or API token).  
    - *Version*: 1.2  
    - *Failure Cases*: API authentication errors, rate limits, network failures.

  - **🧮 Process Classification**  
    - *Type*: Code (JavaScript)  
    - *Technical Role*: Parses AI JSON response, merges classification with original conversation details, and formats task metadata (title, description, tags, priority).  
    - *Configuration*:  
      - Parses AI response from `choices[0].message.content`  
      - Extracts conversation fields: id, subject, message, customer name/email, created_at  
      - Prepares ClickUp task title and description strings with embedded classification data.  
    - *Input*: Output from 🧠 Classifier – AI prompt  
    - *Output*: Structured JSON ready for downstream routing  
    - *Version*: 2  
    - *Failure Cases*: JSON parse errors if AI response invalid, missing fields in conversation data.

  - **Sticky Note**  
    - Describes AI classification logic and output fields.

#### 1.3 Conditional Routing

- **Overview**:  
  This block routes the processed classification data based on the `category` field, branching into Support, Product, or Sales paths.

- **Nodes Involved**:  
  - 🛠️ Is Support Request? (If node)  
  - 📦 Is Product Request? (If node)  
  - 💼 Is Sales Request? (If node)  
  - Sticky Note (📝 descriptive)

- **Node Details**:

  - **🛠️ Is Support Request?**  
    - *Type*: If (Boolean condition)  
    - *Technical Role*: Checks if `category` equals `"Support"`.  
    - *Configuration*: Strict string equality, case sensitive, leftValue set dynamically to `{{ $json.category }}`  
    - *Input*: Output from 🧮 Process Classification  
    - *Output*: True branch flows to creating support task  
    - *Version*: 2  
    - *Failure Cases*: Missing or malformed category field.

  - **📦 Is Product Request?**  
    - *Type*: If  
    - *Technical Role*: Checks if `category` equals `"Product"`.  
    - *Input*: Output from 🧮 Process Classification  
    - *Output*: True branch to product task creation  
    - *Version*: 2

  - **💼 Is Sales Request?**  
    - *Type*: If  
    - *Technical Role*: Checks if `category` equals `"Sales"`.  
    - *Input*: Output from 🧮 Process Classification  
    - *Output*: True branch to Slack notification node  
    - *Version*: 2

  - **Sticky Note**  
    - Explains routing logic based on AI category.

#### 1.4 Action Handling

- **Overview**:  
  Based on the routed category, this block either creates a task in ClickUp for Support or Product inquiries or sends a Slack notification for Sales inquiries with detailed AI classification context.

- **Nodes Involved**:  
  - 🧾 Create Support Task (ClickUp node)  
  - 🛍️ Create Product Task (ClickUp node)  
  - 📣 Slack – Notify Sales team (Slack node)  
  - Sticky Note (📝 descriptive)

- **Node Details**:

  - **🧾 Create Support Task**  
    - *Type*: ClickUp task creation  
    - *Technical Role*: Creates a new task in ClickUp for Support category conversations with metadata.  
    - *Configuration*:  
      - List, Team, Space, Folder IDs/names must be configured for target workspace  
      - Task name set from `task_title` (e.g., “[SUPPORT] Subject”)  
      - Additional fields: tags (comma-separated), status, priority (mapped from urgency), assignees  
      - Authentication via OAuth2 ClickUp credentials  
    - *Input*: True branch output from 🛠️ Is Support Request?  
    - *Output*: Task creation response  
    - *Version*: 1  
    - *Failure Cases*: Authentication errors, invalid IDs, API limits, missing required fields.

  - **🛍️ Create Product Task**  
    - *Type*: ClickUp task creation  
    - *Technical Role*: Same as Support task but for Product category.  
    - *Input*: True branch output from 📦 Is Product Request?  
    - *Version*: 1

  - **📣 Slack – Notify Sales team**  
    - *Type*: Slack node  
    - *Technical Role*: Sends a formatted Slack message to a designated channel notifying sales team of a new inquiry.  
    - *Configuration*:  
      - Message text includes customer info, subject, sentiment, urgency, original message, AI reasoning, conversation ID  
      - Channel specified by ID  
      - Authenticated with Slack API credentials  
    - *Input*: True branch output from 💼 Is Sales Request?  
    - *Version*: 2.3  
    - *Failure Cases*: Invalid Slack credentials, channel ID errors, message formatting issues.

  - **Sticky Note**  
    - Summarizes actions taken for each category.

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                   | Input Node(s)              | Output Node(s)                                  | Sticky Note                                               |
|----------------------------|------------------------------|---------------------------------|----------------------------|------------------------------------------------|-----------------------------------------------------------|
| 📨 Intercom Webhook         | Webhook                      | Receive Intercom conversation   | —                          | 🧠 Classifier – AI prompt                       | Receives incoming conversation data from Intercom.       |
| 🧠 Classifier – AI prompt   | LangChain Agent              | Classify conversation with AI   | 📨 Intercom Webhook         | 🧮 Process Classification                       | AI classifies conversation into structured JSON.         |
| GPT model (gpt-4o-mini)    | LangChain OpenAI Chat Model  | Provide GPT-4o-mini model       | — (linked as AI model)      | 🧠 Classifier – AI prompt (ai_languageModel)   | Backend LLM for AI classification.                        |
| 🧮 Process Classification   | Code (JavaScript)            | Parse AI response and prepare data | 🧠 Classifier – AI prompt   | 🛠️ Is Support Request?, 📦 Is Product Request?, 💼 Is Sales Request? | Parses AI JSON and formats task metadata.                 |
| 🛠️ Is Support Request?       | If (conditional)             | Check if category is Support    | 🧮 Process Classification   | 🧾 Create Support Task                           | Routes Support category conversations.                    |
| 📦 Is Product Request?       | If (conditional)             | Check if category is Product    | 🧮 Process Classification   | 🛍️ Create Product Task                           | Routes Product category conversations.                    |
| 💼 Is Sales Request?         | If (conditional)             | Check if category is Sales      | 🧮 Process Classification   | 📣 Slack – Notify Sales team                     | Routes Sales category conversations.                      |
| 🧾 Create Support Task       | ClickUp                      | Create Support task in ClickUp  | 🛠️ Is Support Request?       | —                                              | Creates ClickUp task with classification data.            |
| 🛍️ Create Product Task       | ClickUp                      | Create Product task in ClickUp  | 📦 Is Product Request?       | —                                              | Creates ClickUp task with classification data.            |
| 📣 Slack – Notify Sales team | Slack                        | Notify Sales team via Slack     | 💼 Is Sales Request?         | —                                              | Sends Slack notification with detailed inquiry info.      |
| Sticky Note                 | Sticky Note                  | Documentation & description     | —                          | —                                              | See respective block descriptions above.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Add a "Webhook" node named `📨 Intercom Webhook`.  
   - Set HTTP Method to `POST`.  
   - Define a unique path for the webhook, e.g., `/intercom-webhook`.  
   - Save and activate the webhook to receive Intercom messages.

2. **Create AI Prompt Node:**  
   - Add a LangChain Agent node named `🧠 Classifier – AI prompt`.  
   - Set prompt type to "define" and enter the detailed prompt instructing AI to classify the conversation into JSON fields: category, sentiment, urgency, reasoning, tags.  
   - Use expression to inject conversation JSON: `{{ $json["data"]["item"] }}`.  
   - Connect this node’s AI language model input to the GPT OpenAI node (next step).

3. **Create GPT Model Node:**  
   - Add a LangChain OpenAI Chat Model node named `GPT model (gpt-4o-mini)`.  
   - Set model to `gpt-4o-mini`.  
   - Configure OpenAI API credentials (API key or OAuth).  
   - Link this node as the AI model for the LangChain Agent node.

4. **Create Code Node to Process AI Output:**  
   - Add a "Code" node named `🧮 Process Classification`.  
   - Use JavaScript code to parse AI JSON response from previous node.  
   - Extract conversation details (id, subject, message, customer name/email).  
   - Generate task title and description strings including AI classification fields.  
   - Connect input from AI prompt node.

5. **Create Conditional If Nodes for Routing:**  
   - Add three "If" nodes named:  
     - `🛠️ Is Support Request?` — condition: `category === "Support"`  
     - `📦 Is Product Request?` — condition: `category === "Product"`  
     - `💼 Is Sales Request?` — condition: `category === "Sales"`  
   - Connect all three nodes from the output of the code node.

6. **Create ClickUp Task Nodes:**  
   - Add two "ClickUp" nodes:  
     - `🧾 Create Support Task`: Configure with your ClickUp list, team, space, and folder IDs.  
       - Set task name to `={{ $json.task_title }}`.  
       - Map tags, status, priority (using urgency), and assignees.  
       - Authenticate with ClickUp OAuth2 credentials.  
       - Connect input from `🛠️ Is Support Request?` true branch.  
     - `🛍️ Create Product Task`: Same configuration as Support Task node.  
       - Connect input from `📦 Is Product Request?` true branch.

7. **Create Slack Notification Node:**  
   - Add a "Slack" node named `📣 Slack – Notify Sales team`.  
   - Configure Slack API credentials.  
   - Set message text using expressions to include customer info, sentiment, urgency, original message, AI reasoning, and conversation ID.  
   - Select the target Slack channel by ID.  
   - Connect input from `💼 Is Sales Request?` true branch.

8. **Connect Nodes in Order:**  
   - `📨 Intercom Webhook` → `🧠 Classifier – AI prompt` → `🧮 Process Classification` → conditional If nodes → respective ClickUp or Slack nodes.

9. **Add Sticky Notes (Optional):**  
   - Document each block with sticky notes describing purpose and logic.

10. **Test the Workflow:**  
    - Trigger the webhook with sample Intercom JSON data.  
    - Verify AI classification output, routing decisions, and task/notification creation.  
    - Handle errors such as invalid JSON from AI or API failures.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                               |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The AI prompt strictly requires outputting ONLY valid JSON to ensure parseability.              | Critical for the code node to work correctly.                 |
| Ensure that the ClickUp OAuth2 credentials have write permissions for task creation.             | ClickUp API documentation: https://clickup.com/api            |
| Slack node requires correct channel ID and valid Slack API token with chat:write scope.          | Slack API docs: https://api.slack.com/messaging/sending       |
| Webhook URL must be publicly accessible and registered in Intercom to receive real messages.     | Intercom webhook setup: https://developers.intercom.com/webhooks/ |
| GPT-4o-mini is a lightweight model; if higher accuracy is needed, consider upgrading model.      | OpenAI models: https://platform.openai.com/docs/models        |

---

**Disclaimer:** The provided text derives exclusively from an n8n workflow automation and strictly adheres to content policies. It contains no illegal or protected elements. Only lawful, public data are processed.