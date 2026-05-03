Generate SQL queries from schema only - AI-powered

https://n8nworkflows.xyz/workflows/generate-sql-queries-from-schema-only---ai-powered-2508


# Generate SQL queries from schema only - AI-powered

### 1. Workflow Overview

This workflow enables secure, AI-powered generation and execution of SQL queries based solely on a database schema, without exposing actual data to the AI agent. It is designed for scenarios requiring data confidentiality while still leveraging AI to interact with the database structure and respond to user queries.

The workflow is logically divided into two main blocks:

- **1.1 Schema Extraction & Storage (Run Once)**  
  This initial setup block connects to a remote MySQL database, extracts the schema of all tables, enriches each schema with its table name, converts the schema data to a binary JSON file, and saves it locally. This block prepares the static database structure for efficient reuse without repeated remote calls.

- **1.2 Chat Interaction & Query Execution (Run Per User Interaction)**  
  This block is triggered on every chat message. It loads the locally stored schema file, combines it with the user’s chat input, and sends both to a LangChain AI Agent configured to generate SQL queries only from the schema. The workflow conditionally executes the generated SQL query in the database if present, formats the query results, and returns both the AI-generated answer and the query output to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Schema Extraction & Storage (Run Once)

**Overview:**  
This block retrieves the full database schema from a remote MySQL server, processes it, and saves it locally as a JSON file. This operation is intended to run once or when the schema is updated, avoiding repeated remote database calls and improving performance.

**Nodes Involved:**  
- When clicking "Test workflow" (Manual Trigger)  
- List all tables in a database (MySQL)  
- Extract database schema (MySQL)  
- Add table name to output (Set)  
- Convert data to binary (ConvertToFile)  
- Save file locally (ReadWriteFile)  
- Sticky Note (instructional notes)

**Node Details:**

- **When clicking "Test workflow"**  
  - Type: Manual trigger  
  - Role: Entry point for starting the schema extraction process manually  
  - Config: No parameters  
  - Connections: Outputs to "List all tables in a database"  
  - Failure Modes: None specific, manual trigger

- **List all tables in a database**  
  - Type: MySQL node  
  - Role: Executes `SHOW TABLES;` to list all tables in the remote database  
  - Credentials: Uses configured MySQL credentials (db4free TTT account)  
  - Output: List of tables used downstream  
  - Failure Modes: Connection errors, authentication failure, query syntax errors

- **Extract database schema**  
  - Type: MySQL node  
  - Role: For each table from previous node, runs `DESCRIBE <table>;` to get schema details  
  - Query Expression: `DESCRIBE {{ $json.Tables_in_tttytdb2023 }};` dynamically uses table names  
  - Failure Modes: Table not found, connection errors

- **Add table name to output**  
  - Type: Set node  
  - Role: Adds a new field `table` to the output containing the current table name  
  - Expression: `={{ $('List all tables in a database').item.json.Tables_in_tttytdb2023 }}`  
  - Purpose: To associate schema rows with their table names

- **Convert data to binary**  
  - Type: ConvertToFile node  
  - Role: Converts the schema JSON data into a binary JSON file format for saving  
  - Operation: toJson

- **Save file locally**  
  - Type: ReadWriteFile node  
  - Role: Writes the binary JSON schema data to local file `./chinook_mysql.json`  
  - Operation: write  
  - Failure Modes: File permission errors, disk space issues

- **Sticky Notes**  
  - Provide context and setup instructions, including database setup prerequisites and explanation of the schema extraction process.

**Summary:**  
This block prepares the static database schema once, enabling the rest of the workflow to operate efficiently without remote database dependency.

---

#### 2.2 Chat Interaction & Query Execution (Run Per User Interaction)

**Overview:**  
This block handles user chat input, leverages the locally stored schema to generate AI responses and SQL queries, conditionally executes queries on the database, and returns combined results to the user. It enforces data confidentiality by never exposing actual data to the AI agent memory.

**Nodes Involved:**  
- Chat Trigger (LangChain chatTrigger)  
- Load the schema from the local file (ReadWriteFile)  
- Extract data from file (ExtractFromFile)  
- Combine schema data and chat input (Set)  
- AI Agent (LangChain agent)  
- OpenAI Chat Model (LangChain lmChatOpenAi)  
- Window Buffer Memory (LangChain memoryBufferWindow)  
- Extract SQL query (Set)  
- Check if query exists (If)  
- Run SQL query (MySQL)  
- Format query results (Set)  
- Combine query result and chat answer (Merge)  
- Prepare final output (Set)  
- No Operation, do nothing (NoOp)  
- Sticky Notes (instructional and explanatory)

**Node Details:**

- **Chat Trigger**  
  - Type: LangChain chatTrigger  
  - Role: Webhook that receives user chat messages to start the workflow  
  - Parameters: Default  
  - Failure Modes: Webhook access issues, malformed input

- **Load the schema from the local file**  
  - Type: ReadWriteFile  
  - Role: Loads the previously saved schema file `./chinook_mysql.json` locally  
  - Operation: read  
  - Failure Modes: File not found, read permission errors

- **Extract data from file**  
  - Type: ExtractFromFile  
  - Role: Converts the binary JSON file content back into usable JSON data  
  - Operation: fromJson

- **Combine schema data and chat input**  
  - Type: Set  
  - Role: Creates the input payload for the AI Agent by combining:  
    - `sessionId`, `action`, `chatinput` from chat trigger  
    - `schema` stringified from the loaded JSON schema  
  - This node ensures the AI Agent has both the user's question and the database schema context.

- **AI Agent**  
  - Type: LangChain agent  
  - Role: Processes user input and schema to generate an AI response and potentially an SQL query  
  - Agent Type: conversationalAgent  
  - Prompt: Includes database schema and user request in the text prompt.  
  - System Message: AI is instructed to generate SQL queries only from schema, not to execute them.  
  - Human Message: Instructions for supported tools and formatting for SQL query generation.  
  - Failure Modes: API errors, prompt formatting errors, rate limits

- **OpenAI Chat Model**  
  - Type: LangChain lmChatOpenAi  
  - Role: The underlying model used by the AI Agent (GPT-4o) with temperature 0.2 for controlled responses  
  - Credentials: OpenAI API key configured  
  - Failure Modes: API key issues, network errors, API limits

- **Window Buffer Memory**  
  - Type: LangChain memoryBufferWindow  
  - Role: Maintains conversational context limited to last 10 messages without storing actual data values  
  - Failure Modes: Memory overflow (unlikely due to window length), data privacy ensured

- **Extract SQL query**  
  - Type: Set  
  - Role: Parses AI Agent’s output text to extract SQL query using a regex matching `SELECT ...;`  
  - Expression: `={{ ($json.output.match(/SELECT[\s\S]*?;/i) || [])[0] || "" }}`  
  - Failure Modes: No query found, malformed output, regex mismatch

- **Check if query exists**  
  - Type: If  
  - Role: Branches workflow depending on whether an SQL query was extracted  
  - Condition: `query` string is not empty  
  - Failure Modes: Logic errors if field missing

- **Run SQL query**  
  - Type: MySQL node  
  - Role: Executes the extracted SQL query on the remote MySQL database  
  - Query: Dynamic `{{ $json.query }}`  
  - Credentials: MySQL credentials (db4free TTT account)  
  - Failure Modes: Query errors, connection failure, SQL injection risk mitigated by AI generation only

- **Format query results**  
  - Type: Set  
  - Role: Reformats the SQL results into a readable markdown table string  
  - Expression:  
    ```  
    {{ Object.keys($jmespath($input.all(),'[].json')[0]).join(' | ') }}  
    {{ ($jmespath($input.all(),'[].json')).map(obj => Object.values(obj).join(' | ')).join('\n') }}  
    ```  
  - Failure Modes: Empty results, data formatting errors

- **Combine query result and chat answer**  
  - Type: Merge (combine)  
  - Role: Combines the AI Agent’s original text output and the formatted SQL query results into a single output  
  - Mode: combine by position

- **Prepare final output**  
  - Type: Set  
  - Role: Constructs the final chat message including the AI answer and the SQL query result in markdown formatting  
  - Expression:  
    ```  
    {{ $json.output }}  
  
    SQL result:  
    ```markdown  
    {{ $json.sqloutput }}  
    ```  
    ```  
  - Failure Modes: Output concatenation errors

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Pass-through node when no SQL query is generated, providing immediate AI answer without query execution

- **Sticky Notes**  
  - Provide detailed explanations of chat message handling, AI Agent behavior, SQL extraction logic, and data privacy assurances.

---

### 3. Summary Table

| Node Name                         | Node Type                      | Functional Role                                         | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                      |
|----------------------------------|--------------------------------|--------------------------------------------------------|----------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking "Test workflow"     | Manual Trigger                 | Manual start for schema extraction                      | -                                | List all tables in a database        |                                                                                                |
| List all tables in a database     | MySQL                         | Fetches list of tables from remote DB                   | When clicking "Test workflow"     | Extract database schema              |                                                                                                |
| Extract database schema           | MySQL                         | Retrieves schema details for each table                 | List all tables in a database     | Add table name to output             |                                                                                                |
| Add table name to output          | Set                           | Adds table name field to schema data                    | Extract database schema           | Convert data to binary               |                                                                                                |
| Convert data to binary            | ConvertToFile                 | Converts JSON schema to binary file format              | Add table name to output          | Save file locally                   |                                                                                                |
| Save file locally                 | ReadWriteFile                 | Saves binary schema file locally                         | Convert data to binary            | -                                   | ## Run this part only once (covers entire schema extraction block)                              |
| Chat Trigger                     | LangChain chatTrigger          | Receives user chat messages                             | -                                | Load the schema from the local file |                                                                                                |
| Load the schema from the local file | ReadWriteFile              | Reads locally saved schema file                          | Chat Trigger                    | Extract data from file               |                                                                                                |
| Extract data from file            | ExtractFromFile               | Converts binary file content to JSON                    | Load the schema from the local file | Combine schema data and chat input |                                                                                                |
| Combine schema data and chat input | Set                         | Combines schema and chat input for AI Agent            | Extract data from file            | AI Agent                           | ## On every chat message (covers entire chat input processing block)                           |
| AI Agent                         | LangChain agent               | Generates AI response and SQL query from schema & input | Combine schema data and chat input | Extract SQL query                  | ### LangChain AI Agent's system prompt is modified (covers AI Agent behavior)                  |
| OpenAI Chat Model                | LangChain lmChatOpenAi         | Provides GPT-4o model for AI Agent                      | AI Agent (ai_languageModel input) | AI Agent                         |                                                                                                |
| Window Buffer Memory             | LangChain memoryBufferWindow   | Maintains conversational context without data values  | AI Agent (ai_memory input)        | AI Agent                         |                                                                                                |
| Extract SQL query                | Set                           | Extracts SQL query from AI Agent response               | AI Agent                        | Check if query exists               | ## SQL query extraction                                                                        |
| Check if query exists            | If                            | Branches based on presence of SQL query                 | Extract SQL query                | Run SQL query / Combine query result and chat answer / No Operation |                                                                                                |
| Run SQL query                   | MySQL                         | Executes SQL query on database                           | Check if query exists (true)     | Format query results               | - The SQL node accesses the database and executes the query. Results are formatted and displayed. |
| Format query results            | Set                           | Formats SQL results into readable markdown table        | Run SQL query                   | Combine query result and chat answer |                                                                                                |
| Combine query result and chat answer | Merge                    | Merges AI answer and SQL query result                    | Format query results, Check if query exists (false) | Prepare final output             |                                                                                                |
| Prepare final output            | Set                           | Prepares final chat output combining AI response and SQL results | Combine query result and chat answer | -                             |                                                                                                |
| No Operation, do nothing        | NoOp                          | Pass-through when no SQL query is present               | Check if query exists (false)     | -                                   |                                                                                                |
| Sticky Note                    | Sticky Note                   | Various instructional notes                             | -                                | -                                   | Multiple notes throughout workflow covering setup, execution, and processing logic             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking "Test workflow"  
   - Type: Manual Trigger  
   - Purpose: To manually start schema extraction

2. **Create MySQL Node to List Tables**  
   - Name: List all tables in a database  
   - Type: MySQL  
   - Credentials: Configure MySQL credentials for your remote database  
   - Query: `SHOW TABLES;`  
   - Connect output of Manual Trigger to this node

3. **Create MySQL Node to Extract Schema**  
   - Name: Extract database schema  
   - Type: MySQL  
   - Credentials: Same MySQL credentials  
   - Query: `DESCRIBE {{ $json.Tables_in_<your_database_name> }};` (replace placeholder with actual field)  
   - Connect output of "List all tables in a database" to this node

4. **Create Set Node to Add Table Name**  
   - Name: Add table name to output  
   - Type: Set  
   - Add field `table` with value: `={{ $('List all tables in a database').item.json.Tables_in_<your_database_name> }}`  
   - Connect output of "Extract database schema" to this node

5. **Create ConvertToFile Node**  
   - Name: Convert data to binary  
   - Type: ConvertToFile  
   - Operation: toJson  
   - Connect output of "Add table name to output" to this node

6. **Create ReadWriteFile Node to Save Schema**  
   - Name: Save file locally  
   - Type: ReadWriteFile  
   - Operation: write  
   - File Name: `./chinook_mysql.json` (ensure write permissions)  
   - Connect output of "Convert data to binary" to this node

7. **Create LangChain Chat Trigger Node**  
   - Name: Chat Trigger  
   - Type: LangChain chatTrigger  
   - Configure webhook ID, exposing it for external chat input  
   - This node starts the chat interaction workflow

8. **Create ReadWriteFile Node to Load Schema**  
   - Name: Load the schema from the local file  
   - Type: ReadWriteFile  
   - Operation: read  
   - File Selector: `./chinook_mysql.json`  
   - Connect output of "Chat Trigger" to this node

9. **Create ExtractFromFile Node**  
   - Name: Extract data from file  
   - Type: ExtractFromFile  
   - Operation: fromJson  
   - Connect output of "Load the schema from the local file" to this node

10. **Create Set Node to Combine Schema and Chat Input**  
    - Name: Combine schema data and chat input  
    - Type: Set  
    - Assign fields:  
      - `sessionId`: `={{ $('Chat Trigger').first().json.sessionId }}`  
      - `action`: `={{ $('Chat Trigger').first().json.action }}`  
      - `chatinput`: `={{ $('Chat Trigger').first().json.chatInput }}`  
      - `schema`: `={{ $json.data }}` (stringified schema JSON)  
    - Connect output of "Extract data from file" to this node

11. **Create LangChain Agent Node**  
    - Name: AI Agent  
    - Type: LangChain agent  
    - Agent: conversationalAgent  
    - Prompt Type: define  
    - Text:  
      ```
      Here is the database schema: {{ $json.schema }}
      Here is the user request: {{ $('Chat Trigger').item.json.chatInput }}
      ```  
    - System Message: Instruct agent to generate SQL queries using schema only, not execute them.  
    - Human Message: Provide instructions for tools and formatting, emphasizing queries must be wrapped in triple quotes.  
    - Connect output of "Combine schema data and chat input" to this node

12. **Create OpenAI Chat Model Node**  
    - Name: OpenAI Chat Model  
    - Type: LangChain lmChatOpenAi  
    - Model: GPT-4o  
    - Options: temperature 0.2  
    - Credentials: OpenAI API credentials  
    - Connect as AI language model input to "AI Agent"

13. **Create Window Buffer Memory Node**  
    - Name: Window Buffer Memory  
    - Type: LangChain memoryBufferWindow  
    - Context Window Length: 10  
    - Connect as AI memory input to "AI Agent"

14. **Create Set Node to Extract SQL Query**  
    - Name: Extract SQL query  
    - Type: Set  
    - Assignment:  
      - `query`: `={{ ($json.output.match(/SELECT[\s\S]*?;/i) || [])[0] || "" }}`  
    - Connect output of "AI Agent" to this node

15. **Create If Node to Check if Query Exists**  
    - Name: Check if query exists  
    - Type: If  
    - Condition: `query` is not empty string  
    - Connect output of "Extract SQL query" to this node

16. **Create MySQL Node to Run SQL Query**  
    - Name: Run SQL query  
    - Type: MySQL  
    - Credentials: MySQL credentials (same as before)  
    - Query: `{{ $json.query }}`  
    - Connect "true" output of "Check if query exists" to this node

17. **Create Set Node to Format Query Results**  
    - Name: Format query results  
    - Type: Set  
    - Assignment:  
      ```
      sqloutput: ={{ Object.keys($jmespath($input.all(),'[].json')[0]).join(' | ') }} 
      {{ ($jmespath($input.all(),'[].json')).map(obj => Object.values(obj).join(' | ')).join('\n') }}
      ```  
    - Connect output of "Run SQL query" to this node

18. **Create Merge Node to Combine Query Result and Chat Answer**  
    - Name: Combine query result and chat answer  
    - Type: Merge  
    - Mode: combine  
    - Combine By: position  
    - Connect output of "Format query results" to input 1  
    - Connect "false" output of "Check if query exists" to input 2 (bypassing SQL)  

19. **Create Set Node to Prepare Final Output**  
    - Name: Prepare final output  
    - Type: Set  
    - Assignment:  
      ```
      output: ={{ $json.output }}

      SQL result:
      ```markdown
      {{ $json.sqloutput }}
      ```
      ```  
    - Connect output of "Combine query result and chat answer" to this node

20. **Create No Operation Node**  
    - Name: No Operation, do nothing  
    - Type: NoOp  
    - Connect "false" output of "Check if query exists" to this node as alternative to SQL execution path  

21. **Link Node Outputs Appropriately**  
    - Ensure all node connections match the original workflow order and logic as described above.

22. **Add Sticky Notes (Optional)**  
    - Add instructional sticky notes in key areas for clarity, referencing setup instructions, data privacy, and AI agent behavior.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                               | Context or Link                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| The workflow uses a free MySQL hosting service [db4free.net](https://db4free.net/signup.php) for demonstration purposes. Setup requires importing a sample database such as Chinook.                           | Setup instructions in sticky notes and the tutorial linked in the description.                                           |
| The Chinook sample database schema for MySQL is available on [GitHub](https://github.com/msimanga/chinook/tree/master/mysql).                                                                              | Recommended data source for testing.                                                                                      |
| This workflow is based on the template: [Create an SQL agent with LangChain and SQLite](https://n8n.io/workflows/2292-talk-to-your-sqlite-database-with-a-langchain-ai-agent/).                              | Original workflow inspiration.                                                                                            |
| The AI Agent memory stores only schema, user questions, and AI replies — no actual data values from executed queries are stored, preserving confidentiality.                                              | Data privacy principle explained in sticky notes.                                                                        |
| The AI system prompt is customized to generate queries only from schema, never executing them itself. Queries are wrapped in triple quotes in AI outputs for consistent extraction.                        | Critical for safe operation and clear query extraction.                                                                  |
| The workflow demonstrates how to leverage LangChain AI agents with n8n to interact with database schemas securely, supporting advanced conversational querying scenarios.                                  | Workflow use case and advanced application.                                                                               |
| For further database comparison and setup, refer to the tutorial: [Compare databases with n8n](https://blog.n8n.io/compare-databases/).                                                                      | Setup and background information.                                                                                        |

---

This detailed reference document provides complete insight into the workflow’s structure, node configurations, logical flow, and reproduction steps, ensuring both human users and automation agents can understand, maintain, and extend it confidently.