Simple Expense Tracker with n8n Chat, AI Agent and Google Sheets

https://n8nworkflows.xyz/workflows/simple-expense-tracker-with-n8n-chat--ai-agent-and-google-sheets-2819


# Simple Expense Tracker with n8n Chat, AI Agent and Google Sheets

### 1. Workflow Overview

This workflow enables users to conveniently add expense entries via simple chat messages using n8n’s AI-powered chat capabilities. The core functionality is to parse a natural language expense message into structured JSON, then save this data as a new row in a Google Sheet. The workflow is designed for personal or small business expense tracking with minimal manual input.

The workflow is logically divided into the following blocks:

- **1.1 Chat Input Reception:** Listens for incoming chat messages containing expense data.
- **1.2 AI Agent Processing:** Uses an AI agent to interpret the chat message and invoke a sub-workflow to parse and save the expense.
- **1.3 Expense Parsing and Saving (Sub-Workflow):** Parses the expense text into structured JSON and appends it as a new row in Google Sheets.
- **1.4 Supporting Nodes:** Includes memory management for conversation context, OpenAI model nodes, and user guidance notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Input Reception

- **Overview:**  
  This block captures incoming chat messages from users that contain expense information in free text form.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Webhook trigger node that listens for chat messages.  
    - Configuration: Uses a webhook ID to receive chat inputs. No additional options configured.  
    - Inputs: External chat message via webhook.  
    - Outputs: Passes chat message data to the AI Agent node.  
    - Edge cases: Webhook connectivity issues, malformed chat messages, or unsupported message formats could cause failures.

#### 2.2 AI Agent Processing

- **Overview:**  
  This block uses an AI agent to process the chat message, parse it, and trigger the sub-workflow that saves the expense data.

- **Nodes Involved:**  
  - AI Agent  
  - Parse msg and save to Sheets (sub-workflow executor)  
  - OpenAI Chat Model  
  - Window Buffer Memory

- **Node Details:**  
  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Acts as the main AI processor, orchestrating the parsing and saving of expense data.  
    - Configuration:  
      - Max iterations: 3 (limits AI response loops)  
      - System message: Defines the AI persona as a helpful accountant who uses the "save to db" tool to save expenses.  
    - Inputs: Receives chat messages from "When chat message received".  
    - Outputs: Calls the "Parse msg and save to Sheets" tool workflow node.  
    - Edge cases: AI model errors, exceeding max iterations, or tool invocation failures.

  - **Parse msg and save to Sheets**  
    - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
    - Role: Invokes the sub-workflow responsible for parsing and saving the expense.  
    - Configuration:  
      - Workflow ID: Points to the same workflow (self-reference) to enable sub-workflow execution.  
      - Workflow inputs: Passes the chat input as `input1`.  
    - Inputs: Receives chat input from AI Agent.  
    - Outputs: Returns parsed and saved expense data back to AI Agent.  
    - Edge cases: Incorrect workflow selection, input mapping errors, or sub-workflow failures.

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides the language model backend for the AI Agent.  
    - Configuration: Uses OpenAI API credentials.  
    - Inputs: Receives prompts from AI Agent or parser nodes.  
    - Outputs: Returns AI-generated text or JSON.  
    - Edge cases: API rate limits, authentication errors, or network timeouts.

  - **Window Buffer Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains recent conversation context for the AI Agent to improve understanding.  
    - Configuration: Default buffer window settings.  
    - Inputs/Outputs: Connected to AI Agent for memory management.  
    - Edge cases: Memory overflow or context loss in long conversations.

#### 2.3 Expense Parsing and Saving (Sub-Workflow)

- **Overview:**  
  This block parses the raw expense text into structured JSON fields and appends the data as a new row in a Google Sheet.

- **Nodes Involved:**  
  - Workflow Input Trigger  
  - Expense text to JSON parser  
  - Save expense into Google Sheets  
  - OpenAI Chat Model1

- **Node Details:**  
  - **Workflow Input Trigger**  
    - Type: `n8n-nodes-base.executeWorkflowTrigger`  
    - Role: Entry point for the sub-workflow execution when called by the toolWorkflow node.  
    - Configuration: Accepts `input1` as the expense text string.  
    - Inputs: Receives input from the toolWorkflow node.  
    - Outputs: Passes input to the parser node.  
    - Edge cases: Missing or malformed input parameters.

  - **Expense text to JSON parser**  
    - Type: `@n8n/n8n-nodes-langchain.informationExtractor`  
    - Role: Uses AI to extract structured data (cost, description, date) from the expense text.  
    - Configuration:  
      - Input text: Expression `=convert expense to JSON: {{ $json.input1 }}`  
      - Attributes defined:  
        - cost (number, required)  
        - descr (string, required)  
        - date (date, optional, UTC format)  
    - Inputs: Receives raw expense text from Workflow Input Trigger.  
    - Outputs: Produces JSON with extracted fields.  
    - Edge cases: Parsing errors, missing required fields, ambiguous date formats.

  - **Save expense into Google Sheets**  
    - Type: `n8n-nodes-base.googleSheets`  
    - Role: Appends the parsed expense data as a new row in a specified Google Sheet.  
    - Configuration:  
      - Operation: Append row  
      - Columns mapped:  
        - msg: original message text  
        - cost: parsed cost  
        - date: parsed date or current date if missing  
        - descr: parsed description  
      - Document ID: User must link to their copied Google Sheet.  
      - Credentials: Google Sheets OAuth2 account required.  
    - Inputs: Receives parsed JSON from the parser node.  
    - Outputs: Confirms row insertion.  
    - Edge cases: Authentication errors, sheet access issues, invalid data types.

  - **OpenAI Chat Model1**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Supports the information extractor node with OpenAI API calls.  
    - Configuration: Uses same OpenAI API credentials as other model nodes.  
    - Inputs/Outputs: Connected to the parser node.  
    - Edge cases: Same as other OpenAI nodes.

#### 2.4 Supporting Nodes

- **Sticky Note**  
  - Type: `n8n-nodes-base.stickyNote`  
  - Role: Provides user instructions and installation notes within the workflow canvas.  
  - Content: Detailed setup instructions for Google Sheets, sub-workflow linking, and usage examples.  
  - Position: Left side for visibility.  
  - No inputs or outputs.

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                          | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                      |
|-----------------------------|---------------------------------------------|----------------------------------------|--------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received  | @n8n/n8n-nodes-langchain.chatTrigger        | Receives chat messages                  | External webhook               | AI Agent                       |                                                                                                |
| AI Agent                    | @n8n/n8n-nodes-langchain.agent              | Processes chat input, orchestrates AI  | When chat message received     | Parse msg and save to Sheets   |                                                                                                |
| Parse msg and save to Sheets| @n8n/n8n-nodes-langchain.toolWorkflow       | Invokes sub-workflow to parse and save | AI Agent                      | AI Agent                      | Make sure that this SAME workflow is chosen in the Workflow dropdown!                           |
| OpenAI Chat Model           | @n8n/n8n-nodes-langchain.lmChatOpenAi       | Provides AI language model backend     | AI Agent / Expense parser      | AI Agent / Expense parser      |                                                                                                |
| Window Buffer Memory        | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context         | AI Agent                      | AI Agent                      |                                                                                                |
| Workflow Input Trigger      | n8n-nodes-base.executeWorkflowTrigger       | Entry point for sub-workflow execution | Parse msg and save to Sheets   | Expense text to JSON parser    |                                                                                                |
| Expense text to JSON parser | @n8n/n8n-nodes-langchain.informationExtractor| Parses expense text to JSON             | Workflow Input Trigger         | Save expense into Google Sheets|                                                                                                |
| Save expense into Google Sheets | n8n-nodes-base.googleSheets               | Appends expense data to Google Sheet   | Expense text to JSON parser    | None                         | Do not forget to Clone the Google Sheet and re-link this node to your sheet!                   |
| OpenAI Chat Model1          | @n8n/n8n-nodes-langchain.lmChatOpenAi       | Supports parsing with OpenAI API       | Expense text to JSON parser    | Expense text to JSON parser    |                                                                                                |
| Sticky Note                 | n8n-nodes-base.stickyNote                    | Provides user instructions             | None                         | None                         | ## Save your expenses via chat message. LLM will parse your message to structured JSON and save as a new row into Google Sheet. Installation instructions included. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook ID (auto-generated) to receive chat messages.

2. **Create "AI Agent" node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set max iterations to 3.  
   - Set system message: "You are a helpful accountant. Use save to db tool to save expense message to DB. respond with \"Your expense saved, here is the output of save sub-workflow:[data]\""  
   - Connect input from "When chat message received".

3. **Create "Parse msg and save to Sheets" node**  
   - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Select the current workflow itself in the workflow dropdown (self-reference).  
   - Set workflow input mapping: pass chat input as `input1` from AI Agent.  
   - Connect input from AI Agent.

4. **Create "OpenAI Chat Model" node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Set credentials: link your OpenAI API account.  
   - Connect as AI language model for AI Agent.

5. **Create "Window Buffer Memory" node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Connect to AI Agent for memory context.

6. **Create "Workflow Input Trigger" node**  
   - Type: `n8n-nodes-base.executeWorkflowTrigger`  
   - Configure input parameter `input1` (string).  
   - This node acts as the entry point for the sub-workflow.

7. **Create "Expense text to JSON parser" node**  
   - Type: `@n8n/n8n-nodes-langchain.informationExtractor`  
   - Set input text expression: `=convert expense to JSON: {{ $json.input1 }}`  
   - Define attributes:  
     - cost (number, required)  
     - descr (string, required)  
     - date (date, optional)  
   - Connect input from "Workflow Input Trigger".

8. **Create "OpenAI Chat Model1" node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Use same OpenAI credentials.  
   - Connect as AI model for the parser node.

9. **Create "Save expense into Google Sheets" node**  
   - Type: `n8n-nodes-base.googleSheets`  
   - Operation: Append row  
   - Map columns:  
     - msg: `={{ $('Workflow Input Trigger').item.json.input1 }}`  
     - cost: `={{ $json.output.cost }}`  
     - date: `={{ $json.output.date ? $json.output.date : $now }}`  
     - descr: `={{ $json.output.descr }}`  
   - Link to your copied Google Sheet document ID.  
   - Set Google Sheets OAuth2 credentials.  
   - Connect input from "Expense text to JSON parser".

10. **Connect nodes as per the logical flow:**  
    - "When chat message received" → "AI Agent"  
    - "AI Agent" → "Parse msg and save to Sheets"  
    - "Parse msg and save to Sheets" → "Workflow Input Trigger" (sub-workflow)  
    - "Workflow Input Trigger" → "Expense text to JSON parser"  
    - "Expense text to JSON parser" → "Save expense into Google Sheets"  
    - AI model nodes connected to AI Agent and parser nodes accordingly.  
    - "Window Buffer Memory" connected to AI Agent.

11. **Add a Sticky Note node** with installation and usage instructions for user reference.

12. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                      | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Clone this Google Sheet to use with the workflow: https://docs.google.com/spreadsheets/d/1D0r3tun7LF7Ypb21CmbTKEtn76WE-kaHvBCM5NdgiPU/edit?gid=0#gid=0                                                                                         | Google Sheets template for expense tracking                                                                          |
| Make sure the sub-workflow executor node ("Parse msg and save to Sheets") references this SAME workflow to enable proper sub-workflow execution.                                                                                                | Important for sub-workflow invocation                                                                                 |
| Send chat messages in the format: "car wash; 59.3 usd; 25 jan 2024" to add expenses via chat.                                                                                                                                                    | Usage instruction                                                                                                     |
| Response example: Your expense saved, here is the output of save sub-workflow:{"cost":59.3,"descr":"car wash","date":"2024-01-25","msg":"car wash; 59.3 usd; 25 jan 2024"}                                                                       | Confirmation message returned by AI agent                                                                             |
| Google Sheets node requires OAuth2 credentials with access to the chosen spreadsheet.                                                                                                                                                            | Credential setup                                                                                                       |
| OpenAI API credentials are required for all AI model nodes.                                                                                                                                                                                      | Credential setup                                                                                                       |

---

This documentation fully describes the workflow structure, node configurations, and usage instructions to enable reproduction, modification, and troubleshooting.