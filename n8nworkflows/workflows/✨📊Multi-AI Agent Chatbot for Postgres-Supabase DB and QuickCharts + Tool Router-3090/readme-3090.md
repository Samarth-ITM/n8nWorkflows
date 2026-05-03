✨📊Multi-AI Agent Chatbot for Postgres/Supabase DB and QuickCharts + Tool Router

https://n8nworkflows.xyz/workflows/---multi-ai-agent-chatbot-for-postgres-supabase-db-and-quickcharts---tool-router-3090


# ✨📊Multi-AI Agent Chatbot for Postgres/Supabase DB and QuickCharts + Tool Router

### 1. Workflow Overview

This workflow implements a **Multi-AI Agent Chatbot** designed for interactive querying and visualization of data stored in Postgres or Supabase databases. It targets data analysts, developers, and BI teams who want to interact with their databases using natural language and generate dynamic charts without manual SQL or chart configuration.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception and Primary AI Management**  
  Captures user chat messages and manages the overall AI-driven conversation flow, routing requests to appropriate tools.

- **1.2 Tool Agent Router**  
  Routes the user request to either the database querying agent or the chart generation agent based on the request type.

- **1.3 Database Querying Block**  
  Handles SQL query execution, schema retrieval, and table definitions using Postgres tools and a dedicated AI agent.

- **1.4 Chart Generation Block**  
  Generates Chart.js-compatible JSON configurations using AI, converts them into QuickChart URLs, and fetches the chart images.

- **1.5 Memory Integration**  
  Maintains chat history in a Postgres table to enable context-aware conversations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Primary AI Management

**Overview:**  
This block receives chat messages from users and manages the primary AI agent that interprets the input and decides which tool to invoke (database query or chart generation).

**Nodes Involved:**  
- When chat message received  
- 🤖Primary Agent  
- query_db_tool (Tool Workflow)  
- generate_quickchart_tool (Tool Workflow)  
- Postgres Chat Memory  
- gpt-4o-mini (Language Model)  
- Sticky Note (Primary AI Manager Agent)

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Entry point for chat messages via webhook  
  - Config: Default options, listens for chat input  
  - Input: External chat message  
  - Output: Passes chat input to Primary Agent  
  - Failures: Webhook connectivity, malformed input

- **🤖Primary Agent**  
  - Type: Langchain Agent  
  - Role: Central AI manager that interprets user input and selects tools  
  - Config: System message defines available tools: `query_database_tool` and `generate_chart_tool`  
  - Input: Chat input from "When chat message received"  
  - Output: Routes to tool workflows based on AI decision  
  - Failures: AI model errors, tool invocation failures

- **query_db_tool**  
  - Type: Langchain Tool Workflow  
  - Role: Invokes the database query workflow within the same main workflow  
  - Config: Passes user prompt and route identifier `"query_database_tool"`  
  - Input: From Primary Agent  
  - Output: Database query results

- **generate_quickchart_tool**  
  - Type: Langchain Tool Workflow  
  - Role: Invokes the chart generation workflow  
  - Config: Passes user prompt, route `"generate_chart_tool"`, and database records (from AI override)  
  - Input: From Primary Agent  
  - Output: Chart JSON configuration and URL

- **Postgres Chat Memory**  
  - Type: Langchain Postgres Chat Memory  
  - Role: Stores chat history in a Postgres table named after the workflow ID  
  - Config: Uses Postgres credentials  
  - Input: Chat messages and AI responses  
  - Output: Provides context for AI agents  
  - Failures: DB connectivity, table schema issues

- **gpt-4o-mini**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Language model used by Primary Agent  
  - Config: Model set to `gpt-4o-mini`, response format text  
  - Input: Prompts from Primary Agent  
  - Output: AI-generated text responses  
  - Failures: API key issues, rate limits

- **Sticky Note (Primary AI Manager Agent)**  
  - Role: Documentation note indicating this block’s purpose

---

#### 2.2 Tool Agent Router

**Overview:**  
This block routes the workflow execution to either the database querying agent or the chart generation agent based on the `route` field in the input.

**Nodes Involved:**  
- When Executed by Another Workflow  
- 🔀Tool Agent Router  
- 🤖Secondary Postgres Agent  
- 🤖Secondary QuickChart Agent  
- Sticky Note (Tool Agent Router)

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point when this workflow is called as a tool from Primary Agent  
  - Config: Accepts inputs `user_prompt`, `route`, and `db_records`  
  - Input: From tool invocation  
  - Output: Passes data to Tool Agent Router

- **🔀Tool Agent Router**  
  - Type: Switch Node  
  - Role: Routes execution based on `route` value (`query_database_tool` or `generate_chart_tool`)  
  - Config: Two outputs named "🔍query" and "📊chart"  
  - Input: From "When Executed by Another Workflow"  
  - Output: Routes to Secondary Postgres Agent or Secondary QuickChart Agent  
  - Failures: Missing or invalid route value

- **🤖Secondary Postgres Agent**  
  - Type: Langchain Agent  
  - Role: Handles SQL database querying tasks  
  - Config: System message instructs to use SQL querying tools  
  - Input: Routed from Tool Agent Router when route = `query_database_tool`  
  - Output: SQL query results or schema info  
  - Failures: SQL errors, AI prompt errors

- **🤖Secondary QuickChart Agent**  
  - Type: Langchain Agent  
  - Role: Generates Chart.js JSON configurations from DB records and user prompt  
  - Config: Custom prompt specifying chart type, labels, and data extraction rules; output parsed as structured JSON  
  - Input: Routed from Tool Agent Router when route = `generate_chart_tool`  
  - Output: Chart JSON configuration  
  - Failures: Parsing errors, AI model errors

- **Sticky Note (Tool Agent Router)**  
  - Role: Documentation note for this routing block

---

#### 2.3 Database Querying Block

**Overview:**  
This block executes SQL queries, retrieves database schema and table definitions, and supports the Secondary Postgres Agent with necessary tools.

**Nodes Involved:**  
- Execute SQL Query  
- DB Schema and Tables  
- Table Definitions  
- gpt-40-mini-1 (Language Model)  
- Sticky Notes (Postgres Tool, Postgres Tools Used)

**Node Details:**

- **Execute SQL Query**  
  - Type: Postgres Tool Node  
  - Role: Executes SQL queries generated by AI  
  - Config: SQL query is dynamically injected from AI output (`$fromAI("sql_query")`)  
  - Credentials: Postgres account  
  - Input: From Secondary Postgres Agent  
  - Output: Query results as JSON  
  - Failures: SQL syntax errors, connection issues

- **DB Schema and Tables**  
  - Type: Postgres Tool Node  
  - Role: Retrieves list of all tables and schemas in the database  
  - Config: Static SQL query on `information_schema.tables` filtering out system schemas  
  - Credentials: Postgres account  
  - Input: From Secondary Postgres Agent as needed  
  - Output: List of tables and schemas  
  - Failures: DB permission errors

- **Table Definitions**  
  - Type: Postgres Tool Node  
  - Role: Retrieves detailed column and constraint info for a given table and schema  
  - Config: SQL query with parameters injected from AI (`$fromAI("table_name")`, `$fromAI("schema_name")`)  
  - Credentials: Postgres account  
  - Input: From Secondary Postgres Agent  
  - Output: Table column metadata and foreign key info  
  - Failures: Missing parameters, SQL errors

- **gpt-40-mini-1**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Language model used by Secondary Postgres Agent  
  - Config: Model `gpt-4o-mini`, text response format  
  - Input: Prompts from Secondary Postgres Agent  
  - Output: SQL queries or schema explanations  
  - Failures: API errors

- **Sticky Notes (Postgres Tool, Postgres Tools Used)**  
  - Role: Documentation for database querying tools and usage

---

#### 2.4 Chart Generation Block

**Overview:**  
This block generates chart configurations using AI, converts them into QuickChart URLs, and fetches the chart images.

**Nodes Involved:**  
- 🤖Secondary QuickChart Agent  
- QuickChart Object Schema  
- QuickChart GET URL  
- Create QuickChart  
- Final QuickChart URL  
- gpt-4o-mini-2 (Language Model)  
- Sticky Notes (QuickChart Tool, QuickChart Schema, Chart Size, Generate a Quickchart)

**Node Details:**

- **🤖Secondary QuickChart Agent**  
  - Type: Langchain Agent  
  - Role: Generates Chart.js JSON config based on user prompt and DB records  
  - Config: Custom prompt specifying chart type, labels, and data extraction rules; output parsed with `QuickChart Object Schema`  
  - Input: Routed from Tool Agent Router  
  - Output: Chart JSON object

- **QuickChart Object Schema**  
  - Type: Langchain Output Parser (Structured)  
  - Role: Parses AI output into structured JSON matching Chart.js schema  
  - Config: Example JSON schema provided for bar chart with labels, datasets, and options  
  - Input: From Secondary QuickChart Agent  
  - Output: Validated JSON chart config

- **QuickChart GET URL**  
  - Type: Set Node  
  - Role: Constructs QuickChart.io URL embedding the JSON chart config  
  - Config: URL template `https://quickchart.io/chart?width=250&height=150&chart=` + encoded JSON string  
  - Input: From QuickChart Object Schema output  
  - Output: URL string

- **Create QuickChart**  
  - Type: HTTP Request Node  
  - Role: Fetches the chart image or data from QuickChart using the generated URL  
  - Config: URL dynamically set from previous node output  
  - Input: From QuickChart GET URL  
  - Output: Chart image or URL response

- **Final QuickChart URL**  
  - Type: Set Node  
  - Role: Stores the final encoded QuickChart URL in a variable `quickchart_url` for output or further use  
  - Input: From Create QuickChart  
  - Output: Final URL string

- **gpt-4o-mini-2**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Language model used by Secondary QuickChart Agent  
  - Config: Model `gpt-4o-mini`, text response format  
  - Input: Prompts from Secondary QuickChart Agent  
  - Output: Chart JSON config  
  - Failures: Parsing errors, API errors

- **Sticky Notes (QuickChart Tool, QuickChart Schema, Chart Size, Generate a Quickchart)**  
  - Role: Documentation and configuration hints for chart generation

---

#### 2.5 Memory Integration

**Overview:**  
Maintains chat history in a Postgres table to provide context for AI agents, enabling more coherent and context-aware conversations.

**Nodes Involved:**  
- Postgres Chat Memory

**Node Details:**

- **Postgres Chat Memory**  
  - Type: Langchain Postgres Chat Memory  
  - Role: Stores and retrieves chat history from a Postgres table named after the workflow ID  
  - Config: Uses Postgres credentials  
  - Input: Chat messages and AI responses from Primary Agent  
  - Output: Provides context for AI language models  
  - Failures: Database connectivity, schema mismatch

---

### 3. Summary Table

| Node Name                   | Node Type                               | Functional Role                          | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                     |
|-----------------------------|---------------------------------------|----------------------------------------|------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received  | Langchain Chat Trigger                 | Entry point for user chat input        | External webhook             | 🤖Primary Agent                |                                                                                                |
| 🤖Primary Agent             | Langchain Agent                       | Main AI manager, routes requests       | When chat message received   | query_db_tool, generate_quickchart_tool | ## 🤖Primary AI Manager Agent                                                                   |
| query_db_tool               | Langchain Tool Workflow               | Invokes DB query workflow               | 🤖Primary Agent              | When Executed by Another Workflow |                                                                                                |
| generate_quickchart_tool    | Langchain Tool Workflow               | Invokes chart generation workflow       | 🤖Primary Agent              | When Executed by Another Workflow |                                                                                                |
| Postgres Chat Memory        | Langchain Postgres Chat Memory       | Stores chat history                     | 🤖Primary Agent              | 🤖Primary Agent                |                                                                                                |
| gpt-4o-mini                | Langchain OpenAI Chat Model           | Language model for Primary Agent        | 🤖Primary Agent              | 🤖Primary Agent                |                                                                                                |
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry point for tool workflows          | query_db_tool, generate_quickchart_tool | 🔀Tool Agent Router           |                                                                                                |
| 🔀Tool Agent Router         | Switch Node                          | Routes to DB or Chart agent             | When Executed by Another Workflow | 🤖Secondary Postgres Agent, 🤖Secondary QuickChart Agent | ## Tool Agent Router                                                                           |
| 🤖Secondary Postgres Agent  | Langchain Agent                      | Handles DB querying                      | 🔀Tool Agent Router          | Execute SQL Query, DB Schema and Tables, Table Definitions | ## ⚒️🤖Secondary Postgres Tool Agent                                                           |
| Execute SQL Query           | Postgres Tool Node                   | Executes SQL queries                     | 🤖Secondary Postgres Agent   | 🤖Secondary Postgres Agent     | ## Postgres Tool                                                                              |
| DB Schema and Tables        | Postgres Tool Node                   | Retrieves DB schema and tables           | 🤖Secondary Postgres Agent   | 🤖Secondary Postgres Agent     | ## Postgres Tool                                                                              |
| Table Definitions           | Postgres Tool Node                   | Retrieves table column and constraint info | 🤖Secondary Postgres Agent   | 🤖Secondary Postgres Agent     | ## Postgres Tool                                                                              |
| gpt-40-mini-1               | Langchain OpenAI Chat Model           | Language model for Secondary Postgres Agent | 🤖Secondary Postgres Agent   | 🤖Secondary Postgres Agent     | ## LLM                                                                                        |
| 🤖Secondary QuickChart Agent | Langchain Agent                      | Generates Chart.js JSON config           | 🔀Tool Agent Router          | QuickChart Object Schema       | ## ⚒️🤖Secondary QuickChart Tool Agent                                                        |
| QuickChart Object Schema    | Langchain Output Parser Structured   | Parses AI output into Chart.js JSON      | 🤖Secondary QuickChart Agent | QuickChart GET URL            | ## QuickChart Tool                                                                           |
| QuickChart GET URL          | Set Node                            | Constructs QuickChart.io URL             | QuickChart Object Schema     | Create QuickChart             | ## Chart Size                                                                                |
| Create QuickChart           | HTTP Request Node                   | Fetches chart image from QuickChart      | QuickChart GET URL           | Final QuickChart URL          | ## Generate a Quickchart                                                                     |
| Final QuickChart URL        | Set Node                            | Stores final QuickChart URL              | Create QuickChart            | Output or further use         |                                                                                                |
| gpt-4o-mini-2               | Langchain OpenAI Chat Model           | Language model for Secondary QuickChart Agent | 🤖Secondary QuickChart Agent | 🤖Secondary QuickChart Agent  | ## LLM                                                                                        |
| Sticky Notes (various)      | Sticky Note                         | Documentation and notes                  | N/A                         | N/A                          | Multiple notes covering setup, tools, and usage as detailed in the workflow description       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the "When chat message received" node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Purpose: Entry point for chat messages via webhook  
   - No special parameters needed.

3. **Add the Primary AI Agent node ("🤖Primary Agent")**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configure system message:  
     ```
     You are a helpful assistant that answers the users questions by using the tools provided.

     ## TOOLS
     - query_database_tool: Use this tool to query the database
     - generate_chart_tool: Use this tool to generate a chart with QuickChart

     Always provide the results of the database query and the link for the chart when applicable.
     ```
   - Connect input from "When chat message received".

4. **Add OpenAI credentials**  
   - Create credentials for OpenAI API (e.g., GPT-4o-mini model).  
   - Assign these credentials to the language model nodes.

5. **Add the Language Model node "gpt-4o-mini"**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `gpt-4o-mini`  
   - Response format: text  
   - Connect to Primary Agent as AI language model.

6. **Add Tool Workflow nodes for database query and chart generation**  
   - `query_db_tool`:  
     - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
     - Name: `query_database_tool`  
     - Workflow ID: same as current workflow  
     - Inputs: `user_prompt` from chat input, `route` set to `"query_database_tool"`  
   - `generate_quickchart_tool`:  
     - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
     - Name: `generate_chart_tool`  
     - Workflow ID: same as current workflow  
     - Inputs: `user_prompt` from chat input, `route` set to `"generate_chart_tool"`, `db_records` from AI override

7. **Connect Tool Workflow nodes as AI tools to Primary Agent.**

8. **Add Postgres Chat Memory node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryPostgresChat`  
   - Table name: use workflow ID + `_chat_history`  
   - Credentials: Postgres account  
   - Connect as AI memory to Primary Agent.

9. **Add "When Executed by Another Workflow" node**  
   - Type: `n8n-nodes-base.executeWorkflowTrigger`  
   - Inputs: `user_prompt`, `route`, `db_records`  
   - Connect outputs from Tool Workflow nodes to this node.

10. **Add "🔀Tool Agent Router" switch node**  
    - Type: Switch  
    - Configure two outputs:  
      - Output "🔍query" if `route == "query_database_tool"`  
      - Output "📊chart" if `route == "generate_chart_tool"`  
    - Connect input from "When Executed by Another Workflow".

11. **Add Secondary Postgres Agent node ("🤖Secondary Postgres Agent")**  
    - Type: Langchain Agent  
    - System message:  
      ```
      You are a helpful assistant with tools for querying a SQL database. Use the tools provided to query the database.
      ```  
    - Text input: `user_prompt` from router  
    - Connect output from "🔍query" output of router.

12. **Add Secondary QuickChart Agent node ("🤖Secondary QuickChart Agent")**  
    - Type: Langchain Agent  
    - Prompt:  
      ```
      Your task is to generate a Chart.js configuration object with the following specifications:
      - Chart type: bar unless otherwise indicated
      - Labels: Use the ML # from each record unless otherwise indicated
      - Show bar for list price if not otherwise indicated
      - Return only the raw JSON object without code fences or explanations

      This is the user prompt: {{ $json.user_prompt }}
      This is the result of the SQL query: {{ $json.db_records }}
      ```  
    - Enable output parser with structured JSON  
    - Connect output from "📊chart" output of router.

13. **Add OpenAI language model nodes for Secondary Agents**  
    - `gpt-40-mini-1` for Secondary Postgres Agent  
    - `gpt-4o-mini-2` for Secondary QuickChart Agent  
    - Configure both with OpenAI credentials and model `gpt-4o-mini`.

14. **Add Postgres Tool nodes for database querying:**  
    - **Execute SQL Query**  
      - Type: Postgres Tool  
      - Query: `{{ $fromAI("sql_query", "SQL Query") }}`  
      - Credentials: Postgres account  
      - Connect input from Secondary Postgres Agent  
    - **DB Schema and Tables**  
      - Type: Postgres Tool  
      - Query: Static SQL to list tables and schemas  
      - Credentials: Postgres account  
      - Connect input from Secondary Postgres Agent  
    - **Table Definitions**  
      - Type: Postgres Tool  
      - Query: SQL to get column and constraint info for a table, with parameters from AI (`table_name`, `schema_name`)  
      - Credentials: Postgres account  
      - Connect input from Secondary Postgres Agent

15. **Add QuickChart generation nodes:**  
    - **QuickChart Object Schema**  
      - Type: Langchain Output Parser Structured  
      - JSON schema example: Chart.js bar chart example JSON  
      - Connect input from Secondary QuickChart Agent  
    - **QuickChart GET URL**  
      - Type: Set Node  
      - Set field `url` to:  
        ```
        https://quickchart.io/chart?width=250&height=150&chart={{ $json.output.toJsonString() }}
        ```  
      - Connect input from QuickChart Object Schema  
    - **Create QuickChart**  
      - Type: HTTP Request  
      - URL: `={{ encodeURI($json.url) }}`  
      - Connect input from QuickChart GET URL  
    - **Final QuickChart URL**  
      - Type: Set Node  
      - Assign `quickchart_url` to encoded URL from Create QuickChart  
      - Connect input from Create QuickChart

16. **Connect outputs accordingly:**  
    - Secondary QuickChart Agent → QuickChart Object Schema → QuickChart GET URL → Create QuickChart → Final QuickChart URL  
    - Secondary Postgres Agent → Postgres Tool nodes → back to Secondary Postgres Agent

17. **Add Sticky Notes** for documentation and clarity at relevant positions.

18. **Configure credentials:**  
    - Postgres credentials with access to your database  
    - OpenAI API credentials for GPT-4o-mini model

19. **Test the workflow:**  
    - Trigger chat messages via webhook  
    - Use prompts like "Generate a bar chart of sales data" or "Show me all users in the database."

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow title and description: Multi-AI Agent Chatbot for Postgres/Supabase DB and QuickCharts + Tool Router | Workflow metadata and overview                       |
| Setup instructions: Create Postgres DB, add credentials, start chatting                                  | Sticky Note16                                        |
| Postgres tools used: Execute SQL Query, DB Schema and Tables, Table Definition                            | Sticky Note17                                        |
| QuickChart customization: Adjust schema and chart size                                                   | Sticky Note18, Sticky Note19, Sticky Note20         |
| Primary AI Manager Agent handles routing to tools                                                        | Sticky Note (Primary AI Manager Agent)               |
| Secondary agents specialize in DB querying and chart generation                                          | Sticky Note6 (Secondary Postgres Tool Agent), Sticky Note12 (Secondary QuickChart Tool Agent) |
| QuickChart documentation link: https://quickchart.io/documentation/                                     | Sticky Note18                                        |
| Workflow diagram and logical flow documented in node connections                                        | Workflow JSON metadata                               |

---

This structured documentation provides a comprehensive understanding of the workflow’s architecture, node configurations, and operational logic, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.