Chat with a Google Sheet using AI

https://n8nworkflows.xyz/workflows/chat-with-a-google-sheet-using-ai-2085


# Chat with a Google Sheet using AI

### 1. Workflow Overview

This workflow enables interactive querying of a Google Sheet's data via a chat interface powered by an AI agent using GPT-4. Users can ask questions about customer data stored in a Google Sheet, and the AI will intelligently access and return relevant portions of that data. The design supports integration with n8n’s built-in chat but is adaptable for other chat platforms like Slack, Teams, or WhatsApp.

The workflow is logically divided into two main parts:

- **1.1 Main Workflow: AI Interaction and Query Processing**  
  Handles chat message reception, memory management, AI model invocation, and delegates specific data retrieval tasks to custom tools.

- **1.2 Sub-Workflow: Google Sheet Data Access Tools**  
  Implements three distinct tools callable by the AI agent to fetch data from the Google Sheet without loading the entire sheet into context, optimizing for GPT’s input size constraints:
    1. List all column names  
    2. Get all values of a specified column  
    3. Get all values for a specified row (customer)

This separation ensures efficient, contextual data retrieval and reduces the data passed into the AI model.

---

### 2. Block-by-Block Analysis

#### 2.1 Main Workflow: AI Interaction and Query Processing

**Overview:**  
This block listens for chat messages, manages conversational memory, invokes the GPT-4 chat model, and runs an AI agent that uses custom tools (implemented as sub-workflows) to retrieve Google Sheet data as needed.

**Nodes Involved:**  
- When chat message received  
- OpenAI Chat Model1  
- Simple Memory  
- AI Agent  
- List columns tool (ToolWorkflow)  
- Get column values tool (ToolWorkflow)  
- Get customer tool (ToolWorkflow)  

**Node Details:**

- **When chat message received**  
  - *Type:* langchain chatTrigger  
  - *Role:* Entry point triggered by user chat messages.  
  - *Config:* Uses webhook with ID for chat input.  
  - *Connections:* Outputs to AI Agent.  
  - *Potential Failures:* Webhook connectivity issues, malformed messages.  
  - *Version:* 1.1

- **OpenAI Chat Model1**  
  - *Type:* langchain lmChatOpenAi  
  - *Role:* Interfaces with GPT-4 model to generate AI responses.  
  - *Config:* Model set to `gpt-4o-mini` (GPT-4 variant).  
  - *Credentials:* Requires OpenAI API key with GPT-4 access.  
  - *Connections:* Outputs to AI Agent as language model input.  
  - *Potential Failures:* API key missing/expired, rate limits, network errors.  
  - *Version:* 1.2

- **Simple Memory**  
  - *Type:* langchain memoryBufferWindow  
  - *Role:* Maintains conversational context window for better dialogue continuity.  
  - *Config:* Default buffer window.  
  - *Connections:* Memory input to AI Agent.  
  - *Potential Failures:* Memory overflow with long conversations (though buffer window mitigates this).  
  - *Version:* 1.3

- **AI Agent**  
  - *Type:* langchain agent  
  - *Role:* Central processing node coordinating AI language model, memory, and tools. Executes the conversational logic and decides when/how to call tools.  
  - *Config:* Default options.  
  - *Connections:* Receives inputs from chat trigger, memory, and OpenAI model; outputs response back to chat.  
  - *Tools:* Linked to three ToolWorkflow nodes below.  
  - *Potential Failures:* Misconfiguration of tools, API errors, timeout from AI response.  
  - *Version:* 1.8

- **List columns tool**  
  - *Type:* langchain toolWorkflow  
  - *Role:* Tool for listing all column names in the Google Sheet.  
  - *Config:* Calls sub-workflow within same parent workflow (id resolved dynamically). Input expects an operation name `"column_names"`.  
  - *Connections:* Registered as a tool for AI Agent.  
  - *Potential Failures:* Sub-workflow invocation failure, Google Sheets API issues.  
  - *Version:* 2

- **Get column values tool**  
  - *Type:* langchain toolWorkflow  
  - *Role:* Tool for fetching all values of a specified column from the sheet.  
  - *Config:* Expects input with `"column_values"` operation and column name query string. Returns array of JSON objects with requested column and row number.  
  - *Connections:* Registered as a tool for AI Agent.  
  - *Potential Failures:* Invalid column name, sub-workflow failure, API rate limits.  
  - *Version:* 2

- **Get customer tool**  
  - *Type:* langchain toolWorkflow  
  - *Role:* Tool for retrieving all column data of a single customer (row) given by row number.  
  - *Config:* Input expects `"row"` operation and row number as string. Returns JSON object with full customer row data.  
  - *Connections:* Registered as a tool for AI Agent.  
  - *Potential Failures:* Invalid row number, sub-workflow failure.  
  - *Version:* 2

---

#### 2.2 Sub-Workflow: Google Sheet Data Access Tools

**Overview:**  
This sub-workflow is invoked by the three custom tools above. It receives operation and query parameters, reads the Google Sheet, then processes and returns the appropriate data subset based on the operation requested.

**Nodes Involved:**  
- When Executed by Another Workflow (executeWorkflowTrigger)  
- Set Google Sheet URL  
- Get Google sheet contents (Google Sheets node)  
- Check operation (Switch)  
- Get column names (Set)  
- Prepare column data (Set)  
- Filter (Filter)  
- Prepare output (Code)

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* executeWorkflowTrigger  
  - *Role:* Entry point when called as a sub-workflow by the AI tools.  
  - *Config:* Accepts inputs `operation` and `query`.  
  - *Connections:* Outputs to Set Google Sheet URL.  
  - *Potential Failures:* Invalid inputs, incorrect parameter passing.  
  - *Version:* 1.1

- **Set Google Sheet URL**  
  - *Type:* Set  
  - *Role:* Assigns the URL of the Google Sheet to a variable `sheetUrl`.  
  - *Config:* Hardcoded to the Google Sheet URL `"https://docs.google.com/spreadsheets/d/1GjFBV8HpraNWG_JyuaQAgTb3zUGguh0S_25nO0CMd8A/edit#gid=736425281"` (user editable).  
  - *Connections:* Outputs to Get Google sheet contents.  
  - *Potential Failures:* URL malformed or incorrect sheet access permissions.  
  - *Version:* 3.4

- **Get Google sheet contents**  
  - *Type:* googleSheets  
  - *Role:* Reads data from the Google Sheet `customer_data` tab using the URL from previous node.  
  - *Config:* Uses OAuth2 Google Sheets credentials.  
  - *Connections:* Outputs to Check operation.  
  - *Potential Failures:* Google API quota exceeded, invalid credentials, sheet name mismatch.  
  - *Version:* 4.5

- **Check operation**  
  - *Type:* Switch  
  - *Role:* Routes flow based on the `operation` input: `"column_names"`, `"column_values"`, or `"row"`.  
  - *Config:* Exact string matching on operation field.  
  - *Connections:*  
    - `"column_names"` → Get column names node  
    - `"column_values"` → Prepare column data node  
    - `"row"` → Filter node  
  - *Potential Failures:* Unexpected operation string, case sensitivity issues.  
  - *Version:* 3.2

- **Get column names**  
  - *Type:* Set  
  - *Role:* Extracts and returns all column names (keys) from the first row of the sheet data as an array in `response`.  
  - *Config:* Uses expression `Object.keys($json)` to get keys.  
  - *Connections:* Outputs to Prepare output.  
  - *Potential Failures:* Empty sheet data, malformed JSON.  
  - *Version:* 3.4

- **Prepare column data**  
  - *Type:* Set  
  - *Role:* Adds a numeric `row_number` property to each row for downstream filtering.  
  - *Config:* Sets `row_number` equal to row index from sheet data.  
  - *Connections:* Outputs to Prepare output.  
  - *Potential Failures:* Missing or invalid row numbers.  
  - *Version:* 3.4

- **Filter**  
  - *Type:* Filter  
  - *Role:* Filters rows where `row_number` matches the query string (row number as string).  
  - *Config:* Strict equality comparison with query parameter.  
  - *Connections:* Outputs to Prepare output.  
  - *Potential Failures:* Query value not a valid row number, filter returning empty.  
  - *Version:* 2.2

- **Prepare output**  
  - *Type:* Code  
  - *Role:* Serializes all matched rows to JSON string under `response`.  
  - *Config:* JavaScript code returns `response` field with JSON stringified array of rows.  
  - *Connections:* Terminal output of sub-workflow.  
  - *Potential Failures:* Code execution errors if input malformed.  
  - *Version:* 2

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                                | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                          |
|----------------------------|-----------------------------------|------------------------------------------------|---------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received  | langchain chatTrigger              | Entry point for user chat messages              | -                               | AI Agent                        | ### Main workflow: AI agent using custom tool                                                      |
| OpenAI Chat Model1          | langchain lmChatOpenAi             | Calls GPT-4 for AI language understanding       | -                               | AI Agent                        |                                                                                                    |
| Simple Memory               | langchain memoryBufferWindow       | Maintains conversational context                | -                               | AI Agent                        |                                                                                                    |
| AI Agent                   | langchain agent                   | Coordinates AI model, memory, and tools          | When chat message received, OpenAI Chat Model1, Simple Memory | -                               |                                                                                                    |
| List columns tool           | langchain toolWorkflow             | Tool: lists all column names in sheet            | -                               | AI Agent                        | These tools all call the sub-workflow below                                                        |
| Get column values tool      | langchain toolWorkflow             | Tool: gets all values of specified column        | -                               | AI Agent                        | These tools all call the sub-workflow below                                                        |
| Get customer tool           | langchain toolWorkflow             | Tool: gets all columns for a specified row       | -                               | AI Agent                        | These tools all call the sub-workflow below                                                        |
| When Executed by Another Workflow | executeWorkflowTrigger          | Sub-workflow entry point accepting operation/query | -                               | Set Google Sheet URL            | ### Sub-workflow: Custom tool<br>This can be called by the agent above. It returns three different types of data from the Google Sheet, which can be used together for more complex queries without returning the whole sheet (which might be too big for GPT to handle) |
| Set Google Sheet URL        | Set                               | Assigns the Google Sheet URL                      | When Executed by Another Workflow | Get Google sheet contents       | Change the URL of the Google Sheet here                                                             |
| Get Google sheet contents   | googleSheets                      | Reads data from Google Sheet using URL           | Set Google Sheet URL            | Check operation                 |                                                                                                    |
| Check operation             | Switch                           | Routes logic by operation type                     | Get Google sheet contents       | Get column names, Prepare column data, Filter |                                                                                                    |
| Get column names            | Set                               | Extracts column names as array                     | Check operation ("column_names") | Prepare output                 |                                                                                                    |
| Prepare column data         | Set                               | Adds row_number property for column_values output | Check operation ("column_values") | Prepare output                 |                                                                                                    |
| Filter                     | Filter                           | Filters rows by row_number matching query          | Check operation ("row")          | Prepare output                 |                                                                                                    |
| Prepare output              | Code                             | Serializes and formats response output             | Get column names, Prepare column data, Filter | -                               |                                                                                                    |
| Sticky Note1                | stickyNote                       | Information about sub-workflow                       | -                               | -                               | ### Sub-workflow: Custom tool<br>This can be called by the agent above. It returns three different types of data from the Google Sheet, which can be used together for more complex queries without returning the whole sheet (which might be too big for GPT to handle) |
| Sticky Note2                | stickyNote                       | Information about main workflow                      | -                               | -                               | ### Main workflow: AI agent using custom tool                                                      |
| Sticky Note3                | stickyNote                       | Prompt example for users                             | -                               | -                               | ## Try me out<br><br>Click the 'Chat' button at the bottom and enter:<br>_Which is our biggest customer?_ |
| Sticky Note4                | stickyNote                       | Instruction to change the Google Sheet URL          | -                               | -                               | Change the URL of the Google Sheet here                                                             |
| Sticky Note                 | stickyNote                       | Notes that all tools call the sub-workflow          | -                               | -                               | These tools all call the sub-workflow below                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: langchain chatTrigger  
   - Configure webhook with default settings.

2. **Create "OpenAI Chat Model1" node**  
   - Type: langchain lmChatOpenAi  
   - Set model to `gpt-4o-mini` (GPT-4 variant).  
   - Provide OpenAI API credentials with GPT-4 access.

3. **Create "Simple Memory" node**  
   - Type: langchain memoryBufferWindow  
   - Use default buffer settings.

4. **Create "AI Agent" node**  
   - Type: langchain agent  
   - Connect inputs from "When chat message received", "OpenAI Chat Model1" (as language model), and "Simple Memory" (as memory).  
   - No special configuration needed for options.  
   - This node will manage tool calls.

5. **Create three ToolWorkflow nodes to serve as tools for AI Agent:**

   - **List columns tool:**  
     - Type: langchain toolWorkflow  
     - Name: `list_columns`  
     - Description: "List all column names in customer data..."  
     - Workflow ID: Set dynamically to the current workflow's ID (to call sub-workflow)  
     - Inputs: `operation: "column_names"`, `query: "none"`  
     - Connect output as AI Agent's tool input.

   - **Get column values tool:**  
     - Type: langchain toolWorkflow  
     - Name: `column_values`  
     - Description: "Get the specified column value for all customers..."  
     - Workflow ID: same as above  
     - Inputs: `operation: "column_values"`, `query: <column_name>`  
     - Connect output as AI Agent's tool input.

   - **Get customer tool:**  
     - Type: langchain toolWorkflow  
     - Name: `get_customer`  
     - Description: "Get all columns for a given customer..."  
     - Workflow ID: same as above  
     - Inputs: `operation: "row"`, `query: <row_number_as_string>`  
     - Connect output as AI Agent's tool input.

6. **Connect "AI Agent" output back to chat interface** (implicit in chatTrigger node).

7. **Create Sub-Workflow (called by ToolWorkflow nodes):**

   - **When Executed by Another Workflow:**  
     - Type: executeWorkflowTrigger  
     - Inputs: `operation` and `query`.

   - **Set Google Sheet URL:**  
     - Type: Set  
     - Assign variable `sheetUrl` with the Google Sheet URL (replace with your own).  

   - **Get Google sheet contents:**  
     - Type: googleSheets  
     - Document ID: Extracted from `sheetUrl` variable.  
     - Sheet name: `customer_data`.  
     - Use OAuth2 Google Sheets credentials.

   - **Check operation:**  
     - Type: Switch  
     - Conditions:  
       - `"column_names"` → route to "Get column names"  
       - `"column_values"` → route to "Prepare column data"  
       - `"row"` → route to "Filter"  

   - **Get column names:**  
     - Type: Set  
     - Set `response` to array of keys from first JSON object (use `Object.keys($json)`).

   - **Prepare column data:**  
     - Type: Set  
     - Add `row_number` property with row index.

   - **Filter:**  
     - Type: Filter  
     - Condition: filter rows where `row_number.toString()` equals input `query`.

   - **Prepare output:**  
     - Type: Code  
     - JavaScript code to serialize filtered data to JSON string in `response`.

8. **Connect nodes in sub-workflow as per above logic.**

9. **Set credentials:**

   - OpenAI API key with GPT-4 support for OpenAI Chat Model node.  
   - Google OAuth2 credentials for Google Sheets node.

10. **Test workflow:**

    - Start the main workflow.  
    - Use chat interface, e.g., ask "Which is our biggest customer?"  
    - Verify correct AI responses based on Google Sheet data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires n8n version 1.19.4 or later.                                                                                        | Version requirement                                                                              |
| To adapt for Slack, Teams, WhatsApp, replace the "When chat message received" node with appropriate input nodes for those platforms.      | Integration adaptation note                                                                     |
| OpenAI GPT-4 API key must be valid and have sufficient quota to avoid rate limiting or timeout errors.                                    | Credential requirement                                                                          |
| Google Sheets URL must be accessible by the configured Google OAuth2 account and have a sheet named `customer_data`.                      | Data source setup                                                                               |
| Example user prompt: "Which is our biggest customer?"                                                                                      | User instruction (Sticky Note)                                                                  |
| More info on n8n chat and LangChain nodes: https://docs.n8n.io/integrations/builtin/n8n-nodes/langchain/                                   | Official n8n documentation                                                                       |
| This workflow demonstrates best practices for AI-assisted data querying with context window limitations through modular tool invocation. | Design insight                                                                                  |

---

This structured documentation provides a detailed, node-level understanding to enable reproduction, modification, and troubleshooting of the "Chat with a Google Sheet using AI" workflow in n8n.