Query PostgreSQL Database with Natural Language using GPT-4o-mini

https://n8nworkflows.xyz/workflows/query-postgresql-database-with-natural-language-using-gpt-4o-mini-7988


# Query PostgreSQL Database with Natural Language using GPT-4o-mini

### 1. Workflow Overview

This n8n workflow enables querying a PostgreSQL database using natural language inputs processed by the GPT-4o-mini model. Its primary use case is to allow users to send chat messages with natural language queries about a specific database table and receive the corresponding data results without writing SQL manually.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures chat messages from an external interface.
- **1.2 Table Configuration:** Sets the target database table for queries.
- **1.3 AI Agent Processing:** Uses GPT-4o-mini via LangChain to interpret the natural language query, map it to SQL, and execute it.
- **1.4 Database Interaction:** Executes SQL queries on PostgreSQL to fetch table schema and query results.
- **1.5 Memory Management:** Maintains conversational context using a simple sliding window memory.
- **1.6 User Guidance:** Provides sticky notes with documentation and instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming chat messages to trigger the workflow.

**Nodes Involved:**  
- When chat message received

**Node Details:**  
- **Name:** When chat message received  
- **Type:** `@n8n/n8n-nodes-langchain.chatTrigger` (Chat Message Webhook Trigger)  
- **Configuration:** Public webhook enabled, no extra options.  
- **Expressions/Variables:** Captures `sessionId` and user chat input as `chatInput` from incoming JSON payload.  
- **IO:** Input: External chat messages (via webhook) → Output: passes data to "Set Table Name" node.  
- **Version:** 1.1  
- **Edge Cases:** Webhook downtime, malformed JSON input, session ID missing or inconsistent.  
- **Sub-workflow:** None.

---

#### 2.2 Table Configuration

**Overview:**  
Sets the database table name dynamically for queries based on the workflow configuration.

**Nodes Involved:**  
- Set Table Name

**Node Details:**  
- **Name:** Set Table Name  
- **Type:** `n8n-nodes-base.set` (Set Node)  
- **Configuration:** Assigns a static string `product_inventory` to the variable `table_name`. This can be edited to target any table.  
- **Expressions/Variables:** None dynamic; fixed assignment.  
- **IO:** Input: Output from "When chat message received" → Output: passes `table_name` to "Database Agent" node.  
- **Version:** 3.4  
- **Edge Cases:** User must manually edit this node to specify the correct table; misconfiguration leads to invalid queries downstream.  
- **Sub-workflow:** None.

---

#### 2.3 AI Agent Processing

**Overview:**  
Processes the natural language input, understands the database schema, constructs a safe SQL query, executes it, and returns results.

**Nodes Involved:**  
- Database Agent  
- OpenAI Chat Model  
- Simple Memory

**Node Details:**  

- **Database Agent**  
  - **Type:** `@n8n/n8n-nodes-langchain.agent` (AI Agent Node)  
  - **Role:** Core logic interprets natural language query to SQL using schema info and executes query.  
  - **Configuration:**  
    - Takes user chat input from previous nodes.  
    - Uses a detailed system message instructing how to map natural language to SQL for a single table.  
    - Specifies workflow steps internally (schema retrieval, mapping terms to columns, discovery queries, SQL construction, execution, and results filtering).  
    - Output rules ensure no SQL or internal details leak to the user, only clean data results.  
  - **Expressions:** Uses `chatInput` from chat trigger, and `table_name` from Set Table Name node.  
  - **IO:**  
    - Input: user chat text and table_name.  
    - Calls tools: OpenAI Chat Model, Execute SQL Query, Get Table Definition.  
    - Memory linked to "Simple Memory" node.  
  - **Version:** 2  
  - **Edge Cases:** Ambiguous queries, unexpected table schemas, database connection errors, AI model timeouts, malformed SQL generation.  
  - **Sub-workflow:** None.

- **OpenAI Chat Model**  
  - **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI GPT Model Node)  
  - **Role:** Provides GPT-4o-mini language model capabilities for interpreting and generating SQL queries.  
  - **Configuration:** Uses `gpt-4o-mini` model preset; no additional options set.  
  - **IO:** Input from "Database Agent" → Output back to "Database Agent".  
  - **Version:** 1.2  
  - **Edge Cases:** API key invalid, rate limiting, request timeouts.  
  - **Credentials:** Requires configured OpenAI API key.

- **Simple Memory**  
  - **Type:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Memory Node)  
  - **Role:** Maintains conversational history for context preservation using a sliding window buffer keyed by session ID.  
  - **Configuration:** Uses session ID from chat trigger to keep context unique per user/session.  
  - **IO:** Input and output connected with "Database Agent" as AI memory.  
  - **Version:** 1.3  
  - **Edge Cases:** Memory overflow, session ID missing, inconsistent context.

---

#### 2.4 Database Interaction

**Overview:**  
Fetches table schema details and executes generated SQL queries safely against PostgreSQL.

**Nodes Involved:**  
- Execute SQL Query  
- Get Table Definition

**Node Details:**  

- **Execute SQL Query**  
  - **Type:** `n8n-nodes-base.postgresTool` (PostgreSQL Query Executor)  
  - **Role:** Executes dynamic SQL queries generated by AI agent to retrieve data rows.  
  - **Configuration:**  
    - Query is passed as a dynamic expression resolved from AI output key `sql_query`.  
    - Operation: `executeQuery`.  
    - Tool description emphasizes schema-qualified table names for correctness.  
  - **IO:** Input: SQL query string from AI agent → Output: query result rows for response.  
  - **Version:** 2.5  
  - **Credentials:** Requires PostgreSQL connection credentials with SELECT permissions.  
  - **Edge Cases:** SQL syntax errors, connection issues, permission errors, empty result sets.

- **Get Table Definition**  
  - **Type:** `n8n-nodes-base.postgresTool`  
  - **Role:** Retrieves detailed schema metadata (columns, data types, constraints, foreign keys) for the specified table to assist AI agent mapping.  
  - **Configuration:**  
    - SQL query joins `information_schema` views filtering by `table_name` (dynamic from `table_name` variable) and `public` schema.  
    - Executes only once per session to optimize performance.  
  - **IO:** Input: `table_name` → Output: schema details to AI agent.  
  - **Version:** 2.5  
  - **Credentials:** Same PostgreSQL credentials as above.  
  - **Edge Cases:** Table not found, schema access denied.

---

#### 2.5 User Guidance

**Overview:**  
Provides visual documentation and instructions directly within the workflow editor.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**  

- **Sticky Note**  
  - **Type:** `n8n-nodes-base.stickyNote`  
  - **Role:** Displays heading "Database Query Agent" as a visual label for the main functional area.  
  - **Position:** Near the main agent nodes.  
  - **Edge Cases:** None.

- **Sticky Note1**  
  - **Type:** `n8n-nodes-base.stickyNote`  
  - **Role:** Detailed step-by-step prerequisites and usage instructions, including credential setup, table configuration, and example queries.  
  - **Content Highlights:**  
    - n8n instance requirements  
    - PostgreSQL and OpenAI API setup  
    - Editing table name in Set Table Name node  
    - Testing permissions  
    - Usage examples of natural language queries  
  - **Edge Cases:** None; purely informational.

---

### 3. Summary Table

| Node Name             | Node Type                                           | Functional Role                       | Input Node(s)              | Output Node(s)               | Sticky Note                                                       |
|-----------------------|----------------------------------------------------|------------------------------------|----------------------------|-----------------------------|------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger            | Input Reception (chat webhook)      | External                   | Set Table Name              |                                                                  |
| Set Table Name        | n8n-nodes-base.set                                 | Table configuration                 | When chat message received | Database Agent              |                                                                  |
| Database Agent        | @n8n/n8n-nodes-langchain.agent                     | AI natural language to SQL agent   | Set Table Name, Simple Memory, OpenAI Chat Model, Get Table Definition, Execute SQL Query | Simple Memory             | Covered by "Database Query Agent" sticky note                   |
| OpenAI Chat Model     | @n8n/n8n-nodes-langchain.lmChatOpenAi              | Provides GPT-4o-mini model          | Database Agent             | Database Agent              | Covered by "Database Query Agent" sticky note                   |
| Simple Memory         | @n8n/n8n-nodes-langchain.memoryBufferWindow        | Maintains conversation context     | Database Agent             | Database Agent              | Covered by "Database Query Agent" sticky note                   |
| Execute SQL Query     | n8n-nodes-base.postgresTool                         | Executes generated SQL queries     | Database Agent             | Database Agent              | Covered by "Database Query Agent" sticky note                   |
| Get Table Definition  | n8n-nodes-base.postgresTool                         | Retrieves table schema metadata    | Database Agent             | Database Agent              | Covered by "Database Query Agent" sticky note                   |
| Sticky Note           | n8n-nodes-base.stickyNote                           | Visual label "Database Query Agent" | None                      | None                       | "## Database Query Agent"                                        |
| Sticky Note1          | n8n-nodes-base.stickyNote                           | User instructions and prerequisites | None                      | None                       | Detailed setup and usage steps including credential setup and examples |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Trigger Node:**  
   - Add `@n8n/n8n-nodes-langchain.chatTrigger` node named `When chat message received`.  
   - Configure with a public webhook (default settings). No special options needed.

2. **Add Set Node for Table Name:**  
   - Add `Set` node named `Set Table Name`.  
   - Add assignment: variable `table_name` = `"product_inventory"` (string).  
   - Connect `When chat message received` → `Set Table Name`.

3. **Add AI Agent Node:**  
   - Add `@n8n/n8n-nodes-langchain.agent` node named `Database Agent`.  
   - Set parameter `text` to expression: `{{ $('When chat message received').item.json.chatInput }}`.  
   - Paste or write the system message provided in the workflow to instruct the AI on SQL mapping logic and output rules.  
   - Connect `Set Table Name` → `Database Agent`.

4. **Add OpenAI Chat Model Node:**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node named `OpenAI Chat Model`.  
   - Select model `gpt-4o-mini`.  
   - Connect `Database Agent` (as ai_languageModel input) → `OpenAI Chat Model`.

5. **Add Simple Memory Node:**  
   - Add `@n8n/n8n-nodes-langchain.memoryBufferWindow` node named `Simple Memory`.  
   - Set `sessionKey` to expression: `={{ $('When chat message received').item.json.sessionId }}`.  
   - Connect `Database Agent` (ai_memory input) → `Simple Memory`.

6. **Add PostgreSQL Execute Query Node:**  
   - Add `Postgres Tool` node named `Execute SQL Query`.  
   - Set operation to `executeQuery`.  
   - Set query to expression: `{{ $fromAI("sql_query", "SQL Query") }}` (this extracts the SQL query generated by AI).  
   - Connect `Database Agent` (ai_tool input) → `Execute SQL Query`.

7. **Add PostgreSQL Get Table Definition Node:**  
   - Add another `Postgres Tool` node named `Get Table Definition`.  
   - Set operation to `executeQuery`.  
   - Paste the provided SQL to fetch schema metadata from `information_schema` filtering by table_name and schema 'public'.  
   - Use expression for table name in the query: `'{{ $json.table_name }}'`.  
   - Enable `executeOnce` to avoid repeated calls.  
   - Connect `Database Agent` (ai_tool input) → `Get Table Definition`.

8. **Set Credentials:**  
   - OpenAI Chat Model node: configure with valid OpenAI API key credentials.  
   - Both Postgres nodes: configure with PostgreSQL credentials that have read (SELECT) permissions on the target database.

9. **Add Sticky Notes (Optional):**  
   - Add `Sticky Note` near the agent nodes titled "Database Query Agent".  
   - Add another `Sticky Note` with detailed setup instructions: prerequisites, credentials, usage examples.

10. **Connect Outputs for Proper Execution:**  
    - Ensure all ai_* inputs and outputs are connected properly on the `Database Agent` node to OpenAI, Simple Memory, Execute SQL Query, and Get Table Definition nodes.  
    - Connect main workflow from `When chat message received` → `Set Table Name` → `Database Agent`.

11. **Test Workflow:**  
    - Deploy and start the workflow.  
    - Send a chat message with natural language query via webhook.  
    - The workflow will process the query, generate SQL, execute it, and return data results.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow requires a PostgreSQL database with proper SELECT permissions and OpenAI API access.                                               | n8n credentials configuration for PostgreSQL and OpenAI API                                        |
| The AI agent uses a system prompt designed for safe SQL generation and schema-aware queries on a single table.                                  | Embedded in Database Agent node parameters                                                         |
| Example queries include "Show all active users", "Find orders from last month over $100", "List products with low inventory".                   | Sticky Note1 content                                                                                |
| The workflow uses GPT-4o-mini, a lightweight GPT-4 variant suitable for cost-effective natural language processing.                            | OpenAI model selection in OpenAI Chat Model node                                                   |
| For advanced use, modify the "Set Table Name" node to target other tables or extend to multiple tables by adapting the system prompt logic.    | Editable Set Table Name node                                                                        |
| Workflow memory management provides session isolation via sessionId to maintain relevant conversational context per user.                     | Simple Memory node configuration                                                                    |
| The workflow respects best practices by not exposing internal SQL or AI reasoning to end-users, returning only clean data result sets.         | System message instructions in Database Agent node                                                 |

---

*Disclaimer: The provided text is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.*