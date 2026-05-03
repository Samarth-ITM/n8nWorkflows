Route Jira Tasks to Experts Using Google Sheets and GPT-4o-mini

https://n8nworkflows.xyz/workflows/route-jira-tasks-to-experts-using-google-sheets-and-gpt-4o-mini-9682


# Route Jira Tasks to Experts Using Google Sheets and GPT-4o-mini

### 1. Workflow Overview

This workflow automates the assignment of Jira issues based on new task entries added to a Google Sheet. It targets teams managing Jira tasks and bugs, aiming to quickly route new issues to the most suitable expert using a Google Sheets roster and AI-powered expertise matching.  

The workflow logic is divided into the following blocks:  
- **1.1 Input Reception:** Detects new rows added to a Google Sheet containing task details.  
- **1.2 AI Processing:** Uses an AI agent powered by Azure OpenAI GPT-4o-mini to analyze the task and match it with the best employee from the expertise roster in Google Sheets.  
- **1.3 Output Parsing:** Parses the structured output from the AI agent, extracting key assignment details.  
- **1.4 Routing & Issue Creation:** Routes the processed data based on issue type (bug or task) and creates the corresponding Jira issue with the appropriate assignee.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens to a Google Sheet for any new rows added, representing new tasks to be assigned. It extracts the “Task Name” and “Related Area” from each new row to trigger the assignment process.

- **Nodes Involved:**  
  - Google Sheets Trigger  

- **Node Details:**  

  - **Google Sheets Trigger**  
    - Type: Trigger node for Google Sheets  
    - Configuration:  
      - Event: rowAdded (triggers when a new row is added)  
      - Poll frequency: every minute  
      - Document ID: points to the Google Sheet containing new tasks ("new tasks for the week")  
      - Sheet name: Sheet1 (gid=0)  
    - Inputs: None (trigger node)  
    - Outputs: Emits new row data including fields “Task Name” and “Area”  
    - Edge cases:  
      - Authorization failure if Google OAuth credentials expire or are revoked  
      - Missed triggers if polling interval is too long or API rate limits reached  
      - Data inconsistencies if the sheet is edited concurrently  
    - Sticky Note: Describes the watch functionality and input fields  

#### 2.2 AI Processing

- **Overview:**  
  This block uses an AI Assignment Agent built on Langchain framework with Azure OpenAI GPT-4o-mini to analyze the task name and related area, consult the employee roster from another Google Sheet, and determine the best-fit employee based on a detailed scoring system.

- **Nodes Involved:**  
  - AI Agent  
  - Azure OpenAI Chat Model  
  - Get row(s) in sheet in Google Sheets  
  - Structured Output Parser  

- **Node Details:**  

  - **AI Agent**  
    - Type: Langchain Agent node (custom AI orchestration)  
    - Configuration:  
      - Input prompt: Combines “Task Name” and “Related Area” from the trigger data  
      - System message: Detailed instructions to normalize inputs, read employee expertise from Google Sheets, score and select best-fit employee, and classify the item as “bug” or “task” based on keywords  
      - Uses batching with 500 ms delay between batches  
      - Output parser enabled with structured JSON schema  
    - Inputs: Receives new task data from Google Sheets Trigger  
    - Outputs: Structured JSON with fields: task_name, assignee_name, expertise, employee_id, item_type  
    - Edge cases:  
      - AI model errors or timeouts  
      - Misclassification if task description is ambiguous  
      - Failure if Google Sheets roster is inaccessible or empty  
    - Sub-workflow: Integrates with Google Sheets tool node and Azure OpenAI model node  

  - **Azure OpenAI Chat Model**  
    - Type: AI language model node using Azure OpenAI API  
    - Configuration:  
      - Model: gpt-4o-mini, optimized for chat completion  
    - Credentials: Azure OpenAI API credentials required  
    - Inputs: Connected as the AI model for the AI Agent  
    - Outputs: AI-generated completions passed back to AI Agent  
    - Edge cases:  
      - API authentication failure  
      - Model usage limits or quota exceeded  

  - **Get row(s) in sheet in Google Sheets**  
    - Type: Google Sheets Tool node (read-only)  
    - Configuration:  
      - Document ID: points to employee roster sheet ("JIRA Tracking Account IDs")  
      - Sheet name: Sheet1 (gid=0)  
      - Reads all rows to provide employee expertise data to AI Agent  
    - Credentials: Google OAuth2 credentials (same or separate from trigger)  
    - Inputs: Invoked as a tool by AI Agent for employee data  
    - Outputs: Employee roster rows  
    - Edge cases:  
      - Data access or permission errors  
      - Empty or malformed roster sheet  

  - **Structured Output Parser**  
    - Type: Langchain Output Parser node  
    - Configuration:  
      - JSON schema example defines expected output fields (task_name, assignee_name, expertise, employee_id, item_type)  
    - Inputs: Parses AI Agent output  
    - Outputs: Structured JSON for downstream nodes  
    - Edge cases:  
      - Parsing failure if AI output format deviates from schema  

#### 2.3 Routing & Issue Creation

- **Overview:**  
  This block decides the Jira issue type based on the AI-classified “item_type” (bug or task) and creates the appropriate Jira issue with the assigned employee.

- **Nodes Involved:**  
  - Switch  
  - Create an issue (for bugs)  
  - Create an issue1 (for tasks)  

- **Node Details:**  

  - **Switch**  
    - Type: Switch node (conditional routing)  
    - Configuration:  
      - Routes based on `$json.output.item_type`  
      - If “bug”: routes to “Create an issue” node  
      - If “task”: routes to “Create an issue1” node  
    - Inputs: Structured output from AI Agent  
    - Outputs: Two branches depending on item_type  
    - Edge cases:  
      - Unexpected or missing item_type values  
      - Case sensitivity issues handled strictly  

  - **Create an issue** (Bug)  
    - Type: Jira node to create an issue  
    - Configuration:  
      - Project: “My Scrum Project” (ID 10000)  
      - Issue Type: Bug (ID 10005)  
      - Summary: AI parsed task_name  
      - Assignee: employee_id from AI output  
    - Credentials: Jira Software Cloud API credentials required  
    - Inputs: From Switch node when item_type is “bug”  
    - Outputs: Jira issue creation response  
    - Edge cases:  
      - Jira API authentication errors  
      - Invalid assignee ID causing assignment failure  
      - Project or issue type misconfiguration  

  - **Create an issue1** (Task)  
    - Type: Jira node to create an issue  
    - Configuration:  
      - Project: “My Scrum Project” (ID 10000)  
      - Issue Type: Task (ID 10004)  
      - Summary: AI parsed task_name  
      - Assignee: employee_id from AI output  
    - Credentials: Jira Software Cloud API credentials required  
    - Inputs: From Switch node when item_type is “task”  
    - Outputs: Jira issue creation response  
    - Edge cases: Same as above node, specific to Task issue type  

---

### 3. Summary Table

| Node Name                         | Node Type                           | Functional Role                                | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                                 |
|----------------------------------|-----------------------------------|-----------------------------------------------|-----------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger            | Google Sheets Trigger              | Watches new rows in tasks sheet                | —                           | AI Agent                | Watches a sheet for newly added rows and passes the Task Name and Related Area into the workflow for processing. |
| AI Agent                       | Langchain Agent                    | AI assignment logic combining task and expertise | Google Sheets Trigger       | Switch                  | Evaluates the task and area, compares against employee expertise from Sheets, and outputs structured assignment fields. |
| Azure OpenAI Chat Model         | Langchain AI Language Model       | Provides GPT-4o-mini chat completions           | AI Agent (ai_languageModel) | AI Agent                |                                                                                                             |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool               | Reads employee roster data                       | AI Agent (ai_tool)           | AI Agent                |                                                                                                             |
| Structured Output Parser         | Langchain Output Parser           | Parses AI output into structured JSON           | AI Agent (ai_outputParser)   | AI Agent                |                                                                                                             |
| Switch                         | Switch                           | Routes workflow based on item_type (bug/task)  | AI Agent                    | Create an issue, Create an issue1 | Routes workflow logic based on item_type to apply different handling or fields for a bug versus a task.        |
| Create an issue                 | Jira                             | Creates Jira bug issue with assignee            | Switch                      | —                       | Creates the Jira issue with the chosen assignee and metadata from the parsed output, completing the intake-to-ticket flow. |
| Create an issue1                | Jira                             | Creates Jira task issue with assignee           | Switch                      | —                       | Creates the Jira issue with the chosen assignee and metadata from the parsed output, completing the intake-to-ticket flow. |
| Sticky Note                    | Sticky Note                      | Documentation node                              | —                           | —                       | ## Google Sheets Trigger  \nWatches a sheet for newly added rows and passes the Task Name and Related Area into the workflow for processing. |
| Sticky Note1                   | Sticky Note                      | Documentation node                              | —                           | —                       | ## AI Agent (Azure OpenAI Chat Model)  \nEvaluates the task and area, compares against employee expertise from Sheets, and outputs the structured assignment fields. |
| Sticky Note2                   | Sticky Note                      | Documentation node                              | —                           | —                       | ## Switch (Rules)  \nRoutes workflow logic based on item_type to apply different handling or fields for a bug versus a task. |
| Sticky Note3                   | Sticky Note                      | Documentation node                              | —                           | —                       | ## Create an issue (Jira)  \nCreates the Jira issue with the chosen assignee and metadata from the parsed output, completing the intake-to-ticket flow. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Google Sheets Trigger** node.  
   - Set event to **rowAdded**.  
   - Select the Google Sheet document ID corresponding to your tasks sheet.  
   - Set sheet name to the relevant tab (e.g., “Sheet1”).  
   - Configure OAuth2 credentials for Google Sheets with necessary permissions.  
   - Set polling interval to every minute.  

2. **Add AI Agent Node:**  
   - Add a **Langchain Agent** node.  
   - Configure the input text as:  
     `Task Name: {{ $json['Task Name'] }}\nRelated Area: {{ $json.Area }}`  
   - Paste the system message exactly as specified, outlining the scoring rules, normalization, synonyms, and expected output fields.  
   - Enable batching with 500 ms delay.  
   - Enable output parser.  
   - Select prompt type as “define”.  
   - Enable retry on fail.  

3. **Add Azure OpenAI Chat Model Node:**  
   - Add **Azure OpenAI Chat Model** node.  
   - Select model **gpt-4o-mini**.  
   - Configure Azure OpenAI API credentials.  
   - Connect this node as the AI language model for the AI Agent node.  

4. **Add Google Sheets Tool Node for Employee Roster:**  
   - Add **Google Sheets Tool** node.  
   - Set document ID to the employee roster Google Sheet.  
   - Set sheet name (e.g., “Sheet1”).  
   - Configure OAuth2 credentials (can be same as trigger node if authorized).  
   - Connect this node as the AI tool for the AI Agent node.  

5. **Add Structured Output Parser Node:**  
   - Add **Langchain Output Parser Structured** node.  
   - Provide a JSON schema example with fields: task_name, assignee_name, expertise, employee_id, item_type.  
   - Connect this node as the AI output parser for the AI Agent node.  

6. **Add Switch Node:**  
   - Add a **Switch** node.  
   - Configure rules on the field `$json.output.item_type`:  
     - If equals “bug”, route to “Create an issue”.  
     - If equals “task”, route to “Create an issue1”.  

7. **Add Jira Issue Creation Nodes:**  
   - Add **Create an issue** node for bugs.  
   - Configure project to your Jira project (e.g., “My Scrum Project”).  
   - Set issue type to Bug.  
   - Set summary to `={{ $json.output.task_name }}`.  
   - Set assignee to `={{ $json.output.employee_id }}`.  
   - Configure Jira Software Cloud credentials.  
   - Connect input from Switch node bug branch.  

   - Add **Create an issue1** node for tasks.  
   - Configure project to same Jira project.  
   - Set issue type to Task.  
   - Set summary and assignee same as above.  
   - Configure Jira credentials.  
   - Connect input from Switch node task branch.  

8. **Connect Workflow:**  
   - Connect Google Sheets Trigger output to AI Agent input.  
   - Connect AI Agent output to Switch input.  
   - Connect Switch outputs to their respective Jira nodes.  

9. **Test and Validate:**  
   - Add test rows to the Google Sheet and verify the workflow triggers and creates Jira issues correctly.  
   - Monitor logs for errors related to authentication, API limits, parsing, or data mismatches.  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow uses Azure OpenAI GPT-4o-mini model for cost-effective yet powerful AI assignment logic.          | Azure OpenAI documentation: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/      |
| Employee expertise matching uses a nuanced scoring system including exact, substring, synonym, and category matches. | Design inspired by best practices for AI-based task routing and skill-based assignment.               |
| Ensure Google Sheets and Jira credentials have appropriate scopes and permissions to avoid runtime errors.      | Google Sheets API: https://developers.google.com/sheets/api <br> Jira Cloud API: https://developer.atlassian.com/cloud/jira/platform/rest/v3/  |
| Sticky notes in the workflow provide clear documentation for each functional block for maintenance and onboarding. |                                                                                                     |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.