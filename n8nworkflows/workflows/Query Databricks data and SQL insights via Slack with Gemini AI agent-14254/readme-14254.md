Query Databricks data and SQL insights via Slack with Gemini AI agent

https://n8nworkflows.xyz/workflows/query-databricks-data-and-sql-insights-via-slack-with-gemini-ai-agent-14254


# Query Databricks data and SQL insights via Slack with Gemini AI agent

# 1. Workflow Overview

This workflow lets users ask questions in Slack about data stored in Databricks, then uses a Gemini-powered AI agent to generate and execute a SQL query, interpret the results, and reply back in Slack.

Its main use case is conversational analytics: a user mentions the Slack app, asks a natural-language question about a Databricks table, and receives a human-readable answer without writing SQL manually.

The workflow is organized into four logical blocks:

## 1.1 Slack Input Reception and Databricks Configuration
The workflow starts when the Slack app is mentioned. It then prepares the Databricks table and warehouse identifiers needed for downstream schema retrieval and query execution.

## 1.2 Databricks Schema Retrieval and Parsing
The workflow requests the table schema from Databricks using a `DESCRIBE` statement, then transforms the response into a simplified schema description suitable for prompting the AI agent.

## 1.3 AI Agent, Model, Memory, and SQL Tooling
A LangChain-based AI agent receives the schema and the Slack message, uses Gemini as the chat model, stores conversation state in Redis, and can call a Databricks SQL execution tool exactly once.

## 1.4 Output Evaluation and Slack Response
The workflow evaluates the agent result and posts the generated response back to Slack, either as a channel message or as a threaded/error-style response depending on the branch taken.

---

# 2. Block-by-Block Analysis

## 2.1 Slack Input Reception and Databricks Configuration

### Overview
This block receives the user’s Slack message and initializes the Databricks connection parameters used by later nodes. It defines the target table and SQL warehouse in a static configuration node.

### Nodes Involved
- When Slack Message Received
- Set Databricks Config

### Node Details

#### When Slack Message Received
- **Type and technical role:** `n8n-nodes-base.slackTrigger`  
  Entry-point trigger node that listens for Slack events.
- **Configuration choices:**
  - Trigger type: `app_mention`
  - `watchWorkspace`: enabled
  - Uses Slack app credentials
- **Key expressions or variables used:**
  - Downstream nodes reference:
    - `$('When Slack Message Received').item.json.text`
    - `$('When Slack Message Received').item.json.channel`
    - `$('When Slack Message Received').item.json.thread_ts`
    - `$('When Slack Message Received').item.json.ts`
- **Input and output connections:**
  - No input; this is the entry point
  - Output → `Set Databricks Config`
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires a properly configured Slack app subscribed to mention events
- **Edge cases or potential failure types:**
  - Slack app not installed in workspace
  - Missing event subscriptions or incorrect OAuth scopes
  - Bot not mentioned, so workflow does not trigger
  - Missing `thread_ts` on non-thread messages; downstream logic handles this with fallback to `ts`
- **Sub-workflow reference:** None

#### Set Databricks Config
- **Type and technical role:** `n8n-nodes-base.set`  
  Assigns static configuration values required for Databricks API calls.
- **Configuration choices:**
  - Creates two string fields:
    - `target_table = enter_your_databricks_table_id`
    - `warehouse_id = enter_your_databricks_warehouse_id`
- **Key expressions or variables used:**
  - Exposes:
    - `$json.target_table`
    - `$json.warehouse_id`
- **Input and output connections:**
  - Input ← `When Slack Message Received`
  - Output → `Fetch Databricks Schema`
- **Version-specific requirements:**
  - `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - Placeholder values left unchanged
  - Incorrect Databricks table identifier format
  - Invalid warehouse ID causing Databricks API failure
- **Sub-workflow reference:** None

---

## 2.2 Databricks Schema Retrieval and Parsing

### Overview
This block fetches the schema of the configured Databricks table and converts the API response into a structured and AI-friendly representation. The schema becomes part of the prompt context for the AI agent.

### Nodes Involved
- Fetch Databricks Schema
- Parse Table Schema

### Node Details

#### Fetch Databricks Schema
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a direct REST request to Databricks SQL Statements API to retrieve table structure.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://enter-your-databricks-url.cloud.databricks.com/api/2.0/sql/statements`
  - Authentication: generic credential type using HTTP Bearer Auth
  - Body parameters:
    - `warehouse_id = {{ $json.warehouse_id }}`
    - `statement = DESCRIBE {{ $json.target_table }}`
    - `wait_timeout = 50s`
- **Key expressions or variables used:**
  - `{{ $json.warehouse_id }}`
  - `{{ $json.target_table }}`
- **Input and output connections:**
  - Input ← `Set Databricks Config`
  - Output → `Parse Table Schema`
- **Version-specific requirements:**
  - `typeVersion: 4.4`
  - Requires valid Databricks bearer token configured in `httpBearerAuth`
- **Edge cases or potential failure types:**
  - Invalid Databricks URL
  - Expired or unauthorized bearer token
  - Wrong warehouse ID
  - Table does not exist
  - `DESCRIBE` syntax may fail if table names need quoting/catalog.schema.table qualification
  - Query timeout if warehouse is unavailable or cold-starting
- **Sub-workflow reference:** None

#### Parse Table Schema
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses raw Databricks query results into structured JSON and a simplified text schema for prompting.
- **Configuration choices:**
  - JavaScript code reads `result.data_array`
  - Maps each row into:
    - `column`
    - `type`
    - `description`
  - Creates `ai_friendly_schema` as a bullet list like:
    - `- column_name (data_type)`
  - Returns:
    - `table_name`
    - `columns_count`
    - `schema_details`
    - `ai_friendly_schema`
- **Key expressions or variables used:**
  - `const rows = $json.result.data_array;`
  - Static return field:
    - `table_name: "franchises"`
- **Input and output connections:**
  - Input ← `Fetch Databricks Schema`
  - Output → `SQL Data Analyst Agent`
- **Version-specific requirements:**
  - `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If `result` or `data_array` is missing, the code throws an error
  - Databricks may return schema metadata rows not intended for business columns
  - `table_name` is hardcoded to `"franchises"` and may not match the configured table
  - Descriptions may be null or absent; handled with `"No description provided"`
- **Sub-workflow reference:** None

---

## 2.3 AI Agent, Model, Memory, and SQL Tooling

### Overview
This block is the core reasoning layer. It supplies the parsed schema and user request to an AI agent, backed by Gemini, with Redis-based memory and a Databricks SQL tool that the agent may call once to answer the question.

### Nodes Involved
- SQL Data Analyst Agent
- Gemini Model
- Redis Chat Memory
- Run Primary SQL Query

### Node Details

#### SQL Data Analyst Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain agent node that orchestrates model reasoning, optional tool usage, and final response generation.
- **Configuration choices:**
  - Prompt type: `define`
  - The prompt instructs the agent to:
    1. Generate one SQL query
    2. Execute it using the SQL tool
    3. Interpret the results conversationally
  - Includes:
    - schema from `{{ $json.ai_friendly_schema }}`
    - user question from Slack
    - table ID from config
  - Rules explicitly limit tool use to once
  - Also allows direct response when no database access is needed
- **Key expressions or variables used:**
  - `{{ $json.ai_friendly_schema }}`
  - `{{ $('When Slack Message Received').item.json.text }}`
  - `{{ $('Set Databricks Config').item.json.target_table }}`
- **Input and output connections:**
  - Main input ← `Parse Table Schema`
  - AI language model input ← `Gemini Model`
  - AI memory input ← `Redis Chat Memory`
  - AI tool input ← `Run Primary SQL Query`
  - Main output → `If Output Valid`
- **Version-specific requirements:**
  - `typeVersion: 3.1`
  - Requires compatible LangChain/AI nodes in the n8n version used
- **Edge cases or potential failure types:**
  - Model may ignore or imperfectly follow tool limits
  - Prompt may be insufficient for complex schemas
  - Slack mention text may include bot mention syntax that should ideally be cleaned
  - If schema parsing failed upstream, prompt context will be incomplete
  - Agent output format is not explicitly normalized, so downstream assumptions may be fragile
- **Sub-workflow reference:** None

#### Gemini Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Supplies the LLM used by the agent.
- **Configuration choices:**
  - Model: `models/gemini-3.1-flash-lite-preview`
  - No extra options configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output (AI language model) → `SQL Data Analyst Agent`
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires Google Gemini / PaLM API credentials
  - The configured model name must exist and be available to the account
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model deprecation or preview model instability
  - Rate limits or quota exhaustion
  - Safety filtering or malformed prompt/tool interaction
- **Sub-workflow reference:** None

#### Redis Chat Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryRedisChat`  
  Stores conversational memory per Slack message thread or message timestamp.
- **Configuration choices:**
  - `sessionIdType = customKey`
  - `sessionKey = {{ thread_ts || ts }}`
  - TTL = `60`
- **Key expressions or variables used:**
  - `{{ $('When Slack Message Received').item.json.thread_ts || $('When Slack Message Received').item.json.ts }}`
- **Input and output connections:**
  - Output (AI memory) → `SQL Data Analyst Agent`
- **Version-specific requirements:**
  - `typeVersion: 1.5`
  - Requires working Redis credentials and reachable Redis instance
- **Edge cases or potential failure types:**
  - TTL of 60 may be very short depending on unit interpretation and desired chat continuity
  - Redis unavailable or auth failure
  - Thread/message timestamp collisions are unlikely but possible in edge integration cases
- **Sub-workflow reference:** None

#### Run Primary SQL Query
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`  
  Tool node exposed to the AI agent for executing a single Databricks SQL query.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://enter-your-databricks-url-here.cloud.databricks.com/api/2.0/sql/statements`
  - Authentication: HTTP Bearer Auth
  - Body parameters:
    - `warehouse_id = {{ $('Set Databricks Config').item.json.warehouse_id }}`
    - `statement = {{ $fromAI(...) }}`
    - `wait_timeout = 50s`
  - Tool description strongly instructs:
    - use only once
    - provide full SQL in `statement`
    - return final answer after execution
- **Key expressions or variables used:**
  - `{{ $('Set Databricks Config').item.json.warehouse_id }}`
  - `{{ $fromAI('parameters1_Value', '', 'string') }}`
- **Input and output connections:**
  - Output (AI tool) → `SQL Data Analyst Agent`
- **Version-specific requirements:**
  - `typeVersion: 4.4`
  - Requires n8n version supporting AI tool nodes and `$fromAI(...)`
- **Edge cases or potential failure types:**
  - Placeholder URL left unchanged
  - Agent may generate invalid SQL
  - SQL injection-like unsafe prompt behavior if the user manipulates instructions
  - Warehouse unavailability or timeout
  - Result size too large for practical model interpretation
  - Agent may still attempt repeated calls despite prompt/tool instructions
- **Sub-workflow reference:** None

---

## 2.4 Output Evaluation and Slack Response

### Overview
This block routes the AI agent output toward one of two Slack posting nodes. One path posts directly to the channel; the other posts with thread metadata for error-style or alternate handling.

### Nodes Involved
- If Output Valid
- Post to Slack Channel
- Post Error to Slack

### Node Details

#### If Output Valid
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional routing node intended to decide how the Slack response should be posted.
- **Configuration choices:**
  - Uses a single string condition:
    - Left value is dynamically built as either:
      - `thread[thread_ts]`
      - or `message_ts[ts]`
    - Checks whether that string `contains "message"`
- **Key expressions or variables used:**
  - `{{ $('When Slack Message Received').item.json.thread_ts ? 'thread[' + $('When Slack Message Received').item.json.thread_ts + ']' : 'message_ts[' + $('When Slack Message Received').item.json.ts + ']' }}`
- **Input and output connections:**
  - Input ← `SQL Data Analyst Agent`
  - True output → `Post to Slack Channel`
  - False output → `Post Error to Slack`
- **Version-specific requirements:**
  - `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - The condition does not validate agent output; it only tests whether a constructed string contains `"message"`
  - Because fallback string starts with `message_ts[...]`, non-thread messages likely go true; thread messages likely go false
  - Node name suggests output validation, but logic actually behaves like thread detection
  - This can confuse maintainers and automation agents
- **Sub-workflow reference:** None

#### Post to Slack Channel
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the agent’s final text response to the Slack channel.
- **Configuration choices:**
  - Text: `{{ $json.output }}`
  - Destination: channel selected by ID from trigger payload
  - `includeLinkToWorkflow`: false
- **Key expressions or variables used:**
  - `{{ $json.output }}`
  - `{{ $('When Slack Message Received').item.json.channel }}`
- **Input and output connections:**
  - Input ← `If Output Valid` (true branch)
- **Version-specific requirements:**
  - `typeVersion: 2.4`
- **Edge cases or potential failure types:**
  - If agent output is not stored in `output`, message will be blank or fail
  - Slack permission issues for posting in target channel
  - Channel ID may be inaccessible to the bot
- **Sub-workflow reference:** None

#### Post Error to Slack
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the agent’s response to Slack, explicitly including thread reply metadata.
- **Configuration choices:**
  - Text: `{{ $json.output }}`
  - Destination: channel from trigger payload
  - Thread timestamp:
    - `{{ $('When Slack Message Received').item.json.thread_ts }}`
  - `includeLinkToWorkflow`: false
- **Key expressions or variables used:**
  - `{{ $json.output }}`
  - `{{ $('When Slack Message Received').item.json.channel }}`
  - `{{ $('When Slack Message Received').item.json.thread_ts }}`
- **Input and output connections:**
  - Input ← `If Output Valid` (false branch)
- **Version-specific requirements:**
  - `typeVersion: 2.4`
- **Edge cases or potential failure types:**
  - If original message is not in a thread and `thread_ts` is empty, Slack reply behavior may be inconsistent
  - Node name suggests error handling, but it actually functions more like threaded posting
  - Same Slack auth and channel-access issues as the other Slack node
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Slack Message Received | n8n-nodes-base.slackTrigger | Receives Slack app mentions and starts the workflow |  | Set Databricks Config | ## Initialize trigger and connection<br>Triggering the process and preparing Databricks connection details. |
| Set Databricks Config | n8n-nodes-base.set | Stores target Databricks table and warehouse identifiers | When Slack Message Received | Fetch Databricks Schema | ## Initialize trigger and connection<br>Triggering the process and preparing Databricks connection details. |
| Fetch Databricks Schema | n8n-nodes-base.httpRequest | Calls Databricks SQL API to describe the target table schema | Set Databricks Config | Parse Table Schema | ## Process schema and AI agent<br>Parsing table schema and processing AI logic using Gemini and tools to execute generated sql query. |
| Parse Table Schema | n8n-nodes-base.code | Converts Databricks schema output into structured and AI-friendly text | Fetch Databricks Schema | SQL Data Analyst Agent | ## Process schema and AI agent<br>Parsing table schema and processing AI logic using Gemini and tools to execute generated sql query. |
| SQL Data Analyst Agent | @n8n/n8n-nodes-langchain.agent | Interprets the user request, optionally runs SQL once, and generates final answer | Parse Table Schema; Gemini Model; Redis Chat Memory; Run Primary SQL Query | If Output Valid | ## Process schema and AI agent<br>Parsing table schema and processing AI logic using Gemini and tools to execute generated sql query. |
| Gemini Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides the LLM used by the AI agent |  | SQL Data Analyst Agent | ## Process schema and AI agent<br>Parsing table schema and processing AI logic using Gemini and tools to execute generated sql query. |
| Redis Chat Memory | @n8n/n8n-nodes-langchain.memoryRedisChat | Maintains per-thread or per-message chat memory in Redis |  | SQL Data Analyst Agent | ## Process schema and AI agent<br>Parsing table schema and processing AI logic using Gemini and tools to execute generated sql query. |
| Run Primary SQL Query | n8n-nodes-base.httpRequestTool | Tool exposed to the agent for one-time SQL execution against Databricks |  | SQL Data Analyst Agent | ## Process schema and AI agent<br>Parsing table schema and processing AI logic using Gemini and tools to execute generated sql query. |
| If Output Valid | n8n-nodes-base.if | Routes final Slack posting path based on thread/message detection logic | SQL Data Analyst Agent | Post to Slack Channel; Post Error to Slack | ## Evaluate conditions and report<br>Handling conditional responses and final Slack notifications. |
| Post to Slack Channel | n8n-nodes-base.slack | Posts response directly to Slack channel | If Output Valid |  | ## Evaluate conditions and report<br>Handling conditional responses and final Slack notifications. |
| Post Error to Slack | n8n-nodes-base.slack | Posts response to Slack using thread metadata | If Output Valid |  | ## Evaluate conditions and report<br>Handling conditional responses and final Slack notifications. |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation for the overall workflow |  |  | ## Chat with databricks<br><br>### How it works<br><br>1. A Slack trigger initiates the workflow upon receiving a message. 2. The workflow fetches the Databricks table schema and parses it for the AI model. 3. An AI agent processes the data using a chat model and SQL tools to generate a response. 4. The workflow evaluates the AI output and sends a message back via Slack.<br><br>### Setup steps<br><br>- Configure the Slack Trigger node with your Slack app credentials.<br>- Update the Databricks Configuration node with your target table and warehouse IDs.<br>- Ensure the AI Agent has valid Google Gemini and Redis credentials.<br>- Connect the Slack nodes to your workspace's notification channel.<br><br>### Customization<br><br>Adjust the system prompt in the AI agent to tailor the SQL generation style for your specific database schema. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for trigger/config block |  |  | ## Initialize trigger and connection<br>Triggering the process and preparing Databricks connection details. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for schema/AI block |  |  | ## Process schema and AI agent<br>Parsing table schema and processing AI logic using Gemini and tools to execute generated sql query. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for response block |  |  | ## Evaluate conditions and report<br>Handling conditional responses and final Slack notifications. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Query Databricks data and SQL insights via Slack with Gemini AI agent`.

2. **Add a Slack Trigger node**
   - Node type: `Slack Trigger`
   - Name: `When Slack Message Received`
   - Configure Slack credentials with your Slack app.
   - Set trigger event to `app_mention`.
   - Enable workspace watching if available.
   - Ensure your Slack app has the necessary scopes and event subscriptions.

3. **Add a Set node**
   - Node type: `Set`
   - Name: `Set Databricks Config`
   - Add two string fields:
     - `target_table`
     - `warehouse_id`
   - Set them to your Databricks target table identifier and SQL warehouse ID.
   - Connect:
     - `When Slack Message Received` → `Set Databricks Config`

4. **Add an HTTP Request node for schema retrieval**
   - Node type: `HTTP Request`
   - Name: `Fetch Databricks Schema`
   - Method: `POST`
   - URL:
     - `https://<your-databricks-host>.cloud.databricks.com/api/2.0/sql/statements`
   - Authentication:
     - Use `Generic Credential Type`
     - Select `HTTP Bearer Auth`
     - Create credentials containing a Databricks personal access token or equivalent bearer token.
   - Enable body sending.
   - Add body parameters:
     - `warehouse_id` = `={{ $json.warehouse_id }}`
     - `statement` = `=DESCRIBE {{ $json.target_table }}`
     - `wait_timeout` = `50s`
   - Connect:
     - `Set Databricks Config` → `Fetch Databricks Schema`

5. **Add a Code node to parse schema**
   - Node type: `Code`
   - Name: `Parse Table Schema`
   - Language: JavaScript
   - Use logic equivalent to:
     - read `result.data_array`
     - map rows to structured schema objects
     - create a simple bullet-list schema string
     - output `schema_details`, `columns_count`, and `ai_friendly_schema`
   - Important: the provided workflow hardcodes `table_name: "franchises"`. Replace this with a dynamic value if needed.
   - Connect:
     - `Fetch Databricks Schema` → `Parse Table Schema`

6. **Add the AI Agent node**
   - Node type: `AI Agent` / `LangChain Agent`
   - Name: `SQL Data Analyst Agent`
   - Prompt mode: define prompt manually
   - Use a prompt equivalent to:
     - identify the assistant as an expert Databricks SQL analyst
     - instruct it to generate one SQL query
     - execute it once with the SQL tool
     - interpret results conversationally
     - answer directly if no database lookup is needed
   - Include these expressions in the prompt:
     - schema: `{{ $json.ai_friendly_schema }}`
     - user question: `{{ $('When Slack Message Received').item.json.text }}`
     - table ID: `{{ $('Set Databricks Config').item.json.target_table }}`
   - Connect:
     - `Parse Table Schema` → `SQL Data Analyst Agent`

7. **Add a Google Gemini chat model node**
   - Node type: `Google Gemini Chat Model`
   - Name: `Gemini Model`
   - Configure Google Gemini / PaLM API credentials.
   - Select model:
     - `models/gemini-3.1-flash-lite-preview`
   - Connect the model to the AI Agent using the AI language model connection:
     - `Gemini Model` → `SQL Data Analyst Agent`

8. **Add a Redis Chat Memory node**
   - Node type: `Redis Chat Memory`
   - Name: `Redis Chat Memory`
   - Configure Redis credentials.
   - Session ID type: custom key
   - Session key expression:
     - `={{ $('When Slack Message Received').item.json.thread_ts || $('When Slack Message Received').item.json.ts }}`
   - TTL: `60`
   - Connect the memory node to the AI Agent using the AI memory connection:
     - `Redis Chat Memory` → `SQL Data Analyst Agent`

9. **Add an HTTP Request Tool node for SQL execution**
   - Node type: `HTTP Request Tool`
   - Name: `Run Primary SQL Query`
   - URL:
     - `https://<your-databricks-host>.cloud.databricks.com/api/2.0/sql/statements`
   - Method: `POST`
   - Authentication:
     - Generic credential type with `HTTP Bearer Auth`
     - Reuse the Databricks bearer credential if appropriate
   - Send body parameters:
     - `warehouse_id` = `={{ $('Set Databricks Config').item.json.warehouse_id }}`
     - `statement` = AI-provided parameter using `$fromAI(...)`
     - `wait_timeout` = `50s`
   - Add a tool description telling the agent:
     - this tool executes a single SQL query
     - it may be used only once
     - it must receive the full SQL statement
     - after execution the agent must provide a final answer
   - Connect the tool to the AI Agent via AI tool connection:
     - `Run Primary SQL Query` → `SQL Data Analyst Agent`

10. **Add an If node**
    - Node type: `If`
    - Name: `If Output Valid`
    - Recreate the expression exactly if you want parity with the original:
      - left value:
        `={{ $('When Slack Message Received').item.json.thread_ts ? 'thread[' + $('When Slack Message Received').item.json.thread_ts + ']' : 'message_ts[' + $('When Slack Message Received').item.json.ts + ']' }}`
      - condition: string `contains`
      - right value: `message`
    - Connect:
      - `SQL Data Analyst Agent` → `If Output Valid`
    - Note: despite its name, this node acts more like thread/non-thread routing than output validation.

11. **Add a Slack node for standard posting**
    - Node type: `Slack`
    - Name: `Post to Slack Channel`
    - Operation: send message to channel
    - Text:
      - `={{ $json.output }}`
    - Channel ID:
      - `={{ $('When Slack Message Received').item.json.channel }}`
    - Disable workflow link inclusion if desired.
    - Connect:
      - `If Output Valid` true branch → `Post to Slack Channel`

12. **Add a second Slack node for threaded posting**
    - Node type: `Slack`
    - Name: `Post Error to Slack`
    - Operation: send message to channel
    - Text:
      - `={{ $json.output }}`
    - Channel ID:
      - `={{ $('When Slack Message Received').item.json.channel }}`
    - Thread timestamp:
      - `={{ $('When Slack Message Received').item.json.thread_ts }}`
    - Disable workflow link inclusion if desired.
    - Connect:
      - `If Output Valid` false branch → `Post Error to Slack`

13. **Add optional sticky notes**
    - Create workspace notes for:
      - overall workflow purpose and setup
      - trigger/config block
      - schema/AI block
      - output/reporting block

14. **Configure credentials**
    - **Slack**
      - Use a Slack app credential for both trigger and posting nodes
      - Ensure event subscriptions and message-posting scopes are granted
    - **Databricks**
      - Use HTTP Bearer Auth with a valid Databricks token
      - Use the correct workspace host in both HTTP nodes
    - **Google Gemini**
      - Configure a Gemini/PaLM API credential with sufficient quota
    - **Redis**
      - Configure host, port, auth, and TLS as required by your Redis deployment

15. **Test the workflow**
    - Mention the Slack app in a channel
    - Confirm:
      - Slack trigger fires
      - schema is fetched from Databricks
      - code node builds `ai_friendly_schema`
      - AI agent either answers directly or uses the SQL tool once
      - final response appears in Slack

16. **Recommended hardening after reproduction**
    - Replace placeholder Databricks URLs and IDs
    - Clean Slack mention syntax before sending text to the model
    - Make `table_name` dynamic in the Code node
    - Improve the `If Output Valid` condition to actually validate agent output
    - Add explicit error handling for Databricks API failures and empty query results
    - Consider adding guardrails for generated SQL, row limits, and approved tables only

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Chat with databricks: A Slack trigger initiates the workflow upon receiving a message. The workflow fetches the Databricks table schema and parses it for the AI model. An AI agent processes the data using a chat model and SQL tools to generate a response. The workflow evaluates the AI output and sends a message back via Slack. | Overall workflow note |
| Configure the Slack Trigger node with your Slack app credentials. | Setup guidance |
| Update the Databricks Configuration node with your target table and warehouse IDs. | Setup guidance |
| Ensure the AI Agent has valid Google Gemini and Redis credentials. | Setup guidance |
| Connect the Slack nodes to your workspace's notification channel. | Setup guidance |
| Adjust the system prompt in the AI agent to tailor the SQL generation style for your specific database schema. | Customization guidance |
| Initialize trigger and connection: Triggering the process and preparing Databricks connection details. | Trigger/config block |
| Process schema and AI agent: Parsing table schema and processing AI logic using Gemini and tools to execute generated sql query. | Schema/AI block |
| Evaluate conditions and report: Handling conditional responses and final Slack notifications. | Response block |