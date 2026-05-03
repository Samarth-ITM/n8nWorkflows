Slack Workflow Router: AI-Powered Workflow Selection

https://n8nworkflows.xyz/workflows/slack-workflow-router--ai-powered-workflow-selection-11838


# Slack Workflow Router: AI-Powered Workflow Selection

### 1. Workflow Overview

This workflow acts as a **Slack Workflow Router** that uses AI to dynamically select and trigger appropriate n8n workflows based on Slack messages that mention the Slack app. It addresses the Slack limitation of one webhook per Slack app by creating a centralized gateway that routes requests intelligently, enabling multiple workflows to be triggered from a single Slack integration.

**Use Cases:**
- A team or organization wanting to manage multiple Slack automations without creating multiple Slack apps.
- Dynamic routing of Slack app mentions to the most relevant automated workflow.
- Centralized management of workflows using a data table with AI-powered selection logic.

**Logical Blocks:**

- **1.1 Slack Input Reception:** Captures Slack app mentions and extracts the user’s request.
- **1.2 Workflow Data Acquisition:** Retrieves the list of available workflows from a data table.
- **1.3 AI Workflow Selection:** Uses an AI agent with OpenAI to interpret the Slack message and decide which workflow to execute.
- **1.4 Decision Making:** Checks if a matching workflow was found.
- **1.5 Execution and Messaging:** Executes the selected workflow and provides status feedback in Slack.
- **1.6 Error Handling and Notifications:** Sends error or no-match messages to Slack when appropriate.

---

### 2. Block-by-Block Analysis

#### 1.1 Slack Input Reception

**Overview:**  
This block triggers the workflow upon a Slack app mention in a specific channel.

**Nodes Involved:**  
- Slack Trigger  
- Sticky Note1

**Node Details:**

- **Slack Trigger**  
  - **Type:** Slack Trigger node  
  - **Role:** Listens for Slack app mentions in a specific channel (channel ID configured)  
  - **Configuration:** Trigger event set to `app_mention` in channel `C1q1q1q1q1q`  
  - **Inputs:** Triggered by Slack message mentioning the app  
  - **Outputs:** Passes Slack event data downstream  
  - **Edge Cases:** Slack API authentication errors, channel ID misconfiguration, network timeouts  
  - **Sticky Note:** "Trigger on app mention"

---

#### 1.2 Workflow Data Acquisition

**Overview:**  
Fetches all workflows registered in a data table that contains workflow IDs, names, and descriptions.

**Nodes Involved:**  
- Get the list of workflows  
- Sticky Note2

**Node Details:**

- **Get the list of workflows**  
  - **Type:** Data Table Tool node  
  - **Role:** Retrieves all rows from a data table listing workflows  
  - **Configuration:** Operation `get`, `returnAll` set to true, data table ID set to `jtqVNyfaCEqkptz7`  
  - **Inputs:** None (triggered downstream of Slack Trigger)  
  - **Outputs:** Passes all workflow records downstream  
  - **Edge Cases:** Data table access errors, empty tables, permissions issues  
  - **Sticky Note:** Explains required columns in the data table: `workflow_id`, `workflow_name`, `workflow_description`

---

#### 1.3 AI Workflow Selection

**Overview:**  
Uses an AI agent powered by OpenAI GPT-4 to analyze the Slack message text and select the best matching workflow based on the data table.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Sticky Note3

**Node Details:**

- **AI Agent**  
  - **Type:** LangChain Agent node  
  - **Role:** Processes the Slack message text and data table workflows, returns a JSON object identifying the best match  
  - **Configuration:**  
    - Prompt: Extracts the Slack message text (`{{ $json.blocks[0].elements[0].elements[1].text}}`)  
    - Instructions to strictly return JSON with `workflow_id` and `workflow_name` or `not_found` if no match  
    - Uses a system message explaining available workflows and matching criteria  
  - **Inputs:** Slack message data and workflows from the data table  
  - **Outputs:** JSON with selected workflow ID and name  
  - **Edge Cases:**  
    - Parsing errors if AI returns invalid JSON  
    - AI model timeouts or rate limits  
    - Mismatch or no suitable workflow found  
  - **Sub-workflow:** None

- **OpenAI Chat Model**  
  - **Type:** LangChain OpenAI model node  
  - **Role:** Provides the GPT-4.1-mini large language model backend for the AI agent  
  - **Configuration:** Model set to GPT-4.1-mini, credentials connected to OpenAI API  
  - **Inputs:** Receives prompts from AI Agent node  
  - **Outputs:** Returns AI-generated completions  
  - **Edge Cases:** API key invalidation, usage limits, network issues

- **Sticky Note3**  
  - Content: "Find the workflow that matches with the description in the Slack message"

---

#### 1.4 Decision Making

**Overview:**  
Checks if a valid matching workflow was identified by the AI agent.

**Nodes Involved:**  
- Parse JSON output  
- Is there a matching workflow?

**Node Details:**

- **Parse JSON output**  
  - **Type:** Set node  
  - **Role:** Parses the AI agent’s JSON output string into structured JSON fields `workflow_id` and `workflow_name`  
  - **Configuration:** Uses `JSON.parse($json.output)` to extract fields  
  - **Inputs:** Raw string output from AI Agent node  
  - **Outputs:** JSON object with `workflow_id` and `workflow_name` for conditional checks  
  - **Edge Cases:** Parsing failures if AI output is malformed

- **Is there a matching workflow?**  
  - **Type:** If node  
  - **Role:** Checks if `workflow_id` does not contain the string `not_found` (case-insensitive)  
  - **Configuration:** Condition: workflow_id NOT containing `"not_found"`  
  - **Inputs:** Parsed JSON output  
  - **Outputs:**  
    - True branch if match found  
    - False branch if no match  
  - **Edge Cases:** Empty or missing `workflow_id` fields

---

#### 1.5 Execution and Messaging

**Overview:**  
Executes the matched workflow and sends status messages back to Slack.

**Nodes Involved:**  
- Send "executing workflow" message  
- Execute Workflow  
- Send message: success  
- Send message: error occurred  
- Send message: no workflow found  
- Sticky Note4

**Node Details:**

- **Send "executing workflow" message**  
  - **Type:** Slack node  
  - **Role:** Sends a message to the Slack channel indicating which workflow is being executed  
  - **Configuration:** Message text includes `workflow_name` and `workflow_id`, channel ID from Slack Trigger node  
  - **Inputs:** True branch from `Is there a matching workflow?`  
  - **Outputs:** Triggers Execute Workflow node  
  - **Edge Cases:** Slack API errors, missing channel ID

- **Execute Workflow**  
  - **Type:** Execute Workflow node  
  - **Role:** Executes the workflow identified by the AI agent  
  - **Configuration:**  
    - Workflow ID taken from parsed JSON `workflow_id`  
    - Waits for sub-workflow to finish (`waitForSubWorkflow`=true)  
    - Empty inputs mapped (`workflowInputs` empty)  
  - **Inputs:** From "Send executing workflow" message node  
  - **Outputs:**  
    - Success triggers "Send message: success"  
    - Error triggers "Send message: error occurred"  
  - **Edge Cases:** Workflow not found, execution errors, input format issues

- **Send message: success**  
  - **Type:** Slack node  
  - **Role:** Sends confirmation message upon successful execution  
  - **Configuration:** Fixed text "Successfully executed workflow", posts to triggering channel  
  - **Inputs:** Success output of Execute Workflow node  
  - **Outputs:** None  
  - **Edge Cases:** Slack API issues

- **Send message: error occurred**  
  - **Type:** Slack node  
  - **Role:** Sends error notification if workflow execution fails  
  - **Configuration:** Fixed text "An error occurred while executing the workflow", sends to Slack channel  
  - **Inputs:** Error output of Execute Workflow node  
  - **Outputs:** None  
  - **Edge Cases:** Slack API issues

- **Send message: no workflow found**  
  - **Type:** Slack node  
  - **Role:** Notifies the channel when no matching workflow is found by AI  
  - **Configuration:** Fixed text "Sorry I could not find a workflow that matches with your request."  
  - **Inputs:** False branch of `Is there a matching workflow?`  
  - **Outputs:** None  
  - **Edge Cases:** Slack API issues

- **Sticky Note4**  
  - Content: "Execute the matching workflow"

---

#### 1.6 Documentation and Project Notes

**Nodes Involved:**  
- Sticky Note5

**Node Details:**

- **Sticky Note5**  
  - **Content:** Comprehensive project description including problem statement, how the workflow works, key features, and setup instructions.  
  - **Purpose:** Provides context and instructions for users and maintainers.  
  - **Position:** Visually covers the entire workflow for easy reference.

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                              | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                                                   |
|-------------------------------|-----------------------------------|----------------------------------------------|---------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Slack Trigger                 | Slack Trigger                     | Entry point on Slack app mention             | —                         | AI Agent                      | Trigger on app mention                                                                                                         |
| AI Agent                     | LangChain Agent                   | Selects matching workflow via AI              | Slack Trigger, Get the list of workflows | Parse JSON output              | Find the workflow that matches with the description in the Slack message                                                       |
| OpenAI Chat Model            | LangChain OpenAI Model            | Provides AI language model (GPT-4.1-mini)     | AI Agent                   | AI Agent                      |                                                                                                                               |
| Get the list of workflows    | Data Table Tool                  | Retrieves registered workflows from data table | —                         | AI Agent                      | Create a data table with columns: workflow_id, workflow_name, workflow_description                                              |
| Parse JSON output            | Set                              | Parses AI Agent JSON string output            | AI Agent                   | Is there a matching workflow?  |                                                                                                                               |
| Is there a matching workflow?| If                               | Checks if a valid workflow was found          | Parse JSON output          | Send "executing workflow" message, Send message: no workflow found |                                                                                                                               |
| Send "executing workflow" message | Slack                         | Notifies Slack that workflow execution starts | Is there a matching workflow? (true) | Execute Workflow               | Execute the matching workflow                                                                                                  |
| Execute Workflow             | Execute Workflow                  | Runs the selected workflow                     | Send "executing workflow" message | Send message: success, Send message: error occurred |                                                                                                                               |
| Send message: success        | Slack                            | Sends success confirmation to Slack           | Execute Workflow (success) | —                             |                                                                                                                               |
| Send message: error occurred | Slack                            | Sends error message to Slack                    | Execute Workflow (error)   | —                             |                                                                                                                               |
| Send message: no workflow found | Slack                         | Notifies Slack when no workflow matches       | Is there a matching workflow? (false) | —                             |                                                                                                                               |
| Sticky Note1                 | Sticky Note                      | Visual note                                    | —                         | —                             | Trigger on app mention                                                                                                         |
| Sticky Note2                 | Sticky Note                      | Visual note                                    | —                         | —                             | Create a data table with workflow_id, workflow_name, workflow_description                                                     |
| Sticky Note3                 | Sticky Note                      | Visual note                                    | —                         | —                             | Find the workflow that matches with the description in the Slack message                                                      |
| Sticky Note4                 | Sticky Note                      | Visual note                                    | —                         | —                             | Execute the matching workflow                                                                                                  |
| Sticky Note5                 | Sticky Note                      | Visual note                                    | —                         | —                             | Comprehensive project overview, problem, solution, setup instructions                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node**  
   - Type: Slack Trigger  
   - Trigger: `app_mention`  
   - Channel ID: Set to the Slack channel where mentions should be detected (e.g., `C1q1q1q1q1q`)  
   - Credentials: Connect your Slack API credentials

2. **Create Data Table for Workflows**  
   - In your n8n project, create a data table named e.g. "workflows"  
   - Add columns:  
     - `workflow_id` (string) - The unique workflow identifier  
     - `workflow_name` (string) - The workflow’s name  
     - `workflow_description` (string) - Description of what the workflow does  
   - Populate this table with your workflows' info

3. **Add Data Table Tool Node**  
   - Type: Data Table Tool  
   - Operation: `get`  
   - Return All: Yes  
   - Data Table ID: Select your workflows data table  
   - No credentials needed for data table access

4. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: GPT-4.1-mini (or similar GPT-4 model)  
   - Credentials: Connect your OpenAI API credentials

5. **Add LangChain AI Agent Node**  
   - Type: LangChain Agent  
   - Text Prompt:  
     ```
     Find a matching workflow for the request below. 
     ---
     {{ $json.blocks[0].elements[0].elements[1].text}}
     ---
     If you find a match, strictly return a valid json object like the one in the example below, with the id and the name of the workflow:

     {"workflow_id"="workflow_id_value","workflow_name"="Workflow Name"}

     If you do not find a match, return the following json as it is:

     {"workflow_id"="not_found","workflow_name"="not_found"}

     Do not return anything else.
     ```
   - System Message: Explain the available workflows and their fields to the AI agent, emphasizing strict JSON output  
   - Link AI Agent to OpenAI Chat Model node as its language model  
   - Feed Slack Trigger output and Data Table Tool output into this node as input (Slack message text and workflow list)

6. **Add Set Node to Parse JSON output**  
   - Type: Set  
   - Use expression: `={{JSON.parse($json.output).workflow_id}}` for `workflow_id`  
   - Use expression: `={{JSON.parse($json.output).workflow_name}}` for `workflow_name`

7. **Add If Node to Check Matching Workflow**  
   - Condition: Check if `workflow_id` does NOT contain string `not_found` (case-insensitive)  
   - True branch: Matching workflow found  
   - False branch: No matching workflow

8. **Add Slack Node to Send "Executing workflow" Message**  
   - Text: `Executing workflow: {{ $json.workflow_name }} ({{ $json.workflow_id }})`  
   - Channel: Use expression to get channel from Slack Trigger (`{{$node["Slack Trigger"].json["channel"]}}`)  
   - Credentials: Slack API

9. **Add Execute Workflow Node**  
   - Workflow ID: Use expression `{{$node["Parse JSON output"].json["workflow_id"]}}`  
   - Wait for sub-workflow to finish: Enabled  
   - Inputs: Empty / default

10. **Add Slack Nodes for Success and Error Messages**  
    - Success Node: Text `Successfully executed workflow`  
    - Error Node: Text `An error occurred while executing the workflow`  
    - Both post to Slack channel from Slack Trigger node  
    - Connect Execute Workflow node outputs accordingly

11. **Add Slack Node for "No workflow found" Message**  
    - Text: `Sorry I could not find a workflow that matches with your request.`  
    - Channel: From Slack Trigger node

12. **Connect Nodes According to Logic:**  
    - Slack Trigger → AI Agent  
    - Get the list of workflows → AI Agent (as tool input)  
    - AI Agent → Parse JSON output  
    - Parse JSON output → If Node  
    - If True → Send "Executing workflow" message → Execute Workflow → Success and Error Slack messages  
    - If False → Send "No workflow found" message

13. **Add Sticky Notes for Documentation** (optional but recommended)  
    - Add notes for trigger explanation, data table setup, AI matching, workflow execution, and project overview for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Slack only allows one webhook per Slack app, limiting direct multiple workflow triggers. This workflow solves that by centralizing Slack mentions and routing to multiple workflows using AI.                                                                                                                                                                                                                                                                                                                                              | Sticky Note5 content                                                                                     |
| Create a data table with columns: `workflow_id`, `workflow_name`, and `workflow_description` to store your workflows.                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note2 content                                                                                     |
| The AI agent uses GPT-4.1-mini model to match Slack messages with workflows based on names and descriptions.                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note3 content                                                                                     |
| This approach allows you to maintain a single Slack app and trigger many workflows dynamically, scalable and maintainable.                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note5 content                                                                                     |

---

This document provides a detailed understanding, node-by-node breakdown, and stepwise reproduction instructions for the Slack Workflow Router workflow, enabling users and AI systems to modify, maintain, or extend the solution confidently.