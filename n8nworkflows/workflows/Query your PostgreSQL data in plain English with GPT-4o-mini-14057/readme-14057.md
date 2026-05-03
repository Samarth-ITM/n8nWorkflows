Query your PostgreSQL data in plain English with GPT-4o-mini

https://n8nworkflows.xyz/workflows/query-your-postgresql-data-in-plain-english-with-gpt-4o-mini-14057


# Query your PostgreSQL data in plain English with GPT-4o-mini

# 1. Workflow Overview

This workflow enables users to query a PostgreSQL table in plain English through a chat interface powered by GPT-4o-mini. It supports session-based table selection, automatic schema discovery, SQL generation constrained to real columns, execution of the query, and chart generation for visual responses.

It also includes a separate, optional data-loading branch that imports data from Google Sheets into PostgreSQL.

## 1.1 Chat Entry and Session State Management

The workflow starts from a chat trigger. It captures the user message, stores or clears the selected table name per chat session, and determines whether the current message is:
- a new table name,
- a table reset command,
- or an analytics request against a previously selected table.

## 1.2 Schema Discovery and Context Preparation

If a table has already been selected and the message is an analytics request, the workflow fetches the table schema from PostgreSQL’s `information_schema.columns`. It then formats that schema into a text representation for the AI agent and creates shorthand-to-column guidance.

If no schema should be loaded yet, a fallback path provides placeholder values instead.

## 1.3 AI Analytics Agent

A LangChain agent receives:
- the user message,
- the selected table,
- the formatted schema,
- and strict operating rules.

Depending on the state, it either:
- asks the user to provide a table name,
- confirms a newly selected table,
- confirms reset,
- reports that the table does not exist,
- or generates and executes SQL through a PostgreSQL tool.

The agent is also instructed to generate a chart for every analytical reply.

## 1.4 AI Tools and Memory

The agent uses:
- GPT-4o-mini as the language model,
- PostgreSQL chat memory stored in Postgres,
- a PostgreSQL tool to execute AI-generated SQL,
- and a QuickChart tool to create chart output.

## 1.5 Optional Google Sheets to PostgreSQL Sync

A separate manual branch truncates a PostgreSQL table, fetches rows from a Google Sheet, and inserts them into PostgreSQL. This is intended for batch refresh of the demo dataset or any similarly structured table.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Chat Input Reception and Table Session Management

### Overview

This block receives chat messages and manages persistent table state per session. It lets a user set a table once, reuse it for later questions, or clear it with `reset table`.

### Nodes Involved

- Chat Trigger
- Manage Table Name

### Node Details

#### Chat Trigger

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Public chat entry point for conversational interaction.
- **Configuration choices:**
  - Public chat is enabled.
  - No additional options are configured.
- **Key expressions or variables used:**
  - Provides `sessionId`
  - Provides `chatInput`
- **Input and output connections:**
  - Entry point node
  - Outputs to `Manage Table Name`
- **Version-specific requirements:**
  - Type version `1.4`
  - Requires n8n with LangChain chat trigger support.
- **Edge cases or potential failure types:**
  - Public exposure may allow unsolicited use if not protected externally.
  - If `sessionId` or `chatInput` were missing, downstream code falls back safely.
- **Sub-workflow reference:** None

#### Manage Table Name

- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript node that stores or clears the selected table name using workflow static data.
- **Configuration choices:**
  - Uses global workflow static data.
  - Segregates state by sanitized `sessionId`.
  - Recognizes `reset table`, `change table`, and `switch table`.
  - Detects a table name only if:
    - no table is currently stored for the session,
    - the message looks like a valid SQL identifier,
    - and it is not a blocked common word.
  - Strips schema prefixes such as `public.orders` to `orders`.
- **Key expressions or variables used:**
  - `$getWorkflowStaticData('global')`
  - `$input.first().json.sessionId`
  - `$input.first().json.chatInput`
  - Emits:
    - `chatInput`
    - `tableName`
    - `justSetTable`
    - `isReset`
    - `sessionId`
- **Input and output connections:**
  - Input from `Chat Trigger`
  - Output to `Need Schema?`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Static data is workflow-global; the code mitigates cross-user contamination by namespacing sessions.
  - Table names in non-`public` schemas are normalized by stripping the prefix, which may be incorrect if multiple schemas contain tables with the same name.
  - Reserved-word blocking may reject some legitimate table names if they match common terms.
  - Session storage persists until changed or reset; there is no automatic expiration.
- **Sub-workflow reference:** None

---

## 2.2 Block: Schema Decision and Retrieval

### Overview

This block decides whether schema discovery is needed. If the user has already selected a table and is asking an actual question, the workflow loads the column metadata from PostgreSQL.

### Nodes Involved

- Need Schema?
- Fetch Schema
- Empty Schema

### Node Details

#### Need Schema?

- **Type and technical role:** `n8n-nodes-base.if`  
  Branching node that determines whether to fetch schema or send placeholder schema content.
- **Configuration choices:**
  - True branch requires all of the following:
    - `tableName` is not empty
    - `justSetTable` is not `true`
    - `isReset` is not `true`
- **Key expressions or variables used:**
  - `{{ $json.tableName }}`
  - `{{ $json.justSetTable }}`
  - `{{ $json.isReset }}`
- **Input and output connections:**
  - Input from `Manage Table Name`
  - True output to `Fetch Schema`
  - False output to `Empty Schema`
- **Version-specific requirements:**
  - Type version `2.2`
- **Edge cases or potential failure types:**
  - Strict type validation means malformed booleans could affect routing.
  - If a table is newly set, schema is deliberately not fetched in this turn; the agent will confirm the table instead.
- **Sub-workflow reference:** None

#### Fetch Schema

- **Type and technical role:** `n8n-nodes-base.postgres`  
  Executes a PostgreSQL query against `information_schema.columns`.
- **Configuration choices:**
  - Operation: `executeQuery`
  - Query retrieves:
    - `column_name`
    - `data_type`
    - `is_nullable`
    - `column_default`
  - Restricts to:
    - `table_schema = 'public'`
    - selected `table_name`
  - Orders by `ordinal_position`
  - Limits output to 120 columns
- **Key expressions or variables used:**
  - `{{ $('Manage Table Name').item.json.tableName }}`
- **Input and output connections:**
  - Input from `Need Schema?` true branch
  - Output to `Format Schema`
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires PostgreSQL credentials named `Postgres account`
- **Edge cases or potential failure types:**
  - Authentication or network failure to PostgreSQL
  - If the table is outside `public`, it will appear as missing
  - String interpolation in SQL could be risky in general, though upstream validation restricts identifier format
  - Tables with more than 120 columns are truncated
- **Sub-workflow reference:** None

#### Empty Schema

- **Type and technical role:** `n8n-nodes-base.set`  
  Supplies placeholder schema text when the workflow should not load schema yet.
- **Configuration choices:**
  - Sets:
    - `columnList = "(schema not loaded — table not yet configured)"`
    - `conceptMap = "(schema not loaded — table not yet configured)"`
  - Includes other incoming fields
- **Key expressions or variables used:** None beyond assigned values
- **Input and output connections:**
  - Input from `Need Schema?` false branch
  - Output to `Unified Analytics Agent`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - No significant operational failure expected
  - Placeholder content relies on the agent prompt to avoid tool use in invalid states
- **Sub-workflow reference:** None

---

## 2.3 Block: Schema Formatting and Column Mapping

### Overview

This block converts raw schema rows into a compact prompt-ready schema description. It also builds shorthand mappings and stores valid column names in static data for possible reuse.

### Nodes Involved

- Format Schema

### Node Details

#### Format Schema

- **Type and technical role:** `n8n-nodes-base.code`  
  Processes schema rows into human-readable and AI-usable instructions.
- **Configuration choices:**
  - Reads table context from `Manage Table Name`
  - Detects empty schema response and emits a user-facing warning
  - Formats each column with:
    - name
    - type
    - nullable status
    - default value
    - money-type cast warning
  - Creates shorthand mappings by stripping common suffixes such as `_eur`, `_usd`, `_share`, `_growth`, `_id`, `_name`, etc.
  - Stores:
    - `validColumns`
    - `validTableName`
    in global static data
- **Key expressions or variables used:**
  - `$('Manage Table Name').first().json`
  - `$input.all()`
  - `$getWorkflowStaticData('global')`
  - Emits:
    - `chatInput`
    - `tableName`
    - `justSetTable`
    - `isReset`
    - `columnList`
    - `conceptMap`
- **Input and output connections:**
  - Input from `Fetch Schema`
  - Output to `Unified Analytics Agent`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Static data keys for valid columns are not session-scoped here, unlike table selection; in multi-user use this could overwrite shared values, though the current agent prompt relies on current item data rather than those static keys.
  - If the table is missing, it emits warning text instead of failing.
  - Schema truncation at 120 columns may omit columns needed for some queries.
  - Suffix-based shorthand mapping may create ambiguous conceptual hints.
- **Sub-workflow reference:** None

---

## 2.4 Block: AI Analytics Agent, LLM, Memory, SQL Tool, and Chart Tool

### Overview

This is the core conversational analytics block. It uses a system prompt to enforce state-specific behavior, schema-safe SQL generation, one retry for self-correction, and chart generation for every analytical answer.

### Nodes Involved

- Unified Analytics Agent
- OpenAI GPT-4o-mini
- Chat Memory
- Execute SQL Query
- QuickChart

### Node Details

#### Unified Analytics Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main AI orchestration node that receives context and invokes tools.
- **Configuration choices:**
  - Uses a detailed `systemMessage` with five operating states:
    1. No table selected
    2. Table just set
    3. Reset requested
    4. Table not found
    5. Analytics mode
  - Strongly constrains SQL generation to exact listed column names only.
  - Instructs the model not to call tools in non-analytics states.
  - In analytics mode:
    - must use `Execute SQL Query`
    - may self-correct once
    - must not exceed two total SQL tool calls
    - must not reveal SQL or raw database errors
  - Requires chart creation through QuickChart on every data reply.
- **Key expressions or variables used:**
  - `{{ $json.tableName ?? 'NOT SET' }}`
  - `{{ $json.justSetTable }}`
  - `{{ $json.isReset ?? false }}`
  - `{{ $json.chatInput }}`
  - `{{ $json.columnList }}`
  - `{{ $json.conceptMap ?? '(schema not loaded)' }}`
- **Input and output connections:**
  - Main input from `Format Schema` or `Empty Schema`
  - AI model input from `OpenAI GPT-4o-mini`
  - AI memory input from `Chat Memory`
  - AI tools:
    - `Execute SQL Query`
    - `QuickChart`
- **Version-specific requirements:**
  - Type version `3.1`
  - Requires n8n AI agent support with compatible tool and memory nodes
- **Edge cases or potential failure types:**
  - Prompt-based safety is not equivalent to strict SQL sandboxing
  - If the model ignores instructions, it may still produce poor SQL or omit chart calls
  - Long schemas may consume prompt budget
  - The instruction says “create chat” instead of “chart”, likely a typo, but context is clear
- **Sub-workflow reference:** None

#### OpenAI GPT-4o-mini

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Chat language model used by the agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0` for deterministic behavior
  - No built-in tools enabled here
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as language model to `Unified Analytics Agent`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI API credentials
- **Edge cases or potential failure types:**
  - Invalid API key
  - Rate limits
  - Model availability changes
  - Token/context limits for large schema prompts
- **Sub-workflow reference:** None

#### Chat Memory

- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryPostgresChat`  
  Persists recent chat history in PostgreSQL for multi-turn continuity.
- **Configuration choices:**
  - Session key uses the `Chat Trigger` session ID
  - Context window length is `6`
- **Key expressions or variables used:**
  - `={{ $('Chat Trigger').item.json.sessionId }}`
- **Input and output connections:**
  - Connected as memory to `Unified Analytics Agent`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires PostgreSQL credentials
- **Edge cases or potential failure types:**
  - Postgres auth/connectivity failure
  - If session IDs change between turns, memory continuity is lost
  - Memory table/schema requirements depend on node implementation and n8n version
- **Sub-workflow reference:** None

#### Execute SQL Query

- **Type and technical role:** `n8n-nodes-base.postgresTool`  
  Tool node callable by the AI agent to run generated SQL against PostgreSQL.
- **Configuration choices:**
  - Operation: `executeQuery`
  - Query is fully supplied by the AI through `$fromAI`
  - Tool description explicitly instructs the agent to:
    - send only a valid `SELECT` or `WITH` query,
    - avoid backticks and semicolons,
    - retry only once after an error,
    - treat failures as `{ "error": "<message>" }`
  - `continueOnFail` is enabled so the tool returns error information instead of aborting the workflow
- **Key expressions or variables used:**
  - `{{ $fromAI('sql_query', 'A complete valid PostgreSQL SELECT or WITH (CTE) query. No backticks, no semicolons.') }}`
- **Input and output connections:**
  - Connected as AI tool to `Unified Analytics Agent`
- **Version-specific requirements:**
  - Type version `2.5`
  - Requires PostgreSQL credentials
  - Requires AI tool-compatible n8n version
- **Edge cases or potential failure types:**
  - SQL syntax errors
  - Missing column/table errors
  - Type cast issues, especially `money`
  - Slow or expensive queries
  - This tool is not hard-restricted to read-only at database level; safety depends on prompt and permissions. In practice, the prompt asks for `SELECT`/`WITH`, but DB credentials should still be read-only where possible.
- **Sub-workflow reference:** None

#### QuickChart

- **Type and technical role:** `n8n-nodes-base.quickChartTool`  
  AI-callable chart generation tool for visualizing returned query results.
- **Configuration choices:**
  - Chart data is populated by the AI through `$fromAI`
  - `chartOptions` and `datasetOptions` are left default/empty
- **Key expressions or variables used:**
  - `={{ $fromAI('Data', ``, 'json') }}`
- **Input and output connections:**
  - Connected as AI tool to `Unified Analytics Agent`
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases or potential failure types:**
  - AI may produce invalid chart JSON
  - Returned data shape may not suit the chosen chart format
  - Large datasets may create unusable charts
- **Sub-workflow reference:** None

---

## 2.5 Block: Manual Data Sync from Google Sheets to PostgreSQL

### Overview

This branch manually refreshes a PostgreSQL table from a Google Sheet. It is independent from the chat analytics path and is intended for loading or replacing dataset contents.

### Nodes Involved

- Manual Trigger
- ⚙️ Config (Sync)
- Truncate Table
- Fetch Google Sheet
- Insert Rows to Postgres

### Node Details

#### Manual Trigger

- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for the sync branch.
- **Configuration choices:** Default
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Entry point for sync path
  - Outputs to `⚙️ Config (Sync)`
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases or potential failure types:**
  - None beyond manual execution context
- **Sub-workflow reference:** None

#### ⚙️ Config (Sync)

- **Type and technical role:** `n8n-nodes-base.set`  
  Stores the target PostgreSQL table name for the sync branch.
- **Configuration choices:**
  - Sets `table_name = "bmw_global_sales"`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Manual Trigger`
  - Output to `Truncate Table`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - If the table name is changed here but not elsewhere, sync and analytics branches may target different structures
- **Sub-workflow reference:** None

#### Truncate Table

- **Type and technical role:** `n8n-nodes-base.postgres`  
  Deletes all existing rows from the configured table before reload.
- **Configuration choices:**
  - Operation: `executeQuery`
  - Query: `TRUNCATE TABLE {{ $json.table_name }}`
- **Key expressions or variables used:**
  - `{{ $json.table_name }}`
- **Input and output connections:**
  - Input from `⚙️ Config (Sync)`
  - Output to `Fetch Google Sheet`
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires PostgreSQL credentials
- **Edge cases or potential failure types:**
  - Destructive operation; data loss if run unintentionally
  - Permission issues if user cannot truncate
  - Dynamic table name is not quoted; avoid special characters
- **Sub-workflow reference:** None

#### Fetch Google Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a specific Google Sheet.
- **Configuration choices:**
  - Uses a fixed document ID:
    - `1BSx2AmyI7DAqxOwFcYT0kG70rueSY_QVm56LeCasRpU`
  - Reads from sheet/tab `gid=0`
  - No custom options configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Truncate Table`
  - Output to `Insert Rows to Postgres`
- **Version-specific requirements:**
  - Type version `4.7`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - OAuth token issues
  - Missing sheet permissions
  - Header names must match expected mappings downstream
- **Sub-workflow reference:** None

#### Insert Rows to Postgres

- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts rows from Google Sheets into PostgreSQL table `public.bmw_global_sales`.
- **Configuration choices:**
  - Operation: insert rows into selected table
  - Schema: `public`
  - Table: `bmw_global_sales`
  - Explicit field mapping:
    - `year ← Year`
    - `model ← Model`
    - `month ← Month`
    - `region ← Region`
    - `bev_share ← BEV_Share`
    - `gdp_growth ← GDP_Growth`
    - `units_sold ← Units_Sold`
    - `revenue_eur ← Revenue_EUR`
    - `avg_price_eur ← Avg_Price_EUR`
    - `premium_share ← Premium_Share`
    - `fuel_price_index ← Fuel_Price_Index`
  - Type conversion attempts are disabled
- **Key expressions or variables used:**
  - Expressions such as `={{ $json.Year }}`
- **Input and output connections:**
  - Input from `Fetch Google Sheet`
  - No downstream node
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires PostgreSQL credentials
- **Edge cases or potential failure types:**
  - Data type mismatches because conversion is disabled
  - Empty rows or malformed spreadsheet values
  - If the target schema/table differs, inserts fail
  - Revenue and average price are mapped as strings in schema metadata, which may need matching DB column types or database-side conversion
- **Sub-workflow reference:** None

---

## 2.6 Block: Documentation and In-Canvas Notes

### Overview

These sticky notes document workflow purpose, usage, setup, and warnings directly in the canvas. They are not executable but are important for interpretation and maintenance.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  General workflow description and setup guidance.
- **Configuration choices:**
  - Large descriptive note covering purpose, setup, and customization
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the input/session block.
- **Configuration choices:** White note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the schema/input management area.
- **Configuration choices:** White note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the AI and charting block.
- **Configuration choices:** White note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the Google Sheets sync branch.
- **Configuration choices:** White note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note5

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Warning note about destructive truncation.
- **Configuration choices:** Red note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Chat Trigger | `@n8n/n8n-nodes-langchain.chatTrigger` | Public chat entry point for user messages |  | Manage Table Name | Handles user input and session tracking  |
| Chat Trigger | `@n8n/n8n-nodes-langchain.chatTrigger` | Public chat entry point for user messages |  | Manage Table Name | Stores table name to avoid repeated input |
| Manage Table Name | `n8n-nodes-base.code` | Per-session table selection, reset handling, and message normalization | Chat Trigger | Need Schema? | Handles user input and session tracking  |
| Manage Table Name | `n8n-nodes-base.code` | Per-session table selection, reset handling, and message normalization | Chat Trigger | Need Schema? | Stores table name to avoid repeated input |
| Need Schema? | `n8n-nodes-base.if` | Routes to schema fetch or placeholder schema content | Manage Table Name | Fetch Schema, Empty Schema | Handles user input and session tracking  |
| Need Schema? | `n8n-nodes-base.if` | Routes to schema fetch or placeholder schema content | Manage Table Name | Fetch Schema, Empty Schema | Stores table name to avoid repeated input |
| Fetch Schema | `n8n-nodes-base.postgres` | Reads PostgreSQL column metadata from `information_schema` | Need Schema? | Format Schema | Handles user input and session tracking  |
| Fetch Schema | `n8n-nodes-base.postgres` | Reads PostgreSQL column metadata from `information_schema` | Need Schema? | Format Schema | Stores table name to avoid repeated input |
| Format Schema | `n8n-nodes-base.code` | Formats schema text and shorthand column hints for the AI agent | Fetch Schema | Unified Analytics Agent | Handles user input and session tracking  |
| Format Schema | `n8n-nodes-base.code` | Formats schema text and shorthand column hints for the AI agent | Fetch Schema | Unified Analytics Agent | Stores table name to avoid repeated input |
| Empty Schema | `n8n-nodes-base.set` | Supplies fallback schema text before a table is configured | Need Schema? | Unified Analytics Agent | Handles user input and session tracking  |
| Empty Schema | `n8n-nodes-base.set` | Supplies fallback schema text before a table is configured | Need Schema? | Unified Analytics Agent | Stores table name to avoid repeated input |
| Unified Analytics Agent | `@n8n/n8n-nodes-langchain.agent` | Conversational analytics orchestrator with state-aware tool use | Format Schema, Empty Schema |  | Generates SQL from natural language  |
| Unified Analytics Agent | `@n8n/n8n-nodes-langchain.agent` | Conversational analytics orchestrator with state-aware tool use | Format Schema, Empty Schema |  | Executes queries and formats results |
| Unified Analytics Agent | `@n8n/n8n-nodes-langchain.agent` | Conversational analytics orchestrator with state-aware tool use | Format Schema, Empty Schema |  | Creates charts from query results  |
| Unified Analytics Agent | `@n8n/n8n-nodes-langchain.agent` | Conversational analytics orchestrator with state-aware tool use | Format Schema, Empty Schema |  | Helps interpret data visually |
| OpenAI GPT-4o-mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM used by the analytics agent |  | Unified Analytics Agent | Generates SQL from natural language  |
| OpenAI GPT-4o-mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM used by the analytics agent |  | Unified Analytics Agent | Executes queries and formats results |
| OpenAI GPT-4o-mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM used by the analytics agent |  | Unified Analytics Agent | Creates charts from query results  |
| OpenAI GPT-4o-mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM used by the analytics agent |  | Unified Analytics Agent | Helps interpret data visually |
| Chat Memory | `@n8n/n8n-nodes-langchain.memoryPostgresChat` | Stores recent conversation state in PostgreSQL |  | Unified Analytics Agent | Generates SQL from natural language  |
| Chat Memory | `@n8n/n8n-nodes-langchain.memoryPostgresChat` | Stores recent conversation state in PostgreSQL |  | Unified Analytics Agent | Executes queries and formats results |
| Chat Memory | `@n8n/n8n-nodes-langchain.memoryPostgresChat` | Stores recent conversation state in PostgreSQL |  | Unified Analytics Agent | Creates charts from query results  |
| Chat Memory | `@n8n/n8n-nodes-langchain.memoryPostgresChat` | Stores recent conversation state in PostgreSQL |  | Unified Analytics Agent | Helps interpret data visually |
| Execute SQL Query | `n8n-nodes-base.postgresTool` | AI-callable PostgreSQL query execution tool |  | Unified Analytics Agent | Generates SQL from natural language  |
| Execute SQL Query | `n8n-nodes-base.postgresTool` | AI-callable PostgreSQL query execution tool |  | Unified Analytics Agent | Executes queries and formats results |
| Execute SQL Query | `n8n-nodes-base.postgresTool` | AI-callable PostgreSQL query execution tool |  | Unified Analytics Agent | Creates charts from query results  |
| Execute SQL Query | `n8n-nodes-base.postgresTool` | AI-callable PostgreSQL query execution tool |  | Unified Analytics Agent | Helps interpret data visually |
| QuickChart | `n8n-nodes-base.quickChartTool` | AI-callable chart generation tool |  | Unified Analytics Agent | Generates SQL from natural language  |
| QuickChart | `n8n-nodes-base.quickChartTool` | AI-callable chart generation tool |  | Unified Analytics Agent | Executes queries and formats results |
| QuickChart | `n8n-nodes-base.quickChartTool` | AI-callable chart generation tool |  | Unified Analytics Agent | Creates charts from query results  |
| QuickChart | `n8n-nodes-base.quickChartTool` | AI-callable chart generation tool |  | Unified Analytics Agent | Helps interpret data visually |
| Manual Trigger | `n8n-nodes-base.manualTrigger` | Manual start for the Google Sheets sync branch |  | ⚙️ Config (Sync) | Loads data from Google Sheets into PostgreSQL  |
| Manual Trigger | `n8n-nodes-base.manualTrigger` | Manual start for the Google Sheets sync branch |  | ⚙️ Config (Sync) | Useful for scheduled or batch updates |
| ⚙️ Config (Sync) | `n8n-nodes-base.set` | Stores target table name for sync operations | Manual Trigger | Truncate Table | Loads data from Google Sheets into PostgreSQL  |
| ⚙️ Config (Sync) | `n8n-nodes-base.set` | Stores target table name for sync operations | Manual Trigger | Truncate Table | Useful for scheduled or batch updates |
| Truncate Table | `n8n-nodes-base.postgres` | Clears the target PostgreSQL table before reload | ⚙️ Config (Sync) | Fetch Google Sheet | Loads data from Google Sheets into PostgreSQL  |
| Truncate Table | `n8n-nodes-base.postgres` | Clears the target PostgreSQL table before reload | ⚙️ Config (Sync) | Fetch Google Sheet | Useful for scheduled or batch updates |
| Truncate Table | `n8n-nodes-base.postgres` | Clears the target PostgreSQL table before reload | ⚙️ Config (Sync) | Fetch Google Sheet | ⚠️ Truncate Table will delete existing data  |
| Truncate Table | `n8n-nodes-base.postgres` | Clears the target PostgreSQL table before reload | ⚙️ Config (Sync) | Fetch Google Sheet | Disable or remove if not required |
| Fetch Google Sheet | `n8n-nodes-base.googleSheets` | Reads source rows from a Google Sheet | Truncate Table | Insert Rows to Postgres | Loads data from Google Sheets into PostgreSQL  |
| Fetch Google Sheet | `n8n-nodes-base.googleSheets` | Reads source rows from a Google Sheet | Truncate Table | Insert Rows to Postgres | Useful for scheduled or batch updates |
| Insert Rows to Postgres | `n8n-nodes-base.postgres` | Inserts mapped Google Sheets rows into PostgreSQL | Fetch Google Sheet |  | Loads data from Google Sheets into PostgreSQL  |
| Insert Rows to Postgres | `n8n-nodes-base.postgres` | Inserts mapped Google Sheets rows into PostgreSQL | Fetch Google Sheet |  | Useful for scheduled or batch updates |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | This workflow lets you query your PostgreSQL database using natural language. |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Instead of writing SQL manually or exporting data, users can simply type questions and receive structured answers with charts. The workflow dynamically reads the table schema, ensures only valid columns are used, and generates safe SQL queries. |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Users can connect to any table directly from chat by typing the table name. The workflow remembers the table for the session, so it does not need to be repeated. To switch tables, users can type `reset table` at any time. |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ### How it works |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - User sends a message via chat |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - Table name is captured and stored per session |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - Schema is fetched once and reused for efficiency |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - AI generates SQL using only valid columns |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - Query is executed and results are returned |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - Charts are generated for visualization |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ### Setup |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | 1. Connect your PostgreSQL credentials |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | 2. Ensure your table exists in the `public` schema |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | 3. (Optional) Configure Google Sheets sync for data ingestion |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | 4. Replace the default table name if using sync workflow |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | 5. Start by typing a table name, then ask questions |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ### Customization |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - Modify system prompt for stricter SQL rules |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - Extend to support multiple schemas |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | - Replace the LLM with any supported model |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Handles user input and session tracking  |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Stores table name to avoid repeated input |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Handles user input and session tracking  |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Stores table name to avoid repeated input |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Generates SQL from natural language  |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Executes queries and formats results |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Creates charts from query results  |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Helps interpret data visually |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Loads data from Google Sheets into PostgreSQL  |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Useful for scheduled or batch updates |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ⚠️ Truncate Table will delete existing data  |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | Disable or remove if not required |

---

# 4. Reproducing the Workflow from Scratch

## A. Build the chat analytics branch

1. **Create a `Chat Trigger` node**
   - Type: `Chat Trigger`
   - Enable public chat access.
   - Keep default options unless you need custom auth or UI behavior.

2. **Create a `Code` node named `Manage Table Name`**
   - Connect `Chat Trigger -> Manage Table Name`.
   - Paste the JavaScript logic that:
     - reads `sessionId` and `chatInput`,
     - stores table state in global static data under a session-specific key,
     - recognizes `reset table`, `change table`, `switch table`,
     - strips schema prefixes like `public.my_table`,
     - validates identifiers,
     - blocks common words,
     - outputs:
       - `chatInput`
       - `tableName`
       - `justSetTable`
       - `isReset`
       - `sessionId`

3. **Create an `If` node named `Need Schema?`**
   - Connect `Manage Table Name -> Need Schema?`.
   - Configure an `AND` condition with:
     - `tableName` is not empty
     - `justSetTable` is not equal to `true`
     - `isReset` is not equal to `true`

4. **Create a `Postgres` node named `Fetch Schema`**
   - Connect the true output of `Need Schema?` to `Fetch Schema`.
   - Set operation to `Execute Query`.
   - Use PostgreSQL credentials.
   - Query the `information_schema.columns` table for the selected table in schema `public`.
   - Include:
     - `column_name`
     - `data_type`
     - `is_nullable`
     - `column_default`
   - Filter by the selected table name from `Manage Table Name`.
   - Order by `ordinal_position`.
   - Limit to 120 rows.

5. **Create a `Code` node named `Format Schema`**
   - Connect `Fetch Schema -> Format Schema`.
   - Add logic that:
     - reads the schema rows,
     - formats a readable column list,
     - marks `money` columns with a cast warning,
     - builds shorthand mappings by stripping common suffixes,
     - outputs:
       - `chatInput`
       - `tableName`
       - `justSetTable`
       - `isReset`
       - `columnList`
       - `conceptMap`
   - Include a fallback message if the table is not found.

6. **Create a `Set` node named `Empty Schema`**
   - Connect the false output of `Need Schema?` to `Empty Schema`.
   - Keep other incoming fields.
   - Set:
     - `columnList = (schema not loaded — table not yet configured)`
     - `conceptMap = (schema not loaded — table not yet configured)`

7. **Create an `AI Agent` node named `Unified Analytics Agent`**
   - Connect:
     - `Format Schema -> Unified Analytics Agent`
     - `Empty Schema -> Unified Analytics Agent`
   - Add a system message that includes:
     - state variables (`tableName`, `justSetTable`, `isReset`, `chatInput`)
     - exact columns list
     - shorthand translations
     - explicit no-tool behavior for setup/reset/not-found states
     - SQL-generation rules for analytics mode
     - self-correction behavior limited to two SQL tool calls total
     - output formatting rules
     - instruction to always generate a chart for analytical answers

8. **Create an `OpenAI Chat Model` node named `OpenAI GPT-4o-mini`**
   - Connect it to the AI Agent as the language model.
   - Choose model `gpt-4o-mini`.
   - Set temperature to `0`.
   - Configure OpenAI credentials.

9. **Create a `Postgres Chat Memory` node named `Chat Memory`**
   - Connect it to the AI Agent as memory.
   - Use PostgreSQL credentials.
   - Set session key to the `Chat Trigger` session ID.
   - Set context window length to `6`.

10. **Create a `Postgres Tool` node named `Execute SQL Query`**
    - Connect it to the AI Agent as an AI tool.
    - Use PostgreSQL credentials.
    - Set operation to `Execute Query`.
    - Query should come from `$fromAI(...)`.
    - Tool description should clearly state:
      - only `SELECT` or `WITH` queries,
      - no semicolons,
      - no backticks,
      - on error, retry only once,
      - never exceed two total calls.
    - Enable `Continue On Fail`.

11. **Create a `QuickChart Tool` node named `QuickChart`**
    - Connect it to the AI Agent as an AI tool.
    - Set chart data from `$fromAI(..., 'json')`.
    - Leave chart options and dataset options default unless you want stricter chart templates.

12. **Test the chat flow**
    - Start the workflow.
    - In chat, type a table name such as `bmw_global_sales`.
    - Then ask a question like:
      - “Show total units sold by year”
      - “What is the average revenue by region?”
    - Confirm:
      - the table selection persists by session,
      - schema is loaded only when needed,
      - results include both narrative output and a chart.

---

## B. Build the optional Google Sheets sync branch

13. **Create a `Manual Trigger` node**
   - This is a separate entry point for loading data from a sheet.

14. **Create a `Set` node named `⚙️ Config (Sync)`**
   - Connect `Manual Trigger -> ⚙️ Config (Sync)`.
   - Add a field:
     - `table_name = bmw_global_sales`
   - Change this if your target table is different.

15. **Create a `Postgres` node named `Truncate Table`**
   - Connect `⚙️ Config (Sync) -> Truncate Table`.
   - Use PostgreSQL credentials.
   - Set operation to `Execute Query`.
   - Query:
     - `TRUNCATE TABLE {{ $json.table_name }}`
   - Only keep this node if full replacement is intended.

16. **Create a `Google Sheets` node named `Fetch Google Sheet`**
   - Connect `Truncate Table -> Fetch Google Sheet`.
   - Configure Google Sheets OAuth2 credentials.
   - Select the spreadsheet document.
   - Select the worksheet/tab to read.
   - Ensure the sheet headers match the field mapping used later.

17. **Create a `Postgres` node named `Insert Rows to Postgres`**
   - Connect `Fetch Google Sheet -> Insert Rows to Postgres`.
   - Set:
     - schema: `public`
     - table: `bmw_global_sales`
   - Map spreadsheet columns to DB columns:
     - `Year -> year`
     - `Model -> model`
     - `Month -> month`
     - `Region -> region`
     - `BEV_Share -> bev_share`
     - `GDP_Growth -> gdp_growth`
     - `Units_Sold -> units_sold`
     - `Revenue_EUR -> revenue_eur`
     - `Avg_Price_EUR -> avg_price_eur`
     - `Premium_Share -> premium_share`
     - `Fuel_Price_Index -> fuel_price_index`
   - Leave automatic type conversion disabled if your database already expects matching formats; otherwise enable conversion carefully.

18. **Test the sync branch**
   - Run manually.
   - Verify:
     - the truncate step is acceptable,
     - rows are fetched correctly,
     - inserts succeed without type mismatch,
     - the chat branch can query the same table afterward.

---

## C. Credentials required

19. **PostgreSQL credentials**
   - Used by:
     - `Fetch Schema`
     - `Chat Memory`
     - `Execute SQL Query`
     - `Truncate Table`
     - `Insert Rows to Postgres`
   - Recommended:
     - use a read-only DB user for analytics nodes,
     - use a separate write-capable user for sync nodes if possible.

20. **OpenAI credentials**
   - Used by `OpenAI GPT-4o-mini`
   - Ensure the account has access to `gpt-4o-mini`.

21. **Google Sheets OAuth2 credentials**
   - Used by `Fetch Google Sheet`
   - Ensure the spreadsheet is accessible to the authorized Google account.

---

## D. Important implementation constraints

22. **The analytics path assumes tables are in the `public` schema**
   - If you need multiple schemas, update:
     - table capture logic,
     - schema query,
     - and possibly the agent prompt.

23. **Use safe database permissions**
   - The SQL tool is prompt-constrained, not database-enforced.
   - Restrict the analytics credential to read-only access.

24. **Be careful with static data**
   - Session table names are namespaced per `sessionId`.
   - Schema validity data in `Format Schema` is stored globally without session namespacing.

25. **Preserve exact column-name enforcement**
   - This is a major reliability mechanism.
   - If you relax it, hallucinated columns become more likely.

26. **Review chart generation behavior**
   - The agent is instructed to always produce a chart.
   - For some single-value answers, you may want to relax that rule.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow lets you query your PostgreSQL database using natural language. | Canvas note |
| Users can connect to any table directly from chat by typing the table name. The workflow remembers the table for the session, so it does not need to be repeated. To switch tables, users can type `reset table` at any time. | Canvas note |
| Setup: Connect PostgreSQL credentials, ensure your table exists in the `public` schema, optionally configure Google Sheets sync, replace the default table name if needed, then start by typing a table name. | Canvas note |
| Customization ideas: modify the system prompt for stricter SQL rules, extend to support multiple schemas, or replace the LLM with another supported model. | Canvas note |
| Warning: `Truncate Table` deletes existing data. Disable or remove it if full refresh is not required. | Canvas note |

If you want, I can also convert this into:
1. a more compact engineering specification,
2. a node-by-node diff-friendly format,
3. or a machine-readable structured inventory for AI agents.