Create an Interactive Snowflake Data Explorer with GPT-4o Chat Interface and Visual Reports

https://n8nworkflows.xyz/workflows/create-an-interactive-snowflake-data-explorer-with-gpt-4o-chat-interface-and-visual-reports-5435


# Create an Interactive Snowflake Data Explorer with GPT-4o Chat Interface and Visual Reports

### 1. Workflow Overview

This workflow enables an interactive conversational interface for querying and exploring data stored in a Snowflake database using an AI-powered chat interface based on GPT-4o. It targets developers, data analysts, and business professionals who want to interact with Snowflake data through natural language without manually writing SQL. The workflow also supports generating dynamic visual reports (tables and charts) from query results and serves these as HTML dashboards via a webhook.

The workflow logic is organized into the following main blocks:

- **1.1 Chat Input Reception and AI Processing:** Receives user chat messages, manages session memory, and invokes an AI agent (GPT-4o) configured as a Snowflake SQL assistant to interpret user requests into SQL queries.
- **1.2 Snowflake Schema and Table Metadata Retrieval:** Retrieves metadata about available database schemas and table definitions to inform SQL query generation.
- **1.3 SQL Query Execution and Result Aggregation:** Executes generated SQL queries on Snowflake, aggregates results, and applies logic based on result size.
- **1.4 Interactive HTML Dashboard Generation:** Builds a rich HTML report including table and graph visualizations using the query results, with client-side interactivity (filtering, sorting, pagination, chart selection).
- **1.5 Webhook API Exposure and Response:** Exposes a webhook endpoint to receive external requests and respond with the generated HTML dashboard or error page.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Input Reception and AI Processing

**Overview:**  
This block listens for incoming chat messages from users, maintains conversational memory per session, and uses the GPT-4o language model configured as a Snowflake SQL assistant. The AI agent interprets natural language inputs into SQL queries, leveraging knowledge of database schema and table definitions.

**Nodes Involved:**  
- When chat message received  
- Simple Memory  
- OpenAI Chat Model1  
- AI Agent1  
- DB Schema1  
- Get table definition  
- Retrieve Data  

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Entry point for chat messages; triggers workflow on new user messages.  
  - Config: Default options; webhookId assigned internally.  
  - Inputs: Webhook HTTP POST with chat payload.  
  - Outputs: Passes message data to AI Agent.  
  - Edge Cases: Network issues, malformed chat data.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context per session (using sessionId from chat message).  
  - Config: Session key bound to chat sessionId dynamically.  
  - Inputs: Chat message node output.  
  - Outputs: Provides context to AI Agent.  
  - Edge Cases: Memory overflow, sessionId missing or malformed.

- **OpenAI Chat Model1**  
  - Type: LangChain OpenAI Chat Model  
  - Role: GPT-4o-mini model for natural language processing.  
  - Config: Model set to "gpt-4o-mini", uses OpenAI API credentials named "Test club key".  
  - Inputs: Chat message + memory context.  
  - Outputs: Processed AI responses.  
  - Edge Cases: API key invalid, rate limits, network failures.

- **AI Agent1**  
  - Type: LangChain Agent  
  - Role: Core AI agent managing interaction, using tools to query database schema and generate SQL.  
  - Config: System message defines role as Snowflake SQL assistant emphasizing schema awareness. Output parser enabled for structured response.  
  - Inputs: Chat message, memory, AI language model, and tools (DB schema nodes).  
  - Outputs: AI-generated SQL queries and responses.  
  - Edge Cases: Parsing failures, incomplete schema info, timeout.

- **DB Schema1**  
  - Type: Snowflake Tool (SQL Query Executor)  
  - Role: Retrieves list of tables in the TPCH_SF1 schema to inform AI about available tables.  
  - Config: Executes manual query on Snowflake information_schema.tables filtered by schema.  
  - Inputs: Invoked as AI tool by AI Agent node.  
  - Outputs: Table list JSON to AI Agent.  
  - Edge Cases: Schema name mismatch, permission errors.

- **Get table definition**  
  - Type: Snowflake Tool (SQL Query Executor)  
  - Role: Retrieves column names and data types for a given table, helping AI build accurate SQL queries.  
  - Config: Parameterized query filtering on table_name from AI input; schema fixed as TPCH_SF1.  
  - Inputs: Invoked as AI tool by AI Agent node with dynamic table_name.  
  - Outputs: Table column metadata to AI Agent.  
  - Edge Cases: Incorrect table_name, empty results, permission errors.

- **Retrieve Data**  
  - Type: LangChain Tool Workflow (Sub-workflow)  
  - Role: Executes the generated SQL query and returns data; supports analytical SQL functions and complex queries.  
  - Config: Connected to an internal workflow with ID "kqpZSjy0tzRRY4hH"; passes query string from AI output.  
  - Inputs: AI-generated SQL query.  
  - Outputs: Query result data to AI Agent.  
  - Edge Cases: SQL errors, execution timeout, large result sets.

---

#### 2.2 Snowflake Query Execution and Result Aggregation

**Overview:**  
This block executes the SQL query generated by the AI agent on the Snowflake database, aggregates the returned data, and makes decisions based on dataset size (e.g., to provide direct results or link to a detailed report).

**Nodes Involved:**  
- When Executed by Another Workflow  
- Execute SQL  
- Aggregate Data  
- If Count>100  
- Link to Report  
- Return Data  
- Return Error  

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Triggers this sub-workflow when called by other workflows, receiving a SQL query as input.  
  - Config: Requires a workflow input parameter named "query".  
  - Inputs: External workflow call with SQL query.  
  - Outputs: Passes query to Execute SQL.  
  - Edge Cases: Missing or invalid query parameter.

- **Execute SQL**  
  - Type: Snowflake Node  
  - Role: Runs the provided SQL query on Snowflake.  
  - Config: Query parameterized from input JSON, operation "executeQuery".  
  - Inputs: SQL query string from trigger node.  
  - Outputs: Raw query results or error.  
  - Edge Cases: SQL syntax error, connection issues.

- **Aggregate Data**  
  - Type: Aggregate Node  
  - Role: Aggregates all returned items into a single array for evaluation.  
  - Config: Aggregates all item data.  
  - Inputs: Raw query results.  
  - Outputs: Aggregated data JSON.  
  - Edge Cases: Empty result sets.

- **If Count>100**  
  - Type: If Node  
  - Role: Conditional branching: if result count exceeds 100 rows, provide a report link; else return data directly.  
  - Config: Checks if length of $json.data array is greater than 100.  
  - Inputs: Aggregated data.  
  - Outputs:  
    - True branch: Link to Report node  
    - False branch: Return Data node  
  - Edge Cases: Incorrect data format, missing data array.

- **Link to Report**  
  - Type: Set Node  
  - Role: Constructs a markdown link to the full report dashboard served by the webhook, encoding the SQL query in URL.  
  - Config: Sets JSON output with markdown link including encoded SQL query parameter.  
  - Inputs: Data from If node.  
  - Outputs: Link response to caller.  
  - Edge Cases: URL encoding issues, malformed queries.

- **Return Data**  
  - Type: Set Node  
  - Role: Returns the raw data directly to the caller if dataset is small.  
  - Config: Passes through JSON output unchanged.  
  - Inputs: Data from If node.  
  - Outputs: Data response.  
  - Edge Cases: Large payload size.

- **Return Error**  
  - Type: Set Node  
  - Role: Returns error details if SQL execution fails.  
  - Config: Passes error JSON for downstream handling.  
  - Inputs: Failed SQL execution outputs.  
  - Outputs: Error response.  
  - Edge Cases: Error formatting.

---

#### 2.3 Interactive HTML Dashboard Generation and Webhook Response

**Overview:**  
This block generates an interactive HTML report to visualize query results as sortable, paginated tables and dynamic charts. It exposes a webhook endpoint that serves the report or an error page depending on processing outcome.

**Nodes Involved:**  
- Webhook  
- Snowflake1  
- Aggregate1  
- Set HTML  
- Respond to Webhook  
- Error page  

**Node Details:**

- **Webhook**  
  - Type: Webhook Node  
  - Role: Public HTTP endpoint to receive requests with SQL queries and respond with interactive reports.  
  - Config: Path configured as "87893585-d157-468d-a9af-7238784e814c", response mode set to respond from another node.  
  - Inputs: HTTP GET/POST requests with SQL query parameter.  
  - Outputs: Query to Snowflake1 node.  
  - Edge Cases: Unauthorized access, malformed requests.

- **Snowflake1**  
  - Type: Snowflake Node  
  - Role: Executes the SQL query received from webhook input.  
  - Config: Query dynamically passed from JSON input.  
  - Inputs: Query from webhook.  
  - Outputs: Query results to Aggregate1.  
  - Edge Cases: SQL errors, connection issues.

- **Aggregate1**  
  - Type: Aggregate Node  
  - Role: Aggregates all rows into a single data array.  
  - Inputs: Raw query results.  
  - Outputs: Data to Set HTML.  
  - Edge Cases: Empty result sets.

- **Set HTML**  
  - Type: Set Node  
  - Role: Dynamically builds full HTML content embedding the query data as JSON and includes JavaScript for interactive table and graph.  
  - Config:  
    - Assigns a single string field "html" containing full HTML document.  
    - The HTML uses React, Chart.js, and vanilla JavaScript to render UI: tabs for table/graph, filtering, sorting, pagination, chart templates, aggregation options.  
    - The JSON data embedded in JavaScript variable `data` is populated from the query results.  
  - Inputs: Aggregated data JSON.  
  - Outputs: HTML content to Respond to Webhook or Error page on failure.  
  - Edge Cases: Large data payloads, HTML string encoding issues, JavaScript errors in browser.

- **Respond to Webhook**  
  - Type: Respond to Webhook Node  
  - Role: Sends the generated HTML page as HTTP response with content-type text/html.  
  - Inputs: HTML content from Set HTML node.  
  - Outputs: HTTP response to client.  
  - Edge Cases: Network errors.

- **Error page**  
  - Type: Set Node  
  - Role: Generates a user-friendly error HTML page with styled message and retry/close buttons for client display when an error occurs in report generation.  
  - Config: Sets "html" containing error page markup.  
  - Inputs: Error data or failure from previous nodes.  
  - Outputs: HTML content to Respond to Webhook.  
  - Edge Cases: Error page display issues.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                                  | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                           |
|----------------------------|------------------------------------|-------------------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger              | Entry point for chat messages                    | (Webhook POST)                   | AI Agent1                       |                                                                                                     |
| Simple Memory              | LangChain Memory Buffer Window      | Maintains session conversational memory         | When chat message received       | AI Agent1                      |                                                                                                     |
| OpenAI Chat Model1         | LangChain OpenAI Chat Model         | GPT-4o-mini model for language understanding    | Simple Memory                   | AI Agent1                      |                                                                                                     |
| AI Agent1                 | LangChain Agent                     | Processes chat input into SQL queries            | When chat message received, Simple Memory, OpenAI Chat Model1, DB Schema1, Get table definition, Retrieve Data | (No direct output, managed internally) | Sticky Note "### Agent"                                                                                 |
| DB Schema1                | Snowflake Tool (SQL executor)       | Retrieves database tables in schema              | AI Agent1 (as tool)             | AI Agent1                      | Sticky Note "### Replace name of schema and database"                                               |
| Get table definition       | Snowflake Tool (SQL executor)       | Retrieves column names and types for table       | AI Agent1 (as tool)             | AI Agent1                      | Sticky Note "### Replace name of schema and database"                                               |
| Retrieve Data             | LangChain Tool Workflow             | Runs AI-generated SQL query and returns data     | AI Agent1 (as tool)             | AI Agent1                      | Sticky Note "### Map this workflow"                                                                 |
| When Executed by Another Workflow | Execute Workflow Trigger           | Sub-workflow trigger for external SQL query      | External                      | Execute SQL                    |                                                                                                     |
| Execute SQL               | Snowflake Node                      | Executes given SQL query                          | When Executed by Another Workflow | Aggregate Data, Return Error  |                                                                                                     |
| Aggregate Data            | Aggregate Node                     | Aggregates SQL result rows                        | Execute SQL                    | If Count>100                   |                                                                                                     |
| If Count>100              | If Node                           | Conditional split on row count                    | Aggregate Data                 | Link to Report, Return Data    |                                                                                                     |
| Link to Report            | Set Node                         | Generates markdown link to detailed report       | If Count>100 (true branch)     | (Output)                      | Sticky Note "### Replace webhook address"                                                           |
| Return Data               | Set Node                         | Returns raw data directly                         | If Count>100 (false branch)    | (Output)                      |                                                                                                     |
| Return Error              | Set Node                         | Returns error details                             | Execute SQL (error branch)     | (Output)                      |                                                                                                     |
| Webhook                   | Webhook Node                     | Public HTTP endpoint for dashboard requests      | HTTP Request                  | Snowflake1                    | Sticky Note "### Report workflow"                                                                   |
| Snowflake1                | Snowflake Node                   | Executes SQL from webhook                         | Webhook                       | Aggregate1, Error page         |                                                                                                     |
| Aggregate1                | Aggregate Node                  | Aggregates rows for HTML report                   | Snowflake1                    | Set HTML                      |                                                                                                     |
| Set HTML                  | Set Node                       | Generates interactive HTML report                 | Aggregate1                    | Respond to Webhook, Error page | Sticky Note "### Report workflow"                                                                   |
| Respond to Webhook        | Respond to Webhook Node          | Sends HTML report or error page                   | Set HTML, Error page           | HTTP Response                 |                                                                                                     |
| Error page                | Set Node                       | Generates error HTML page                          | Snowflake1 error branch, Set HTML error branch | Respond to Webhook           |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add "When chat message received" node:**  
   - Type: @n8n/n8n-nodes-langchain.chatTrigger  
   - Configure default options, no special parameters.  
   - This node acts as the chat entry point.

3. **Add "Simple Memory" node:**  
   - Type: @n8n/n8n-nodes-langchain.memoryBufferWindow  
   - Parameter: Set `sessionKey` to `={{ $('When chat message received').item.json.sessionId }}` (to maintain session context).

4. **Add "OpenAI Chat Model1" node:**  
   - Type: @n8n/n8n-nodes-langchain.lmChatOpenAi  
   - Set Model to "gpt-4o-mini".  
   - Credentials: Attach OpenAI API credential with valid key.  
   - No additional options needed.

5. **Add "AI Agent1" node:**  
   - Type: @n8n/n8n-nodes-langchain.agent  
   - Parameters:  
     - System message: "You are Snowflake SQL assistant.\n\nUse tools to retrieve data from Snowflake and answer user.\n\nIMPORTANT Always check database schema and table definition for preparing SQL query."  
     - Enable output parser.  
   - Connect inputs: from "When chat message received" (main), "Simple Memory" (ai_memory), "OpenAI Chat Model1" (ai_languageModel).

6. **Add "DB Schema1" node:**  
   - Type: Snowflake Tool (n8n-nodes-base.snowflakeTool)  
   - Query:  
     ```sql
     SELECT table_schema, table_name
     FROM information_schema.tables
     WHERE table_schema = 'YOUR_SCHEMA_NAME';
     ```  
   - Credentials: Connect Snowflake account with proper permissions.  
   - Connect as AI tool input to "AI Agent1".

7. **Add "Get table definition" node:**  
   - Type: Snowflake Tool  
   - Query:  
     ```sql
     SELECT column_name, data_type
     FROM information_schema.columns
     WHERE table_name = '{{ $fromAI("table_name") }}'
       AND table_schema = 'YOUR_SCHEMA_NAME'
     ORDER BY ordinal_position;
     ```  
   - Credentials: Same Snowflake account.  
   - Connect as AI tool input to "AI Agent1".

8. **Add "Retrieve Data" node:**  
   - Type: LangChain Tool Workflow  
   - Parameter: Link to sub-workflow that executes AI-generated SQL query and returns results.  
   - Pass the parameter: `query` from AI output field `sql_query`.  
   - Connect as AI tool input to "AI Agent1".

9. **Create sub-workflow for executing SQL query:**  
   - Add "When Executed by Another Workflow" trigger node with input parameter `query` (string).  
   - Add "Execute SQL" Snowflake node: query `{{ $json.query }}`.  
   - Add "Aggregate Data" node to aggregate rows.  
   - Add "If Count>100" node: condition to check if number of rows > 100.  
   - If true: "Link to Report" node to generate markdown link with URL encoding of query.  
   - If false: "Return Data" node to return raw query results.  
   - If error in Execute SQL: "Return Error" node.  
   - Connect nodes accordingly.  
   - Save and get workflow ID; use in "Retrieve Data" node in main workflow.

10. **Add "Webhook" node:**  
    - Type: Webhook  
    - Path: e.g., "87893585-d157-468d-a9af-7238784e814c"  
    - Response mode: "responseNode"  
    - This node will receive external requests with SQL query parameter.

11. **Add "Snowflake1" node:**  
    - Type: Snowflake  
    - Query: `{{ $json.query.sql }}` (from webhook request).  
    - Connect "Webhook" main output to this node.

12. **Add "Aggregate1" node:**  
    - Type: Aggregate  
    - Aggregate all rows from Snowflake1.  
    - Connect Snowflake1 output to Aggregate1.

13. **Add "Set HTML" node:**  
    - Type: Set  
    - Assign single field "html" with full HTML document string embedding JSON data as JavaScript variable.  
    - The HTML includes React and Chart.js dependencies, interactive table and graph code, filtering, sorting, pagination, chart templates, aggregation options, and tab switching.  
    - Connect Aggregate1 output to Set HTML.

14. **Add "Respond to Webhook" node:**  
    - Type: Respond to Webhook  
    - Respond with content-type text/html, body from Set HTML node field `html`.  
    - Connect Set HTML main output to Respond to Webhook.

15. **Add "Error page" node:**  
    - Type: Set  
    - Set "html" field with styled error page HTML for client error display.  
    - Connect Snowflake1 error output and Set HTML error output to this node.

16. **Connect "Error page" node to "Respond to Webhook" node:**  
    - On error flow, Respond to Webhook sends error page HTML.

17. **Create and configure all Snowflake credentials:**  
    - Host, account, warehouse, database, schema, username, password.  
    - Test connections before use.

18. **Replace placeholders in SQL queries:**  
    - Replace `'YOUR_SCHEMA_NAME'` in DB Schema1 and Get table definition nodes with your actual schema name.

19. **Set up webhook endpoint and test:**  
    - Deploy workflow and test chat input and report generation by sending chat messages and querying via webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                             | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| ![5min Logo](https://res.cloudinary.com/de9jgixzm/image/upload/Skool%20Assets/ejm3hqnvhgwpnu2fv92s)  AI Agent to chat with Snowflake database with UI. Made by [Mark Shcherbakov](https://www.linkedin.com/in/marklowcoding/) from community [5minAI](https://www.skool.com/5minai-pro) | Project credits and branding.                                                                                       |
| Setup video (~5 min): [![Youtube Thumbnail](https://res.cloudinary.com/de9jgixzm/image/upload/nvg4dvgajspjzqudh2wa)](https://youtu.be/r7er-HCRsX4)                                                                                                       | Video instructions for setup and usage.                                                                            |
| Replace webhook URL in "Link to Report" node to your deployed webhook address for proper report linking.                                                                                                                                                 | Important for correct report navigation.                                                                            |
| Replace schema and database names in "DB Schema1" and "Get table definition" nodes to match your Snowflake environment.                                                                                                                                    | Critical to prevent query errors.                                                                                   |
| The HTML report uses React 17.0.2 and Chart.js loaded from CDN for interactive visualization; ensure internet connectivity or adapt to local hosting if needed.                                                                                           | External dependencies.                                                                                              |
| Supported SQL analytical functions in sub-workflow include GROUP BY, SUM, AVG, COUNT, MIN, MAX, MEDIAN, STDDEV, VARIANCE, PERCENTILE_CONT, MODE, TREND, and window functions.                                                                             | Enhances query capabilities for advanced analytics.                                                                |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, adhering strictly to all current content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.