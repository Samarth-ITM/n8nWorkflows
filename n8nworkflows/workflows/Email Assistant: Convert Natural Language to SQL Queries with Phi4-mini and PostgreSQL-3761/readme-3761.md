Email Assistant: Convert Natural Language to SQL Queries with Phi4-mini and PostgreSQL

https://n8nworkflows.xyz/workflows/email-assistant--convert-natural-language-to-sql-queries-with-phi4-mini-and-postgresql-3761


# Email Assistant: Convert Natural Language to SQL Queries with Phi4-mini and PostgreSQL

---

## 1. Workflow Overview

This workflow, titled **"Email Assistant: Convert Natural Language to SQL Queries with Phi4-mini and PostgreSQL"**, is designed to bridge human and automated queries expressed in natural language into executable SQL statements targeting a PostgreSQL database, specifically containing structured email metadata.

It serves two primary use cases:

- **Interactive Querying:** Human users or robots can trigger this workflow via a chat interface or workflow trigger, pose natural language questions about email data, and receive SQL query results.
- **Automation Integration:** Acts as a sub-workflow or chat trigger to support other workflows that extract and store email information, making structured data accessible via flexible natural language queries.

### Logical Blocks

1.1 **Initial Schema Extraction and Storage**  
Loads tables and columns from the Postgres database, compiles the schema, converts it to a JSON file, and stores it locally on disk for quick reuse.

1.2 **Input Reception and Routing**  
Listens for natural language inputs via chat trigger, manual trigger, or workflow trigger, managing the input source and session data.

1.3 **Schema Reload and Conditional Cache Refresh**  
Loads the locally saved schema file and conditionally refreshes schema extraction if the file doesn’t exist or on manual retry.

1.4 **Prompt Preparation and AI Query Generation**  
Combines input query and database schema into a carefully engineered prompt to the Ollama `phi4-mini` LLM model for generating SQL statements.

1.5 **SQL Query Processing and Validation**  
Extracts and validates the raw SQL query from AI output, ensures trailing semicolon, and conditionally continues querying.

1.6 **SQL Execution and Result Formatting**  
Executes the validated SQL query against the PostgreSQL database and formats the results for output.

1.7 **Output Merging and Delivery**  
Combines the SQL text and the query result for final structured output to initiating channel.

---

## 2. Block-by-Block Analysis

---

### 1.1 Initial Schema Extraction and Storage

**Overview:**  
This block is responsible for introspecting the connected PostgreSQL database structure by listing tables and their columns, appending the table name, converting the result to JSON, and persisting the schema to a local file. This facilitates offline reuse and schema consistency.

**Nodes Involved:**  
- List all tables in a database  
- List all columns in a table  
- Add table name to output  
- Convert data to binary  
- Save file locally  
- If ran manually  
- Sticky Note (overview)

**Node Details:**

- **List all tables in a database**  
  - Type: PostgreSQL node executing a SQL query  
  - Configuration: Runs `SELECT table_name FROM INFORMATION_SCHEMA.TABLES WHERE table_schema='public'` to get all table names  
  - Connections: Output goes to "List all columns in a table"  
  - Edge cases: Database connectivity failures, authorization issues

- **List all columns in a table**  
  - Type: PostgreSQL node querying columns metadata  
  - Configuration: Queries `INFORMATION_SCHEMA.COLUMNS` filtering by current table name, fetching column name, data type, array status, and nullability  
  - Connections: Output to "Add table name to output"  
  - Edge cases: Cases when table name doesn't exist, or schema permissions missing

- **Add table name to output**  
  - Type: Set node  
  - Configuration: Adds `table` field carrying the table name, defaults to `'emails_metadata'` if none found  
  - Connections: Output to "Convert data to binary"  
  - Edge cases: Missing upstream data leading to fallback value being used

- **Convert data to binary**  
  - Type: ConvertToFile node  
  - Configuration: Converts incoming JSON data to JSON file content in binary format  
  - Connections: Output passes to "Save file locally"

- **Save file locally**  
  - Type: ReadWriteFile node  
  - Configuration: Writes converted binary content to local file path `/files/pgsql-{{ $workflow.id }}.json`  
  - Connections: Output feeds "If ran manually"

- **If ran manually**  
  - Type: If node  
  - Configuration: Checks if executed from manual "Test workflow" trigger; influences subsequent logic flow

- **Sticky Note (overview)**  
  - Provides a high-level explanation of this initial schema extraction section

---

### 1.2 Input Reception and Routing

**Overview:**  
This block handles natural language input reception from various entry points (chat interface, manual test trigger, or explicit workflow trigger) and prepares uniform data for downstream processing.

**Nodes Involved:**  
- Chat Trigger  
- WorkflowTrigger  
- When clicking "Test workflow" (manual trigger)  
- If ran manually  
- Sticky Note1 (overview)

**Node Details:**

- **Chat Trigger**  
  - Type: LangChain chatTrigger node  
  - Configuration: Listens for incoming chat-based prompts; output includes `chatInput`, `sessionId`, and `action` fields  
  - Connections: Output leads to "Load the schema from the local file"  
  - Edge cases: Webhook or connectivity issues, malformed input payloads

- **WorkflowTrigger**  
  - Type: ExecuteWorkflowTrigger node  
  - Configuration: Accepts `natural_language_query` input parameter from another workflow or API call  
  - Connections: Flows into "Load the schema from the local file"

- **When clicking "Test workflow"**  
  - Type: ManualTrigger node  
  - Configuration: Used for testing or manual runs; triggers schema extraction nodes when activated

- **If ran manually**  
  - Type: If node  
  - Configuration: Detects manual trigger execution to branch logic accordingly  
  - Inputs: From "Save file locally" and "When clicking Test workflow"  
  - Outputs: Triggers schema loading or refresh as needed

- **Sticky Note1**  
  - Describes this block’s role in reacting to triggers via chat or as a sub-workflow

---

### 1.3 Schema Reload and Conditional Cache Refresh

**Overview:**  
Loads previously saved database schema JSON file for use in AI prompt preparation. If the file does not exist, or on manual run retries, refreshes the schema extraction by starting over at the initial schema scan.

**Nodes Involved:**  
- Load the schema from the local file  
- If file exists or already retried generating it  
- Extract data from file  
- List all tables in a database

**Node Details:**

- **Load the schema from the local file**  
  - Type: ReadWriteFile node  
  - Configuration: Attempts to read the saved schema file `/files/pgsql-{{ $workflow.id }}.json` with an error strategy to continue  
  - Outputs data or empty if file missing  
  - Connections: Output leads to conditional node "If file exists or already retried generating it"

- **If file exists or already retried generating it**  
  - Type: If node  
  - Configuration: Checks if binary file data exists or workflow was manually triggered; controls whether schema extraction must be retried  
  - True path: Extracts data from file  
  - False path: Triggers fresh schema extraction by invoking "List all tables in a database"  
  - Edge cases: File permission errors, retry loops if file corrupted

- **Extract data from file**  
  - Type: ExtractFromFile node  
  - Configuration: Converts binary JSON data back to usable JSON structure for prompt preparation  
  - Connections: Output goes to "Combine schema data and chat input"

- **List all tables in a database**  
  - Mentioned here as the fallback schema extraction trigger on cache miss

---

### 1.4 Prompt Preparation and AI Query Generation

**Overview:**  
Combines the extracted schema with the natural language user input and context session info, then sends a carefully crafted prompt to the Ollama-powered `phi4-mini` LLM agent to generate a PostgreSQL query matching the user’s query.

**Nodes Involved:**  
- Combine schema data and chat input  
- AI Agent  
- Ollama Chat Model  
- Sticky Note2 (prompt engineering comment)

**Node Details:**

- **Combine schema data and chat input**  
  - Type: Set node  
  - Configuration: Creates a composite JSON object with fields:  
    - `sessionId`, `action` from Chat Trigger  
    - `chatinput` sourced either from WorkflowTrigger's input or Chat Trigger input  
    - `schema` content extracted from schema file  
  - Connections: Output sent to "AI Agent"

- **AI Agent**  
  - Type: LangChain Agent node  
  - Configuration: Uses custom prompt and system messages specifying:  
    - AI acts as an expert SQL generator for PostgreSQL, strictly adhering to supplied schema  
    - Rules defining how to interpret data types, metadata fields, and query construction  
    - Output shall be raw SQL query only, no explanations or code blocks  
  - Relies on the database schema JSON and natural language query fields  
  - Connections: Output goes to "Extract SQL query"

- **Ollama Chat Model**  
  - Type: LangChain Ollama Chat Model node  
  - Configuration: Uses model `phi4-mini:latest` locally on Ollama  
  - Receives prompt from "Combine schema data and chat input" indirectly  
  - Connections: Output passed to "AI Agent" for final SQL generation

- **Sticky Note2**  
  - Provides insight about the “refined prompt engineering” behind the AI Agent prompt, noting use of other AI assistant tools in preparation

---

### 1.5 SQL Query Processing and Validation

**Overview:**  
Extracts the raw SQL query from the AI Agent output, validates its syntax with basic heuristics (presence and trailing semicolon), and conditions flow based on validity for safe execution.

**Nodes Involved:**  
- Extract SQL query  
- Check for trailing semicolon  
- Add trailing semicolon  
- Check if query exists

**Node Details:**

- **Extract SQL query**  
  - Type: Set node  
  - Configuration: Uses regex to extract first SQL SELECT statement matching `SELECT ...` up to a semicolon from AI output; defaults to empty if none found  
  - Output: field `query`

- **Check for trailing semicolon**  
  - Type: If node  
  - Configuration: Checks if `query` string exists and does not end with `;`  
  - True path: Adds trailing semicolon  
  - False path: Proceeds to query existence check

- **Add trailing semicolon**  
  - Type: Set node  
  - Configuration: Appends semicolon to `query` field string

- **Check if query exists**  
  - Type: If node  
  - Configuration: Checks if `query` is a non-empty string, branching:  
    - If true: proceeds with SQL execution  
    - If false: sets empty output  
  - Edge cases: Empty or invalid AI output, prompt failures

---

### 1.6 SQL Execution and Result Formatting

**Overview:**  
Executes the validated SQL query on PostgreSQL and formats the rows into a readable string with headers and pipe delimiters to be returned to the invoker.

**Nodes Involved:**  
- Postgres  
- Format query results  
- Format empty output

**Node Details:**

- **Postgres**  
  - Type: PostgreSQL node  
  - Configuration: Executes the dynamic SQL query from `query` field  
  - On error: continues regular output (does not fail workflow)  
  - Output: Raw row data JSON

- **Format query results**  
  - Type: Set node  
  - Configuration: Transforms JSON rows into a pipe-delimited string:  
    - First line: column headers joined by " | "  
    - Subsequent lines: row values joined similarly separated by newline  
  - Output field: `sqloutput`

- **Format empty output**  
  - Type: Set node  
  - Configuration: Assures that `query` field is present even if empty for consistent downstream use  
  - Executed if no valid query existed

---

### 1.7 Output Merging and Delivery

**Overview:**  
Combines the SQL query text and formatted query output for comprehensive return payload, ready for downstream workflows or chat interface display.

**Nodes Involved:**  
- Combine query result and chat answer

**Node Details:**

- **Combine query result and chat answer**  
  - Type: Merge node  
  - Configuration: Combines input streams using “combine by position”, including unpaired items, merging `query` and `sqloutput` fields from prior nodes  
  - Ensures unified output with both query text and results

---

## 3. Summary Table

| Node Name                        | Node Type                     | Functional Role                              | Input Node(s)                 | Output Node(s)                   | Sticky Note                                               |
|---------------------------------|-------------------------------|----------------------------------------------|-------------------------------|---------------------------------|-----------------------------------------------------------|
| List all tables in a database    | Postgres                      | Retrieve all table names from DB              | When clicking "Test workflow"  | List all columns in a table      | See notes for table schema extraction overview            |
| List all columns in a table      | Postgres                      | Retrieve column info per table                 | List all tables in a database  | Add table name to output         |                                                           |
| Add table name to output         | Set                          | Add `table` field for schema context           | List all columns in a table    | Convert data to binary           |                                                           |
| Convert data to binary           | ConvertToFile                 | Convert JSON schema into binary file           | Add table name to output       | Save file locally                |                                                           |
| Save file locally                | ReadWriteFile                | Persist schema JSON file to local disk         | Convert data to binary         | If ran manually                 |                                                           |
| If ran manually                  | If                           | Detect manual execution trigger for flow control | Save file locally            | Load the schema from the local file |                                                           |
| Load the schema from the local file | ReadWriteFile              | Read saved local schema file into workflow     | Chat Trigger, WorkflowTrigger, If ran manually | If file exists or already retried generating it |                                                           |
| If file exists or already retried generating it | If               | Conditional to either load schema or refresh   | Load the schema from the local file | Extract data from file, List all tables in a database |                                                           |
| Extract data from file           | ExtractFromFile               | Parse binary to JSON schema                     | If file exists or already retried generating it | Combine schema data and chat input |                                                           |
| Combine schema data and chat input | Set                        | Combine input query, session, and schema data   | Extract data from file         | AI Agent                        |                                                           |
| AI Agent                        | LangChain Agent              | Generate SQL query text from prompt and schema | Combine schema data and chat input | Extract SQL query               | See prompt engineering notes                              |
| Ollama Chat Model               | LangChain Ollama Chat Model | Execute LLM query using phi4-mini model        | AI Agent                      | AI Agent (LLM output)            | See prompt engineering notes                              |
| Extract SQL query                | Set                          | Extract SQL query from AI output string         | AI Agent                      | Check for trailing semicolon     |                                                           |
| Check for trailing semicolon    | If                           | Validate SQL ends with semicolon, else add it  | Extract SQL query              | Add trailing semicolon, Check if query exists |                                                         |
| Add trailing semicolon          | Set                          | Append missing semicolon to SQL query           | Check for trailing semicolon  | Check if query exists            |                                                           |
| Check if query exists           | If                           | Branch based on if SQL query is non-empty       | Add trailing semicolon, Check for trailing semicolon | Combine query result and chat answer, Postgres |                                               |
| Postgres                       | Postgres                      | Execute received SQL query on PostgreSQL DB    | Check if query exists          | Format query results            |                                                           |
| Format query results            | Set                          | Format SQL result rows into readable string     | Postgres                      | Combine query result and chat answer |                                                           |
| Format empty output             | Set                          | Handle empty query cases consistently            | Check if query exists          | Combine query result and chat answer |                                                           |
| Combine query result and chat answer | Merge                    | Merge SQL query text and results for output     | Check if query exists, Format query results, Format empty output | (End)                      |                                                           |
| Chat Trigger                   | LangChain chatTrigger         | Receive natural language input via chat         | (none)                       | Load the schema from the local file | Sticky note1: Input trigger section overview             |
| WorkflowTrigger                | ExecuteWorkflowTrigger        | Receive natural language query as workflow input | (none)                     | Load the schema from the local file | Sticky note1: Input trigger section overview             |
| When clicking "Test workflow"  | ManualTrigger                 | Manual run trigger to start schema extraction    | (none)                       | List all tables in a database     | Sticky note: Initial schema extraction overview          |
| Sticky Note                   | StickyNote                    | Overview description of initial schema block     | (none)                       | (none)                         | See notes in Block 1.1                                   |
| Sticky Note1                  | StickyNote                    | Overview description of input reception block    | (none)                       | (none)                         | See notes in Block 1.2                                   |
| Sticky Note2                  | StickyNote                    | Comment on refined prompt engineering             | (none)                       | (none)                         | See notes in Block 1.4                                   |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node named** `"When clicking \"Test workflow\""` - no parameters.

2. **Create Postgres node `"List all tables in a database"`:**
   - Credentials: PostgreSQL credentials connected to your target DB.
   - Query:
     ```sql
     SELECT table_name FROM INFORMATION_SCHEMA.TABLES WHERE table_schema='public'
     ```
   - Connect manual trigger output to this node.

3. **Create Postgres node `"List all columns in a table"`:**
   - Query:
     ```sql
     SELECT
       column_name,
       udt_name as data_type,
       CASE WHEN udt_name = 'ARRAY' THEN TRUE ELSE FALSE END AS is_array,
       is_nullable
     FROM INFORMATION_SCHEMA.COLUMNS
     WHERE table_name = '{{ $json.table_name }}'
     ```
   - Connect output of previous node to this node.

4. **Create Set node `"Add table name to output"`:**
   - Add a string field `table` with value `={{ $('List all tables in a database').item.json.table_name ?? 'emails_metadata' }}`
   - Enable option to keep other fields.
   - Connect `"List all columns in a table"` to this node.

5. **Create ConvertToFile node `"Convert data to binary"`:**
   - Operation: toJson
   - Connect `"Add table name to output"` to this node.

6. **Create ReadWriteFile node `"Save file locally"`:**
   - Operation: write
   - Filename: `/files/pgsql-{{ $workflow.id }}.json`
   - Connect `"Convert data to binary"` to this node.

7. **Create If node `"If ran manually"`:**
   - Condition: OR condition to check if `"When clicking \"Test workflow\""` node was executed.
   - Connect `"Save file locally"` to this node (main output).
   - Also, separately connect `"When clicking \"Test workflow\""` node to this node (for manual trigger detection).

8. **Create ReadWriteFile node `"Load the schema from the local file"`:**
   - File to read: `/files/pgsql-{{ $workflow.id }}.json`
   - Set max tries to 2; continue workflow if error occurs.
   - Connect `"Chat Trigger"`, `"WorkflowTrigger"`, and `"If ran manually"` nodes to this node.

9. **Create If node `"If file exists or already retried generating it"`:**
   - Conditions: True if binary content exists OR `"If ran manually"` is true.
   - Connect `"Load the schema from the local file"` to this node.

10. **Create ExtractFromFile node `"Extract data from file"`:**
    - Operation: fromJson
    - Connect true output of `"If file exists or already retried generating it"` to this node.

11. **Wire false output of `"If file exists or already retried generating it"` back to trigger `"List all tables in a database"` to retry schema extraction.**

12. **Create Set node `"Combine schema data and chat input"`:**
    - Assign:
      - `sessionId`: from `$('Chat Trigger').first().json.sessionId` (conditional on execution)
      - `action`: from `$('Chat Trigger').first().json.action` (conditional)
      - `chatinput`: conditional expression choosing query from `"WorkflowTrigger"` input or from `"Chat Trigger"`'s `chatInput`
      - `schema`: from `$json.data` (coming from extracted file)
    - Connect from `"Extract data from file"`.

13. **Create Ollama Chat Model node `"Ollama Chat Model"`:**
    - Model: `phi4-mini:latest`
    - Configure Ollama credentials for local usage.
    - Connect `"Combine schema data and chat input"` to this node.

14. **Create LangChain Agent node `"AI Agent"`:**
    - Configure prompt (system message and user prompt):
      - System message defines strict SQL generation instructions adhering to schema, handling text, dates, arrays, boolean logic, metadata columns only.
      - User prompt includes today's date, schema, and user’s natural language input.
    - Connect `"Ollama Chat Model"` node’s `ai_languageModel` output to `"AI Agent"` node.

15. **Create Set node `"Extract SQL query"`:**
    - Extract raw SQL using regex like `SELECT[^;]*`
    - Output field: `query`
    - Connect `"AI Agent"` to this node.

16. **Create If node `"Check for trailing semicolon"`:**
    - Condition: query exists and does not end with `;`
    - Connect `"Extract SQL query"` output to `"Check for trailing semicolon"`.

17. **Create Set node `"Add trailing semicolon"`:**
    - Append semicolon to `query` string
    - Connect true output from `"Check for trailing semicolon"` here.

18. **Connect false output of `"Check for trailing semicolon"` and output of `"Add trailing semicolon"` both to the next If node.**

19. **Create If node `"Check if query exists"`:**
    - Condition: query string is not empty
    - Connect outputs from `"Check for trailing semicolon"`(false) and `"Add trailing semicolon"` (true) here.

20. **Create Postgres node `"Postgres"`:**
    - Query field: `{{ $json.query }}`
    - Use PostgreSQL credentials
    - Set onError to continue (fail-safe)
    - Connect true output of `"Check if query exists"` here.

21. **Create Set node `"Format query results"`:**
    - Create string `sqloutput` by formatting JSON rows into pipe-delimited strings with headers on first line
    - Connect `"Postgres"` output to here.

22. **Create Set node `"Format empty output"`:**
    - Set `query` field to empty string or pass-through as needed
    - Connect false output of `"Check if query exists"` to here.

23. **Create Merge node `"Combine query result and chat answer"`:**
    - Mode: combine by position, include unpaired
    - Connect `"Check if query exists"` false output (via `"Format empty output"`) and true output (via `"Format query results"`) here.

---

## 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow intended for sensitive email data; runs Ollama local model `phi4-mini` to protect privacy.           | https://ollama.com/library/phi4-mini                                                            |
| The prompt used in "AI Agent" node is carefully engineered to avoid errors and ensure schema adherence.       | See Sticky Note2 content referencing Kagi Assistant and Claude 3.7 Sonnet as inspirations        |
| To restrict schema extraction to a single table, modify the `"List all tables in a database"` node query: `WHERE table_schema='public' AND table_name='my_emails_table_name'` | Customization section in workflow description                                                    |
| This workflow can be used both as a chat interactive tool or integrated as a sub-workflow in larger pipelines. | Standard triggers and workflow trigger nodes enable flexibility                                 |
| Output includes both generated SQL query text `query` and execution results string `sqloutput` for convenience.| Enables easy consumption in UI or further automation                                             |

---

This completes a comprehensive, structured reference for the "Email Assistant: Convert Natural Language to SQL Queries with Phi4-mini and PostgreSQL" workflow.