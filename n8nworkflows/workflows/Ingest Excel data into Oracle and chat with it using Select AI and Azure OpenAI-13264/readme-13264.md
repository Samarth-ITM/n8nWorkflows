Ingest Excel data into Oracle and chat with it using Select AI and Azure OpenAI

https://n8nworkflows.xyz/workflows/ingest-excel-data-into-oracle-and-chat-with-it-using-select-ai-and-azure-openai-13264


# Ingest Excel data into Oracle and chat with it using Select AI and Azure OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Ingest Excel data into Oracle and chat with it using Select AI and Azure OpenAI  
**Workflow name (JSON):** AI-powered Excel data ingestion and chat with Oracle Select AI

This workflow has **two entry points**:

### 1.1 Workflow A — Excel ingestion into Oracle + Select AI registration (Webhook upload)
- Receives an uploaded Excel file via HTTP POST
- Validates and normalizes the binary file payload
- Extracts rows from the Excel file
- Infers an Oracle-compatible schema (columns/types) and generates a unique table name
- Creates the Oracle table dynamically
- Inserts rows in batches
- Creates/overwrites an **Oracle Select AI profile** configured for **Azure OpenAI**
- Registers the created table in the profile object list (enforced)
- Returns a success JSON response to the uploader

### 1.2 Workflow B — Chat with the ingested data (Chat trigger)
- Accepts a natural-language question from the n8n chat interface
- Uses Oracle Select AI (`DBMS_CLOUD_AI.GENERATE` with `action='runsql'`) to translate the question into SQL and execute it
- Formats and returns the result to the chat UI

---

## 2. Block-by-Block Analysis

### Block A1 — File upload reception & binary normalization
**Overview:** Accepts the Excel file via webhook and ensures downstream nodes receive a consistent binary key (`binary.data`) with a correct MIME type and extension.

**Nodes involved:**
- Webhook - File Upload
- Validate & Normalize File

#### Node: Webhook - File Upload
- **Type / role:** `Webhook` (HTTP entry point for file uploads)
- **Configuration (interpreted):**
  - Method: `POST`
  - Path: `/upload-excel`
  - Response mode: `lastNode` (response returned by final node in this branch)
  - Raw body disabled
- **Inputs/Outputs:**
  - Output → `Validate & Normalize File`
- **Edge cases / failures:**
  - No file sent → downstream Code node throws
  - Large uploads may hit n8n/server reverse proxy limits
- **Version notes:** typeVersion `1.1`

#### Node: Validate & Normalize File
- **Type / role:** `Code` (validates upload + normalizes binary property name)
- **Key logic:**
  - Ensures `item.binary` exists and is non-empty
  - Detects the first binary property name (webhook can name it variably)
  - Validates file extension is `.xlsx` or `.xls`
  - Sets `mimeType` accordingly:
    - `.xlsx` → `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
    - `.xls` → `application/vnd.ms-excel`
  - Copies binary entry to `item.binary.data` if needed (standardizes downstream consumption)
- **Inputs/Outputs:**
  - Input ← `Webhook - File Upload`
  - Output → `Extract from Excel`
- **Edge cases / failures:**
  - No binary data / empty binary object → throws explicit error
  - Unsupported extension (e.g., `.csv`) → throws explicit error
  - **Sticky note suggests** adding file size validation here (not implemented)
- **Version notes:** typeVersion `2`

---

### Block A2 — Excel extraction & schema inference
**Overview:** Reads spreadsheet content into JSON rows, then infers Oracle column names/types and creates a unique table name.

**Nodes involved:**
- Extract from Excel
- Infer Schema & Table Name

#### Node: Extract from Excel
- **Type / role:** `Extract From File` (parses Excel binary into JSON rows)
- **Configuration:**
  - Operation: `xlsx` (reads Excel workbook)
- **Inputs/Outputs:**
  - Input ← `Validate & Normalize File` (expects binary at `binary.data`)
  - Output → `Infer Schema & Table Name`
- **Edge cases / failures:**
  - Invalid/corrupt Excel file
  - Password-protected spreadsheets (typically fail)
  - Unexpected sheet format may yield empty extraction
- **Version notes:** typeVersion `1`

#### Node: Infer Schema & Table Name
- **Type / role:** `Code` (infers Oracle schema and generates table name)
- **Key logic:**
  - Loads all extracted rows: `rows = $input.all().map(i => i.json)`
  - Fails if empty: “Excel file is empty”
  - Sanitizes column names for Oracle:
    - Trim
    - Replace non-alphanumeric with `_`
    - Strip leading/trailing `_`
    - Uppercase
  - Infers type from **first row only**:
    - number → `NUMBER`
    - date-like string (parsable by `Date.parse`) → `DATE`
    - otherwise → `VARCHAR2(4000)`
  - Cleans all rows to use sanitized keys
  - Generates table name: `UPLOAD_EXCEL_<timestamp>` using ISO timestamp stripped of separators
- **Outputs:**
  - Single item with:
    - `tableName` (string)
    - `columns` (array of `{name,type}`)
    - `rows` (array of cleaned row objects)
- **Inputs/Outputs:**
  - Input ← `Extract from Excel`
  - Output → `Create Oracle Table`
- **Edge cases / failures:**
  - Type inference may be wrong if first row has blanks/odd formatting
  - Column name collisions after sanitization (e.g., “First-Name” and “First Name”) → later inserts may misbehave
  - Oracle reserved words used as column names (not handled)
  - Oracle identifier length limits (not handled)
- **Version notes:** typeVersion `2`

---

### Block A3 — Oracle table creation & row insertion (batched)
**Overview:** Creates the inferred table in Oracle, then restores the cleaned rows and inserts them in batches (default 50).

**Nodes involved:**
- Create Oracle Table
- Restore Data Rows
- Split into Batches
- Insert Rows into Oracle

#### Node: Create Oracle Table
- **Type / role:** `Oracle Database` (executes DDL)
- **Configuration:**
  - Operation: `execute`
  - Query is generated via expression:
    - Creates columns from `$json.columns`
    - Adds `UPLOADED_AT TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
    - SQL: `CREATE TABLE <tableName> (<col defs>, UPLOADED_AT ...)`
- **Credentials:**
  - Uses “Oracle Database Credentials account”
- **Connections:**
  - Input ← `Infer Schema & Table Name`
  - Output → `Split into Batches` **and** `Restore Data Rows` (two parallel downstream connections from the same output)
- **Edge cases / failures:**
  - Insufficient privileges to create tables
  - Table name already exists (unlikely but possible)
  - Invalid identifiers due to sanitization gaps (reserved words, length)
  - DDL lock/contention
- **Version notes:** typeVersion `1`

#### Node: Restore Data Rows
- **Type / role:** `Code` (converts stored `rows` array back into n8n items)
- **Key logic:**
  - Reads from the earlier node: `$("Infer Schema & Table Name").first().json`
  - Returns `infer.rows.map(r => ({ json: r }))`
- **Connections:**
  - Input ← `Create Oracle Table` (note: it does not use Create’s output; it uses cross-node reference)
  - Output → `Split into Batches`
- **Edge cases / failures:**
  - If “Infer Schema & Table Name” produced no rows, earlier block already throws
  - If node names change, the expression reference breaks
- **Version notes:** typeVersion `2`

#### Node: Split into Batches
- **Type / role:** `Split In Batches` (batch control for inserts)
- **Configuration:**
  - `batchSize: 50`
- **Connections:**
  - Input ← `Create Oracle Table` **and** `Restore Data Rows`
  - Output → `Insert Rows into Oracle`
- **Important behavior note:**
  - This node is designed to be used with a loop (execute batch → “Continue” back to SplitInBatches).  
  - **This workflow does not include the loop-back connection**, so it may only process the first batch unless n8n’s execution path happens to iterate differently. In most setups, you should add a loop (see Section 4).
- **Edge cases / failures:**
  - Very large datasets may still time out or exceed memory without proper looping/commit strategy
- **Version notes:** typeVersion `3`

#### Node: Insert Rows into Oracle
- **Type / role:** `Oracle Database` (inserts JSON rows into the new table)
- **Configuration:**
  - Uses “Insert” operation via table mapping (not raw SQL)
  - Table name expression: `$("Infer Schema & Table Name").first().json.tableName`
  - Schema name expression: `$json.schema || 'YOUR_SCHEMA_NAME'`
  - Column mapping: `autoMapInputData`
- **Credentials:** Oracle credentials
- **Connections:**
  - Input ← `Split into Batches`
  - Output → `Configure Select AI Settings`
- **Edge cases / failures:**
  - **Schema default placeholder** (`YOUR_SCHEMA_NAME`) will fail unless changed
  - Insert can fail due to:
    - data type mismatch (DATE parsing, NUMBER parsing)
    - column length overflow (VARCHAR2(4000) still may overflow if text > 4000 bytes/chars depending on DB settings)
    - invalid date strings
  - Bulk insert behavior may vary depending on node implementation/version
- **Version notes:** typeVersion `1`

---

### Block A4 — Select AI profile configuration & registration
**Overview:** Builds a configuration object for Select AI (Azure OpenAI provider) and registers/overwrites a Select AI profile limiting access to the uploaded table.

**Nodes involved:**
- Configure Select AI Settings
- Register with Select AI

#### Node: Configure Select AI Settings
- **Type / role:** `Set` (creates a structured configuration object)
- **Configuration:**
  - Creates `selectAIConfig` object:
    - `profile_name`: `EXCEL_AI`
    - `provider`: `azure`
    - `azure_resource_name`: placeholder `YOUR_AZURE_RESOURCE_NAME`
    - `azure_deployment_name`: placeholder `YOUR_DEPLOYMENT_NAME`
    - `credential_name`: placeholder `YOUR_ORACLE_CREDENTIAL_NAME`
    - `table_name`: from inferred `tableName`
- **Connections:**
  - Input ← `Insert Rows into Oracle`
  - Output → `Register with Select AI`
- **Edge cases / failures:**
  - If you do not replace placeholders, registration will fail in the next node
  - If table name contains characters Oracle disallows (unlikely given generation), profile creation may fail
- **Version notes:** typeVersion `3.4`

#### Node: Register with Select AI
- **Type / role:** `Oracle Database` (PL/SQL block to create Select AI profile)
- **Configuration:**
  - Operation: `execute`
  - PL/SQL does:
    - Reads current schema owner via `SYS_CONTEXT('USERENV', 'CURRENT_USER')`
    - Attempts to drop existing profile `DBMS_CLOUD_AI.DROP_PROFILE` (errors ignored)
    - Creates profile `DBMS_CLOUD_AI.CREATE_PROFILE` with JSON attributes:
      - provider, azure_resource_name, azure_deployment_name, credential_name
      - `object_list`: array with `{owner: current_user, name: table_name}`
      - `enforce_object_list`: TRUE (restricts accessible objects)
- **Credentials:** Oracle credentials
- **Connections:**
  - Input ← `Configure Select AI Settings`
  - Output → `Prepare Response`
- **Edge cases / failures:**
  - Oracle database must have Select AI / `DBMS_CLOUD_AI` available and user must have privileges
  - `credential_name` must exist in DB (created via `DBMS_CLOUD.CREATE_CREDENTIAL`)
  - Profile name conflicts: node drops and recreates, but drop may fail due to privileges
  - JSON quoting: values are injected via templating; unusual characters could break if not escaped (profile/table names should be safe here)
- **Version notes:** typeVersion `1`

---

### Block A5 — HTTP response building
**Overview:** Builds a structured success response with next steps, then responds to the upload webhook.

**Nodes involved:**
- Prepare Response
- Return Success

#### Node: Prepare Response
- **Type / role:** `Code` (formats final response JSON)
- **Key logic:**
  - Reads infer results from `Infer Schema & Table Name`
  - Reads `selectAIConfig` from current item (`$json.selectAIConfig`)
  - Returns:
    - `success`, `tableName`, `columns`, `rowCount`, `selectAIProfile`, `message`, `nextSteps`
- **Connections:**
  - Input ← `Register with Select AI`
  - Output → `Return Success`
- **Edge cases / failures:**
  - If node names change, the cross-node reference breaks
- **Version notes:** typeVersion `2`

#### Node: Return Success
- **Type / role:** `Respond to Webhook` (HTTP response emitter)
- **Configuration:** returns output of previous node to the webhook caller
- **Connections:**
  - Input ← `Prepare Response`
- **Edge cases / failures:**
  - If workflow errors earlier, caller receives error response instead
- **Version notes:** typeVersion `1`

---

### Block B1 — Chat trigger, Select AI query execution, response formatting
**Overview:** Accepts a chat prompt, runs it through Oracle Select AI to generate and execute SQL, then formats the output for the chat UI.

**Nodes involved:**
- Chat Input
- Configure Select AI Profile
- Oracle Select AI Query
- Format Chat Response

#### Node: Chat Input
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` (public chat entry point)
- **Configuration:**
  - Public: `true`
  - Webhook ID: `chat-with-excel-ai`
- **Connections:**
  - Output → `Configure Select AI Profile`
- **Edge cases / failures:**
  - Public endpoint can be abused; consider auth/rate limiting
- **Version notes:** typeVersion `1`

#### Node: Configure Select AI Profile
- **Type / role:** `Set` (chooses Select AI profile name)
- **Configuration:**
  - Sets `profileName = "EXCEL_AI"`
- **Connections:**
  - Input ← `Chat Input`
  - Output → `Oracle Select AI Query`
- **Edge cases / failures:**
  - If ingestion created a different profile name, chat won’t find the right profile
- **Version notes:** typeVersion `3.4`

#### Node: Oracle Select AI Query
- **Type / role:** `Oracle Database` (executes Select AI call)
- **Configuration:**
  - Operation: `execute`
  - SQL expression builds:
    - `DBMS_CLOUD_AI.GENERATE(prompt => '<escaped chatInput>', profile_name => '<profileName>', action => 'runsql')`
    - Escapes single quotes in prompt: `replace(/'/g, "''")`
    - Returns `AS RESPONSE FROM dual`
- **Connections:**
  - Input ← `Configure Select AI Profile`
  - Output → `Format Chat Response`
- **Edge cases / failures:**
  - `profileName` not found / not accessible
  - Select AI privilege/feature unavailable
  - Prompt leads to invalid SQL or SQL that fails at execution time
  - Output format may vary; downstream expects `RESPONSE` or first value
- **Version notes:** typeVersion `1`

#### Node: Format Chat Response
- **Type / role:** `Code` (extracts `RESPONSE` and formats for chat)
- **Key logic:**
  - `const response = $json.RESPONSE || Object.values($json)[0];`
  - If missing: returns message asking to rephrase
  - Else returns `{ message: response }`
- **Connections:** terminal node for chat path
- **Edge cases / failures:**
  - If Oracle node returns nested structures, `Object.values($json)[0]` may not be what you want
- **Version notes:** typeVersion `2`

---

### Block C — Documentation / guidance notes (Sticky Notes)
**Overview:** Sticky notes are not executable but provide required configuration guidance and expectations.

**Nodes involved (all are sticky notes):**
- Sticky Note (Workflow A Flow)
- Sticky Note1 (Workflow B How It Works)
- Sticky Note2 (Configure Oracle Database Credentials)
- Sticky Note3 (Update Schema Name)
- Sticky Note4 (Configure Azure OpenAI Settings)
- Sticky Note5 (File Size Limits)
- Sticky Note6 (Registers with select AI)
- Sticky Note7 (Return Success Output)
- Sticky Note8 (Change Batch Size)
- Sticky Note9 (Oracle table and Select AI profile needed)
- Sticky Note10 (Expected Response)
- Sticky Note11 (DBMS_CLOUD_AI.GENERATE explanation)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - File Upload | Webhook | Receives Excel file via HTTP POST | — | Validate & Normalize File | ## Workflow A Flow: 1. Webhook receives Excel file ↓ 2. Validate & normalize file ↓ 3. Extract data from Excel ↓ 4. Infer schema (column names & types) ↓ 5. Create Oracle table dynamically ↓ 6. Insert data in batches (50 rows at a time) ↓ 7. Configure Select AI settings ↓ 8. Register table with Select AI ↓ 9. Return success response |
| Validate & Normalize File | Code | Validates file presence/type, normalizes binary key to `binary.data` | Webhook - File Upload | Extract from Excel | ### File Size Limits Add validation in "Validate & Normalize File" |
| Extract from Excel | Extract From File | Parses Excel into JSON rows | Validate & Normalize File | Infer Schema & Table Name | ## Workflow A Flow: 1. Webhook receives Excel file ↓ 2. Validate & normalize file ↓ 3. Extract data from Excel ↓ 4. Infer schema (column names & types) ↓ 5. Create Oracle table dynamically ↓ 6. Insert data in batches (50 rows at a time) ↓ 7. Configure Select AI settings ↓ 8. Register table with Select AI ↓ 9. Return success response |
| Infer Schema & Table Name | Code | Infers Oracle schema + generates unique table name | Extract from Excel | Create Oracle Table | ## Workflow A Flow: 1. Webhook receives Excel file ↓ 2. Validate & normalize file ↓ 3. Extract data from Excel ↓ 4. Infer schema (column names & types) ↓ 5. Create Oracle table dynamically ↓ 6. Insert data in batches (50 rows at a time) ↓ 7. Configure Select AI settings ↓ 8. Register table with Select AI ↓ 9. Return success response |
| Create Oracle Table | Oracle Database | Creates inferred table via dynamic DDL | Infer Schema & Table Name | Split into Batches; Restore Data Rows | Configure your Oracle Database credentials in the credentials panel |
| Restore Data Rows | Code | Expands stored `rows[]` into items for insertion | Create Oracle Table | Split into Batches | ## Workflow A Flow: 1. Webhook receives Excel file ↓ 2. Validate & normalize file ↓ 3. Extract data from Excel ↓ 4. Infer schema (column names & types) ↓ 5. Create Oracle table dynamically ↓ 6. Insert data in batches (50 rows at a time) ↓ 7. Configure Select AI settings ↓ 8. Register table with Select AI ↓ 9. Return success response |
| Split into Batches | Split In Batches | Processes rows in batches (default 50) | Create Oracle Table; Restore Data Rows | Insert Rows into Oracle | ### Change Batch Size Default: 50 rows per batch Adjust based on your data size and database performance |
| Insert Rows into Oracle | Oracle Database | Inserts batch rows into the generated table | Split into Batches | Configure Select AI Settings | ### Update Schema Name In the "Insert Rows into Oracle" node: Change YOUR_SCHEMA_NAME to your actual Oracle schema (e.g., ADMIN, SCOTT, etc.) |
| Configure Select AI Settings | Set | Builds Select AI Azure config object incl. table name | Insert Rows into Oracle | Register with Select AI | ### Configure Azure OpenAI Settings In the "Configure Select AI Settings" node, update the selectAIConfig object |
| Register with Select AI | Oracle Database | Drops/creates Select AI profile with enforced object list | Configure Select AI Settings | Prepare Response | ### Registers with select AI Registers the data with Oracle Select AI for natural language querying powered by Azure OpenAI. |
| Prepare Response | Code | Builds success response payload | Register with Select AI | Return Success | ### Return Success Output: Returns { success, tableName, columns, rowCount, selectAIProfile }. **tableName** is passed to the chat workflow so Select AI knows which table to query. |
| Return Success | Respond to Webhook | Returns final JSON response to upload caller | Prepare Response | — | ### Expected Response { "success": true, "tableName": "UPLOAD_EXCEL_20260209123456789", "columns": ["ID","NAME","AGE","CITY","SALARY"], "rowCount": 150, "selectAIProfile": "EXCEL_AI", "message": "Excel file successfully ingested and registered with Oracle Select AI", "nextSteps": [ "Query your data using: SELECT AI EXCEL_AI your question here", "Example: SELECT AI EXCEL_AI show me the top 10 records by salary" ] } |
| Chat Input | LangChain Chat Trigger | Public chat entry point | — | Configure Select AI Profile | ## Workflow B ### How It Works User asks question in natural language ↓ Chat Input captures the question ↓ Configure Select AI Profile (sets profile name) ↓ Oracle Select AI Query - Sends question to DBMS_CLOUD_AI.GENERATE - AI converts question to SQL - Executes SQL against your data - Returns formatted results ↓ Format Chat Response (cleans up the output) ↓ Display answer to user |
| Configure Select AI Profile | Set | Sets Select AI profile name for chat queries | Chat Input | Oracle Select AI Query | ⚠️ IMPORTANT: Update 'profileName' to match your Select AI profile name. This should be the same profile created when you uploaded your Excel file. Default: EXCEL_AI |
| Oracle Select AI Query | Oracle Database | Calls `DBMS_CLOUD_AI.GENERATE(..., action='runsql')` | Configure Select AI Profile | Format Chat Response | Invokes DBMS_CLOUD_AI.GENERATE with action='runsql' to translate the chat prompt into SQL, execute it on the Select AI–registered table, and return the result set. |
| Format Chat Response | Code | Extracts/cleans response for chat UI | Oracle Select AI Query | — | Formats the AI response for display in the chat interface |
| Sticky Note | Sticky Note | Visual documentation (Workflow A flow) | — | — | ## Workflow A Flow: 1. Webhook receives Excel file ↓ 2. Validate & normalize file ↓ 3. Extract data from Excel ↓ 4. Infer schema (column names & types) ↓ 5. Create Oracle table dynamically ↓ 6. Insert data in batches (50 rows at a time) ↓ 7. Configure Select AI settings ↓ 8. Register table with Select AI ↓ 9. Return success response |
| Sticky Note1 | Sticky Note | Visual documentation (Workflow B flow) | — | — | ## Workflow B ### How It Works User asks question in natural language ↓ Chat Input captures the question ↓ Configure Select AI Profile (sets profile name) ↓ Oracle Select AI Query - Sends question to DBMS_CLOUD_AI.GENERATE - AI converts question to SQL - Executes SQL against your data - Returns formatted results ↓ Format Chat Response (cleans up the output) ↓ Display answer to user |
| Sticky Note2 | Sticky Note | Credential setup guidance | — | — | ## Configure Oracle Database Credentials Click on any Oracle Database node (e.g., "Create Oracle Table") Click Create New Credential Enter your Oracle connection details: Host, Database (service/SID), User, Password, Port 1521 Save the credential Apply the same credential to all Oracle nodes: Create Oracle Table; Insert Rows into Oracle; Register with Select AI |
| Sticky Note3 | Sticky Note | Schema-name reminder | — | — | ### Update Schema Name In the "Insert Rows into Oracle" node: Change YOUR_SCHEMA_NAME to your actual Oracle schema (e.g., ADMIN, SCOTT, etc.) |
| Sticky Note4 | Sticky Note | Azure OpenAI config reminder | — | — | ### Configure Azure OpenAI Settings In the "Configure Select AI Settings" node, update the selectAIConfig object |
| Sticky Note5 | Sticky Note | File size validation reminder | — | — | ### File Size Limits Add validation in "Validate & Normalize File" |
| Sticky Note6 | Sticky Note | Select AI registration explanation | — | — | ### Registers with select AI Registers the data with Oracle Select AI for natural language querying powered by Azure OpenAI. |
| Sticky Note7 | Sticky Note | Success output meaning | — | — | ### Return Success Output: Returns { success, tableName, columns, rowCount, selectAIProfile }. **tableName** is passed to the chat workflow so Select AI knows which table to query. |
| Sticky Note8 | Sticky Note | Batch size reminder | — | — | ### Change Batch Size Default: 50 rows per batch Adjust based on your data size and database performance |
| Sticky Note9 | Sticky Note | High-level purpose | — | — | **This creates the Oracle table and Select AI profile needed for querying** |
| Sticky Note10 | Sticky Note | Expected HTTP response example | — | — | ### Expected Response { "success": true, "tableName": "UPLOAD_EXCEL_20260209123456789", "columns": ["ID","NAME","AGE","CITY","SALARY"], "rowCount": 150, "selectAIProfile": "EXCEL_AI", "message": "Excel file successfully ingested and registered with Oracle Select AI", "nextSteps": [ "Query your data using: SELECT AI EXCEL_AI your question here", "Example: SELECT AI EXCEL_AI show me the top 10 records by salary" ] } |
| Sticky Note11 | Sticky Note | DBMS_CLOUD_AI.GENERATE behavior explanation | — | — | Invokes DBMS_CLOUD_AI.GENERATE with action='runsql' to translate the chat prompt into SQL, execute it on the Select AI–registered table, and return the result set. |

---

## 4. Reproducing the Workflow from Scratch

### A. Prerequisites (Oracle + Azure OpenAI)
1. **Oracle requirements**
   - A reachable Oracle DB endpoint (ATP/OCI/On-prem)
   - User privileges to:
     - `CREATE TABLE`
     - run `DBMS_CLOUD_AI.CREATE_PROFILE`, `DBMS_CLOUD_AI.DROP_PROFILE`, `DBMS_CLOUD_AI.GENERATE`
   - Create an Oracle credential that Select AI will use to call Azure OpenAI (typically via `DBMS_CLOUD.CREATE_CREDENTIAL`), and note its **credential name**.

2. **Azure OpenAI requirements**
   - Azure OpenAI resource name (e.g., `my-aoai-resource`)
   - A deployed model name (deployment) (e.g., `gpt-4o-mini`)
   - Network access from Oracle environment to Azure endpoints (depends on setup)

### B. Workflow A — Upload Excel → Create table → Insert → Register Select AI → Respond

1. **Create node:** *Webhook*  
   - Name: `Webhook - File Upload`  
   - Method: `POST`  
   - Path: `upload-excel`  
   - Response mode: `Last node`

2. **Create node:** *Code*  
   - Name: `Validate & Normalize File`  
   - Paste logic that:
     - verifies `item.binary` exists
     - enforces `.xlsx` / `.xls`
     - writes standardized binary to `item.binary.data`

3. **Connect:** `Webhook - File Upload` → `Validate & Normalize File`

4. **Create node:** *Extract From File*  
   - Name: `Extract from Excel`  
   - Operation: `xlsx`

5. **Connect:** `Validate & Normalize File` → `Extract from Excel`

6. **Create node:** *Code*  
   - Name: `Infer Schema & Table Name`  
   - Implement:
     - sanitize column names to Oracle-safe uppercase with underscores
     - infer types (NUMBER/DATE/VARCHAR2(4000))
     - generate table name like `UPLOAD_EXCEL_<timestamp>`
     - output `{ tableName, columns, rows }`

7. **Connect:** `Extract from Excel` → `Infer Schema & Table Name`

8. **Create node:** *Oracle Database*  
   - Name: `Create Oracle Table`  
   - Operation: `Execute`  
   - Credentials: create/select an **Oracle Database credential** in n8n  
   - Query: dynamic CREATE TABLE built from inferred columns + `UPLOADED_AT`

9. **Connect:** `Infer Schema & Table Name` → `Create Oracle Table`

10. **Create node:** *Code*  
    - Name: `Restore Data Rows`  
    - Logic: read `$("Infer Schema & Table Name").first().json.rows` and return each row as an item

11. **Connect:** `Create Oracle Table` → `Restore Data Rows`  
    - (This ensures table exists before inserts start.)

12. **Create node:** *Split In Batches*  
    - Name: `Split into Batches`  
    - Batch size: `50`

13. **Connect:** `Restore Data Rows` → `Split into Batches`

14. **Create node:** *Oracle Database*  
    - Name: `Insert Rows into Oracle`  
    - Operation: Insert (table-based insert)
    - Table name: expression from inferred `tableName`
    - Schema name: set to your schema (replace placeholder)
    - Column mapping: Auto-map input data
    - Credentials: same Oracle credential

15. **Connect:** `Split into Batches` → `Insert Rows into Oracle`

16. **Important fix (recommended): add the batch loop**
    - Add a connection: `Insert Rows into Oracle` → `Split into Batches` (to continue next batch)
    - And add a second output path from `Split into Batches` “No Items Left” to proceed to Select AI registration (implementation depends on n8n UI for SplitInBatches outputs).  
    - Without this, only the first batch may insert.

17. **Create node:** *Set*  
    - Name: `Configure Select AI Settings`  
    - Add an **object** field `selectAIConfig` containing:
      - profile_name (e.g., `EXCEL_AI`)
      - provider = `azure`
      - azure_resource_name
      - azure_deployment_name
      - credential_name (Oracle credential created in DB)
      - table_name = inferred `tableName`

18. **Connect:** `Insert Rows into Oracle` (or post-loop completion) → `Configure Select AI Settings`

19. **Create node:** *Oracle Database*  
    - Name: `Register with Select AI`  
    - Operation: `Execute`  
    - Query: PL/SQL block that:
      - detects current user
      - drops profile if exists (ignore errors)
      - creates profile with `object_list` containing the uploaded table and `enforce_object_list = TRUE`

20. **Connect:** `Configure Select AI Settings` → `Register with Select AI`

21. **Create node:** *Code*  
    - Name: `Prepare Response`  
    - Output JSON including: `success, tableName, columns, rowCount, selectAIProfile, message, nextSteps`

22. **Connect:** `Register with Select AI` → `Prepare Response`

23. **Create node:** *Respond to Webhook*  
    - Name: `Return Success`

24. **Connect:** `Prepare Response` → `Return Success`

### C. Workflow B — Chat → Select AI runsql → Format response

25. **Create node:** *Chat Trigger* (LangChain)  
    - Name: `Chat Input`  
    - Public: enabled (or restrict if needed)

26. **Create node:** *Set*  
    - Name: `Configure Select AI Profile`  
    - Set `profileName = EXCEL_AI` (must match ingestion profile name)

27. **Connect:** `Chat Input` → `Configure Select AI Profile`

28. **Create node:** *Oracle Database*  
    - Name: `Oracle Select AI Query`  
    - Operation: Execute  
    - Query: `SELECT DBMS_CLOUD_AI.GENERATE(prompt => '<escaped chatInput>', profile_name => '<profileName>', action => 'runsql') AS RESPONSE FROM dual`
    - Ensure prompt escapes `'` → `''`
    - Credentials: same Oracle credential

29. **Connect:** `Configure Select AI Profile` → `Oracle Select AI Query`

30. **Create node:** *Code*  
    - Name: `Format Chat Response`  
    - Extract `$json.RESPONSE` (fallback to first value) and return `{message: ...}`

31. **Connect:** `Oracle Select AI Query` → `Format Chat Response`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Oracle credentials must be configured once and reused across all Oracle nodes | See sticky note “Configure Oracle Database Credentials” |
| Update `YOUR_SCHEMA_NAME` in the insert node to your real Oracle schema | See sticky note “Update Schema Name” |
| Replace Azure placeholders in Select AI config (`azure_resource_name`, `azure_deployment_name`, `credential_name`) | See sticky note “Configure Azure OpenAI Settings” |
| Consider adding file size validation in the validation Code node | See sticky note “File Size Limits” |
| Workflow B relies on the Select AI profile name being consistent (default `EXCEL_AI`) | See “Configure Select AI Profile” node |
| Expected upload response structure includes `tableName` and `selectAIProfile` plus guidance for next steps | See sticky note “Expected Response” |
| Select AI call uses `DBMS_CLOUD_AI.GENERATE` with `action='runsql'` | See sticky note “Invokes DBMS_CLOUD_AI.GENERATE …” |