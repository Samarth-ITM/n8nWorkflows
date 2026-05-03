Conversing with Data: Transforming Text into SQL Queries and Visual Curves

https://n8nworkflows.xyz/workflows/conversing-with-data--transforming-text-into-sql-queries-and-visual-curves-3497


# Conversing with Data: Transforming Text into SQL Queries and Visual Curves

### 1. Workflow Overview

This workflow, titled **"Conversing with Data: Transforming Text into SQL Queries and Visual Curves"**, enables users to interact with a PostgreSQL database using natural language queries. It translates user text inputs into SQL queries via AI, executes these queries on the database, and then visualizes the results using QuickChart. The workflow is designed to facilitate seamless data retrieval and visualization without requiring manual SQL query writing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Memory Management**: Captures user input via chat and maintains conversational context.
- **1.2 Database Schema Handling**: Retrieves and processes the database schema to inform query generation.
- **1.3 AI-Based SQL Query Generation**: Combines user input and schema, then uses AI (DeepSeek API) to generate SQL queries.
- **1.4 SQL Query Execution and Validation**: Executes the generated SQL query on PostgreSQL and checks for validity.
- **1.5 Data Formatting and Visualization Preparation**: Formats query results and prepares data for visualization.
- **1.6 Visualization Generation and Final Output**: Uses AI to generate visualization instructions and prepares the final response for the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Memory Management

- **Overview:**  
  This block receives the user's natural language query via a chat trigger and manages conversational memory to maintain context across interactions.

- **Nodes Involved:**  
  - Chat Trigger  
  - Window Buffer Memory  
  - AI Agent (initial processing)

- **Node Details:**

  - **Chat Trigger**  
    - *Type:* `@n8n/n8n-nodes-langchain.chatTrigger`  
    - *Role:* Entry point for user input via chat interface.  
    - *Configuration:* Uses webhook to receive chat messages.  
    - *Connections:* Outputs to "Load the schema from the local file".  
    - *Edge Cases:* Webhook failures, malformed input.

  - **Window Buffer Memory**  
    - *Type:* `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - *Role:* Maintains a sliding window of recent conversation history to provide context for AI processing.  
    - *Configuration:* Default parameters, no custom settings.  
    - *Connections:* Feeds memory context into "AI Agent".  
    - *Edge Cases:* Memory overflow or loss, context truncation.

  - **AI Agent**  
    - *Type:* `@n8n/n8n-nodes-langchain.agent`  
    - *Role:* Processes combined input and schema to generate SQL queries.  
    - *Configuration:* Uses DeepSeek API as language model.  
    - *Connections:* Receives input from "Combine schema data and chat input", outputs to "Extract SQL query".  
    - *Edge Cases:* API authentication errors, rate limits, response parsing errors.

---

#### 1.2 Database Schema Handling

- **Overview:**  
  This block loads the database schema from a local file and extracts relevant data to inform the AI in generating accurate SQL queries.

- **Nodes Involved:**  
  - Load the schema from the local file  
  - Extract data from file  
  - Combine schema data and chat input

- **Node Details:**

  - **Load the schema from the local file**  
    - *Type:* `n8n-nodes-base.readWriteFile`  
    - *Role:* Reads the saved JSON schema file from local storage.  
    - *Configuration:* Reads file containing database schema details.  
    - *Connections:* Outputs to "Extract data from file".  
    - *Edge Cases:* File not found, read permission errors, malformed JSON.

  - **Extract data from file**  
    - *Type:* `n8n-nodes-base.extractFromFile`  
    - *Role:* Parses the schema JSON to extract table and column information.  
    - *Configuration:* Default extraction settings.  
    - *Connections:* Outputs to "Combine schema data and chat input".  
    - *Edge Cases:* Parsing errors, unexpected schema format.

  - **Combine schema data and chat input**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Merges the extracted schema data with the user's chat input to form a comprehensive prompt for AI.  
    - *Configuration:* Sets combined data as input for AI Agent.  
    - *Connections:* Outputs to "AI Agent".  
    - *Edge Cases:* Data merging errors, missing fields.

---

#### 1.3 AI-Based SQL Query Generation

- **Overview:**  
  This block uses the DeepSeek API via the AI Agent to translate the natural language query combined with schema context into a valid SQL query.

- **Nodes Involved:**  
  - AI Agent  
  - Extract SQL query  
  - Check if query exists

- **Node Details:**

  - **AI Agent**  
    - *Type:* `@n8n/n8n-nodes-langchain.agent`  
    - *Role:* Generates SQL query from combined input.  
    - *Configuration:* Uses DeepSeek API credentials configured in n8n.  
    - *Connections:* Outputs to "Extract SQL query".  
    - *Edge Cases:* API errors, incomplete or invalid SQL generation.

  - **Extract SQL query**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Extracts the SQL query string from AI Agent output.  
    - *Configuration:* Sets the SQL query as a workflow variable.  
    - *Connections:* Outputs to "Check if query exists".  
    - *Edge Cases:* Missing or malformed SQL in AI response.

  - **Check if query exists**  
    - *Type:* `n8n-nodes-base.if`  
    - *Role:* Validates presence of SQL query before proceeding.  
    - *Configuration:* Conditional check on SQL query variable.  
    - *Connections:*  
      - If true: proceeds to "Final SQL result" node.  
      - If false: routes to "No Operation, do nothing".  
    - *Edge Cases:* False negatives due to parsing errors.

---

#### 1.4 SQL Query Execution and Validation

- **Overview:**  
  Executes the validated SQL query against the PostgreSQL database and retrieves the result set.

- **Nodes Involved:**  
  - Final SQL result  
  - Format query results  
  - Edit Fields  
  - plot agent

- **Node Details:**

  - **Final SQL result**  
    - *Type:* `n8n-nodes-base.postgres`  
    - *Role:* Runs the SQL query on PostgreSQL database.  
    - *Configuration:* Uses PostgreSQL credentials configured in n8n; query set dynamically from previous node.  
    - *Connections:* Outputs to "Format query results".  
    - *Edge Cases:* SQL syntax errors, connection failures, timeouts.

  - **Format query results**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Formats raw query results into a structured format suitable for visualization and further processing.  
    - *Configuration:* Sets fields for chart data extraction.  
    - *Connections:* Outputs to "Combine query result and chat answer" and "Edit Fields".  
    - *Edge Cases:* Empty result sets, data type inconsistencies.

  - **Edit Fields**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Prepares and cleans data fields for the plotting agent.  
    - *Configuration:* Adjusts data structure as required by AI visualization agent.  
    - *Connections:* Outputs to "plot agent".  
    - *Edge Cases:* Missing fields, data format mismatches.

  - **plot agent**  
    - *Type:* `@n8n/n8n-nodes-langchain.agent`  
    - *Role:* Uses AI to generate visualization instructions (e.g., chart type, labels) based on query results.  
    - *Configuration:* Uses DeepSeek or OpenAI model for visualization prompt.  
    - *Connections:* Outputs to "Edit Fields1".  
    - *Edge Cases:* AI misinterpretation, API errors.

---

#### 1.5 Data Formatting and Visualization Preparation

- **Overview:**  
  This block merges the SQL query results with AI-generated visualization instructions and prepares the final output.

- **Nodes Involved:**  
  - Edit Fields1  
  - Combine query result and chat answer  
  - Prepare final output  
  - Structured Output Parser

- **Node Details:**

  - **Edit Fields1**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Finalizes data formatting for output.  
    - *Configuration:* Sets final fields for response.  
    - *Connections:* Outputs to "Combine query result and chat answer".  
    - *Edge Cases:* Data mismatch, missing fields.

  - **Combine query result and chat answer**  
    - *Type:* `n8n-nodes-base.merge`  
    - *Role:* Merges SQL query results with AI-generated visualization data and chat responses.  
    - *Configuration:* Merges multiple inputs into a single output.  
    - *Connections:* Outputs to "Prepare final output".  
    - *Edge Cases:* Merge conflicts, data loss.

  - **Prepare final output**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Prepares the final response payload including visualization URLs and data insights.  
    - *Configuration:* Sets output fields for user delivery.  
    - *Connections:* Terminal node for output delivery.  
    - *Edge Cases:* Missing data, formatting errors.

  - **Structured Output Parser**  
    - *Type:* `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - *Role:* Parses AI output into structured format for visualization agent.  
    - *Configuration:* Default structured parsing settings.  
    - *Connections:* Feeds into "plot agent".  
    - *Edge Cases:* Parsing failures, unexpected AI output format.

---

#### 1.6 Visualization Generation and Final Output

- **Overview:**  
  This block handles the generation of visual charts using QuickChart and delivers the final visualization and data insights to the user.

- **Nodes Involved:**  
  - No Operation, do nothing (fallback)  
  - Sticky Notes (documentation and comments)

- **Node Details:**

  - **No Operation, do nothing**  
    - *Type:* `n8n-nodes-base.noOp`  
    - *Role:* Acts as a fallback when no valid SQL query is generated.  
    - *Configuration:* No parameters.  
    - *Connections:* Terminal node for false condition in "Check if query exists".  
    - *Edge Cases:* Workflow halts gracefully on invalid input.

  - **Sticky Notes**  
    - *Type:* `n8n-nodes-base.stickyNote`  
    - *Role:* Provide inline documentation and comments within the workflow editor.  
    - *Configuration:* Various notes placed near relevant nodes.  
    - *Connections:* None (informational only).  
    - *Edge Cases:* None.

---

### 3. Summary Table

| Node Name                         | Node Type                                    | Functional Role                              | Input Node(s)                      | Output Node(s)                        | Sticky Note                              |
|----------------------------------|----------------------------------------------|----------------------------------------------|-----------------------------------|-------------------------------------|-----------------------------------------|
| Chat Trigger                     | @n8n/n8n-nodes-langchain.chatTrigger         | Receives user natural language input         | -                                 | Load the schema from the local file |                                         |
| Load the schema from the local file | n8n-nodes-base.readWriteFile                  | Reads saved database schema JSON file         | Chat Trigger                     | Extract data from file               |                                         |
| Extract data from file           | n8n-nodes-base.extractFromFile                | Parses schema JSON to extract table info      | Load the schema from the local file | Combine schema data and chat input |                                         |
| Combine schema data and chat input | n8n-nodes-base.set                            | Merges schema and user input for AI prompt    | Extract data from file            | AI Agent                           |                                         |
| Window Buffer Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow  | Maintains conversation context memory         | -                                 | AI Agent                           |                                         |
| AI Agent                        | @n8n/n8n-nodes-langchain.agent                | Generates SQL query from combined input       | Combine schema data and chat input, Window Buffer Memory | Extract SQL query                |                                         |
| Extract SQL query               | n8n-nodes-base.set                            | Extracts SQL query string from AI output      | AI Agent                        | Check if query exists               |                                         |
| Check if query exists           | n8n-nodes-base.if                             | Validates presence of SQL query                | Extract SQL query               | Final SQL result, Combine query result and chat answer, No Operation, do nothing |                                         |
| Final SQL result                | n8n-nodes-base.postgres                        | Executes SQL query on PostgreSQL               | Check if query exists            | Format query results               |                                         |
| Format query results            | n8n-nodes-base.set                            | Formats SQL query results for visualization   | Final SQL result                | Combine query result and chat answer, Edit Fields |                                         |
| Edit Fields                    | n8n-nodes-base.set                            | Prepares data fields for visualization agent  | Format query results            | plot agent                       |                                         |
| plot agent                    | @n8n/n8n-nodes-langchain.agent                | Generates visualization instructions via AI  | Edit Fields                    | Edit Fields1                     |                                         |
| Edit Fields1                   | n8n-nodes-base.set                            | Finalizes data formatting for output          | plot agent                    | Combine query result and chat answer |                                         |
| Combine query result and chat answer | n8n-nodes-base.merge                          | Merges SQL results and AI visualization data  | Format query results, Check if query exists (true), Edit Fields1 | Prepare final output             |                                         |
| Prepare final output           | n8n-nodes-base.set                            | Prepares final response for user               | Combine query result and chat answer | -                               |                                         |
| No Operation, do nothing       | n8n-nodes-base.noOp                           | Fallback for missing SQL query                  | Check if query exists (false)   | -                                 |                                         |
| List all tables in a database  | n8n-nodes-base.postgres                        | Retrieves list of tables in database            | When clicking "Test workflow"   | Schema Extractor                 |                                         |
| Schema Extractor               | n8n-nodes-base.postgres                        | Extracts detailed schema info from database    | List all tables in a database    | Add table name to output          |                                         |
| Add table name to output       | n8n-nodes-base.set                            | Adds table names to schema output               | Schema Extractor               | Convert data to Json             |                                         |
| Convert data to Json           | n8n-nodes-base.convertToFile                   | Converts schema data to JSON file                | Add table name to output        | Save file locally               |                                         |
| Save file locally              | n8n-nodes-base.readWriteFile                   | Saves schema JSON file locally                   | Convert data to Json            | -                               |                                         |
| Structured Output Parser       | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI output into structured format         | -                             | plot agent                     |                                         |
| Deepseek-chat                 | @n8n/n8n-nodes-langchain.lmChatOpenAi          | AI language model for SQL generation             | -                             | AI Agent                       |                                         |
| deepseek-chat                 | @n8n/n8n-nodes-langchain.lmChatOpenAi          | AI language model for visualization generation   | -                             | plot agent                     |                                         |
| When clicking "Test workflow" | n8n-nodes-base.manualTrigger                    | Manual trigger for testing workflow              | -                             | List all tables in a database  |                                         |
| Sticky Notes                  | n8n-nodes-base.stickyNote                        | Inline documentation and comments                 | -                             | -                               | Various notes placed near relevant nodes |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook to receive user natural language queries.

2. **Add File Read Node to Load Schema**  
   - Type: `n8n-nodes-base.readWriteFile`  
   - Configure to read the local JSON file containing the database schema.

3. **Add Extract From File Node**  
   - Type: `n8n-nodes-base.extractFromFile`  
   - Connect from schema file read node.  
   - Default extraction settings to parse JSON schema.

4. **Add Set Node to Combine Schema and Chat Input**  
   - Type: `n8n-nodes-base.set`  
   - Merge extracted schema data and user chat input into one data object.

5. **Add Window Buffer Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - No special configuration; maintains recent conversation context.

6. **Add AI Agent Node for SQL Generation**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configure with DeepSeek API credentials.  
   - Connect inputs from combined schema/chat data and memory buffer.

7. **Add Set Node to Extract SQL Query**  
   - Type: `n8n-nodes-base.set`  
   - Extract SQL query string from AI Agent output.

8. **Add If Node to Check SQL Query Existence**  
   - Type: `n8n-nodes-base.if`  
   - Condition: Check if SQL query string is present and valid.

9. **Add PostgreSQL Node to Execute SQL Query**  
   - Type: `n8n-nodes-base.postgres`  
   - Configure with PostgreSQL credentials.  
   - Use SQL query from previous node.

10. **Add Set Node to Format Query Results**  
    - Type: `n8n-nodes-base.set`  
    - Format raw query results for visualization.

11. **Add Set Node to Edit Fields for Visualization**  
    - Type: `n8n-nodes-base.set`  
    - Prepare data fields as required by visualization AI.

12. **Add AI Agent Node for Visualization Instructions**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Configure with DeepSeek or OpenAI credentials.  
    - Connect input from edited fields node.

13. **Add Set Node to Finalize Visualization Data**  
    - Type: `n8n-nodes-base.set`  
    - Final formatting of visualization instructions.

14. **Add Merge Node to Combine Query Results and Visualization Data**  
    - Type: `n8n-nodes-base.merge`  
    - Merge formatted query results and visualization instructions.

15. **Add Set Node to Prepare Final Output**  
    - Type: `n8n-nodes-base.set`  
    - Prepare the final response including visualization URLs and data insights.

16. **Add No Operation Node for Fallback**  
    - Type: `n8n-nodes-base.noOp`  
    - Connect as fallback if no valid SQL query is generated.

17. **Add Supporting Nodes for Schema Extraction (Optional for Setup)**  
    - Manual Trigger node to start schema extraction.  
    - PostgreSQL nodes to list tables and extract schema details.  
    - Set, convertToFile, and readWriteFile nodes to save schema JSON locally.

18. **Credential Setup**  
    - Configure DeepSeek API credentials with your API key.  
    - Configure PostgreSQL credentials with connection details (host, port, user, password, database).  
    - Ensure QuickChart service is accessible via URL (no credentials needed).

19. **Testing and Validation**  
    - Use the manual trigger to extract and save schema initially.  
    - Test chat trigger with sample natural language queries.  
    - Verify SQL generation, execution, and visualization output.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| To expose a local PostgreSQL database for remote access, use ngrok TCP tunneling on port 5432.     | https://ngrok.com/download and run `ngrok tcp 5432`                                             |
| DeepSeek API is used for natural language to SQL translation; obtain API key from DeepSeek account.| DeepSeek API documentation and account portal                                                    |
| QuickChart service URL for visualization: `https://quickchart.io/chart`                            | QuickChart official website: https://quickchart.io                                              |
| n8n installation and credential setup guide                                                        | https://docs.n8n.io/getting-started/installation/                                               |
| Workflow tags include "Talk to data" to categorize this workflow                                  | Tag visible in n8n workflow metadata                                                            |

---

This documentation provides a comprehensive understanding of the workflow structure, node configurations, and operational logic, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.