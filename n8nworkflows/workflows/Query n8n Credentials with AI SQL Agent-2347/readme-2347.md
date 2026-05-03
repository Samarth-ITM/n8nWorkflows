Query n8n Credentials with AI SQL Agent

https://n8nworkflows.xyz/workflows/query-n8n-credentials-with-ai-sql-agent-2347


# Query n8n Credentials with AI SQL Agent

### 1. Workflow Overview

This workflow enables querying and searching over n8n instance credentials using an AI-driven SQL agent. It is designed for users who want to explore which workflows use specific credentials or apps without exposing any decrypted credential data.

The workflow is logically divided into two main blocks:

- **1.1 Fetching and Storing Credentials Data:**  
  Queries the n8n API to fetch all workflows, extracts credentials references from each workflow’s nodes, and stores this metadata in a local SQLite database. This block ensures sensitive credential values remain secure by only storing metadata.

- **1.2 AI-Driven Query Interface:**  
  Sets up an AI agent with a custom SQL tool that accesses the SQLite database. The agent accepts natural language queries from users, translates them into SQL queries to interrogate workflow credential data, and returns results in conversational form.

---

### 2. Block-by-Block Analysis

#### 2.1 Fetching and Storing Credentials Data

**Overview:**  
This block retrieves workflows from the n8n instance via its API, extracts all associated credential references, and stores this data in a SQLite database. The database is created in-memory and will reset if the instance restarts.

**Nodes Involved:**  
- When clicking "Test workflow" (Manual Trigger)  
- n8n (n8n API node)  
- Map Workflows & Credentials (Set node)  
- Save to Database (Code node, Python)  
- Sticky Notes (informational only)  
- Sticky Note (API Key requirement note)

**Node Details:**

- **When clicking "Test workflow"**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually  
  - *Config:* Default trigger, no parameters  
  - *Inputs:* None  
  - *Outputs:* Triggers the n8n API node  
  - *Failures:* None expected

- **n8n**  
  - *Type:* n8n API node  
  - *Role:* Fetches workflows data from the n8n instance using built-in API  
  - *Config:* No filters, defaults to fetching all workflows available to API key  
  - *Credentials:* Requires n8n API credential with appropriate scopes  
  - *Inputs:* Trigger from Manual Trigger  
  - *Outputs:* Workflow JSON data to "Map Workflows & Credentials" node  
  - *Failures:* API key invalid or insufficient permissions; network issues

- **Map Workflows & Credentials**  
  - *Type:* Set node  
  - *Role:* Extracts workflow id, name, and compiles a flattened list of all credentials referenced in nodes  
  - *Config:* Dynamically maps:  
    - `workflow_id` = workflow’s `id`  
    - `workflow_name` = workflow’s `name`  
    - `credentials` = array combining all credential objects found on workflow nodes (transformed into array of `{type, ...credential}` objects)  
  - *Inputs:* Workflow JSON objects from n8n node  
  - *Outputs:* Structured data for DB insertion  
  - *Failures:* Expression failures if input JSON structure unexpected

- **Save to Database**  
  - *Type:* Code node (Python)  
  - *Role:* Creates SQLite database (in-memory), creates table if not exists, inserts or replaces workflow credential records  
  - *Config:*  
    - Table `n8n_workflow_credentials` with columns: `workflow_id` (PK), `workflow_name`, `credentials` (JSON string)  
    - Inserts each item from input  
  - *Inputs:* Output from Set node  
  - *Outputs:* Number of affected rows (for logging/debugging)  
  - *Failures:* SQLite errors (e.g., write issues), data serialization error

- **Sticky Notes** (Several)  
  - Provide contextual information and warnings such as API key requirement, data security assurances, and usage instructions.

---

#### 2.2 AI-Driven Query Interface

**Overview:**  
This block sets up a natural language interface powered by an AI agent that can query the SQLite database created previously. Users can ask questions about workflows and their credentials, which the agent translates to SQL and executes, returning meaningful answers.

**Nodes Involved:**  
- Chat Trigger (LangChain Chat Trigger node)  
- OpenAI Chat Model (LangChain OpenAI Chat node)  
- Window Buffer Memory (LangChain Memory node)  
- Query Workflow Credentials Database (LangChain Tool Code node)  
- Workflow Credentials Helper Agent (LangChain Agent node)  
- Sticky Notes (informational)

**Node Details:**

- **Chat Trigger**  
  - *Type:* LangChain Chat Trigger node  
  - *Role:* Webhook endpoint that receives incoming chat messages (user queries)  
  - *Config:* Default webhook ID assigned  
  - *Inputs:* External webhook call  
  - *Outputs:* User messages forwarded to AI agent  
  - *Failures:* Webhook connectivity or unauthorized access

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Natural language understanding and generation using OpenAI API  
  - *Config:* Uses OpenAI API credentials stored in n8n  
  - *Credentials:* OpenAI API key required  
  - *Inputs:* Messages from Chat Trigger  
  - *Outputs:* AI-generated replies forwarded to Agent  
  - *Failures:* API quota limits, invalid credentials, network errors

- **Window Buffer Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Maintains conversational context over a sliding window of recent messages  
  - *Config:* Default parameters  
  - *Inputs:* Chat messages and AI responses  
  - *Outputs:* Context-aware conversation history to Agent  
  - *Failures:* Memory overflow or corruption unlikely

- **Query Workflow Credentials Database**  
  - *Type:* LangChain Tool Code (Python)  
  - *Role:* Executes SQL queries on the SQLite database created earlier  
  - *Config:*  
    - Accepts SQL `SELECT` queries as input  
    - Returns query results as JSON string  
    - Database file: `n8n_workflow_credentials.db`  
    - Available table schema clearly documented in node description  
  - *Inputs:* SQL query from Agent  
  - *Outputs:* Query results to Agent  
  - *Failures:* SQL syntax errors, database file missing (if instance restarted), query timeout

- **Workflow Credentials Helper Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Orchestrates interaction between user queries, OpenAI chat model, memory, and SQL query tool  
  - *Config:*  
    - System prompt instructs agent to interpret user queries as requests about n8n workflow credentials  
    - Provides URL schema for linking workflows dynamically if requested  
  - *Inputs:*  
    - AI language model output from OpenAI Chat Model  
    - Memory context from Window Buffer Memory  
    - SQL tool from Query Workflow Credentials Database  
  - *Outputs:* Final AI-generated answer sent back to Chat Trigger response  
  - *Failures:* Misinterpretation of user intent, malformed SQL queries, database unavailability

- **Sticky Notes**  
  - Explain usage, encourage trying queries, and describe the AI interface’s purpose.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                                   | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                                  |
|-------------------------------|--------------------------------|-------------------------------------------------|--------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow"  | Manual Trigger                 | Starts workflow manually                         | None                           | n8n                              |                                                                                                              |
| n8n                           | n8n API node                  | Fetches all workflows from n8n API              | When clicking "Test workflow"  | Map Workflows & Credentials       |                                                                                                              |
| Map Workflows & Credentials    | Set node                     | Extracts workflow credentials metadata           | n8n                            | Save to Database                  |                                                                                                              |
| Save to Database               | Code node (Python)            | Inserts/updates SQLite DB with workflow data    | Map Workflows & Credentials    | None                             |                                                                                                              |
| Sticky Note                   | Sticky Note                   | Notes API key requirement                        | None                           | None                             | 🚨Required: You'll need an n8n API key. Note: available workflows will be scoped to your key.                |
| Sticky Note1                  | Sticky Note                   | Explains Step 1: storing credentials             | None                           | None                             | ## Step 1. Store Workflows Credential Mappings to Database. Database will be wiped on instance restart.     |
| Sticky Note4                  | Sticky Note                   | Usage instructions and examples                  | None                           | None                             | Try It Out! This workflow lets you query workflow credentials using an AI SQL agent with example queries.    |
| Chat Trigger                 | LangChain Chat Trigger         | Receives user chat queries                        | None                           | Workflow Credentials Helper Agent |                                                                                                              |
| OpenAI Chat Model             | LangChain OpenAI Chat Model    | Processes queries with OpenAI                     | Chat Trigger                  | Workflow Credentials Helper Agent |                                                                                                              |
| Window Buffer Memory          | LangChain Memory Buffer Window | Maintains conversational context                  | Chat Trigger/OpenAI Chat Model | Workflow Credentials Helper Agent |                                                                                                              |
| Query Workflow Credentials Database | LangChain Tool Code (Python) | Executes SQL queries on credentials DB            | Workflow Credentials Helper Agent | Workflow Credentials Helper Agent | Database contains workflow_id, workflow_name, credentials (JSON string). Prefer SQL LIKE for app names.      |
| Workflow Credentials Helper Agent | LangChain Agent            | AI agent coordinating query to DB and chat model | Chat Trigger, OpenAI Chat Model, Window Buffer Memory, Tool Code | Chat Trigger                  | System prompt guides agent to find info on n8n workflow credentials; dynamic workflow URLs supported.       |
| Sticky Note2                 | Sticky Note                   | Explains Step 2: AI agent search interface       | None                           | None                             | Step 2. Use Agent as Search Interface to query workflows by credentials using natural language.              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node and name it `When clicking "Test workflow"`.

2. **Configure n8n API Node**  
   - Add an **n8n API** node named `n8n`.  
   - Use credentials with an API key scoped to read workflows.  
   - Leave filters empty to fetch all workflows.  
   - Connect output from Manual Trigger → n8n API node.

3. **Add Set Node to Map Credentials**  
   - Add a **Set** node named `Map Workflows & Credentials`.  
   - Set fields:  
     - `workflow_id`: `{{$json["id"]}}`  
     - `workflow_name`: `{{$json["name"]}}`  
     - `credentials`: Use expression to extract and flatten credentials from all nodes:  
       `{{$json.nodes.map(node => node.credentials).compact().reduce((acc,cred) => { const keys = Object.keys(cred); const items = keys.map(key => ({ type: key,  ...cred[key] })); acc.push(...items); return acc; }, [])}}`  
   - Connect output from n8n API node → Set node.

4. **Add Code Node for SQLite DB**  
   - Add **Code** node named `Save to Database` with language set to Python.  
   - Paste the following code (adapted as needed):  
     ```python
     import json
     import sqlite3
     con = sqlite3.connect("n8n_workflow_credentials.db")

     cur = con.cursor()
     cur.execute("CREATE TABLE IF NOT EXISTS n8n_workflow_credentials (workflow_id TEXT PRIMARY KEY, workflow_name TEXT, credentials TEXT);")

     for item in _input.all():
       cur.execute('INSERT OR REPLACE INTO n8n_workflow_credentials VALUES(?,?,?)', (
         item.json.workflow_id,
         item.json.workflow_name,
         json.dumps(item.json.credentials.to_py())
       ))

     con.commit()
     con.close()

     return [{ "affected_rows": len(_input.all()) }]
     ```  
   - Connect output from Set node → Code node.

5. **Add LangChain Chat Trigger**  
   - Add **Chat Trigger** node (LangChain).  
   - This will serve as the webhook receiving user queries.

6. **Add OpenAI Chat Model Node**  
   - Add **OpenAI Chat Model** node (LangChain).  
   - Configure with OpenAI API credentials.  
   - Connect Chat Trigger → OpenAI Chat Model.

7. **Add Window Buffer Memory Node**  
   - Add **Window Buffer Memory** node (LangChain).  
   - Connect Chat Trigger → Window Buffer Memory.  
   - Connect OpenAI Chat Model → Window Buffer Memory if required (depending on n8n version, usually memory connects as input to Agent).

8. **Add Tool Code Node for SQL Queries**  
   - Add **Tool Code** node named `Query Workflow Credentials Database`.  
   - Language: Python.  
   - Code:  
     ```python
     import json
     import sqlite3
     con = sqlite3.connect("n8n_workflow_credentials.db")

     cur = con.cursor()
     res = cur.execute(query);

     output = json.dumps(res.fetchall())

     con.close()
     return output;
     ```  
   - This node accepts a SQL SELECT query and returns JSON results.

9. **Add LangChain Agent Node**  
   - Add **Agent** node named `Workflow Credentials Helper Agent`.  
   - Configure system prompt to instruct it to interpret user queries as about n8n workflow credentials, referencing workflows by name and credential types.  
   - Provide URL schema template for workflow links:  
     `{{ window.location.protocol + '//' + window.location.host }}/workflow/$workflow_id`  
   - Connect inputs:  
     - `ai_languageModel` → OpenAI Chat Model  
     - `ai_memory` → Window Buffer Memory  
     - `ai_tool` → Query Workflow Credentials Database  
     - Input comes from Chat Trigger  
   - Output returns to Chat Trigger to reply to user.

10. **Add Sticky Notes** (Optional)  
    - Add sticky notes for instructions, usage examples, and API key requirement warnings as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Your credentials remain secure: this workflow does not decrypt or expose any actual credential values.                | Security best practice for workflows handling credentials                                          |
| SQLite database is created in-memory and will be lost upon n8n instance restart.                                       | Important for understanding data persistence limitations                                         |
| Example queries: “Which workflows are using Slack and Google Calendar?”, “Which workflows have AI in their name?”     | Demonstrates practical use cases for the AI SQL agent interface                                   |
| Requires an n8n API key with scope to read workflows.                                                                 | API key management                                                                                 |
| AI agent uses SQL LIKE operator to match app names as credential app names may be obscured in JSON.                    | Query optimization and correctness                                                               |
| LangChain nodes require n8n version supporting LangChain integration (v0.201.0+ recommended).                          | Version compatibility                                                                              |
| Workflow URL schema uses dynamic replacement for workflow IDs to create clickable links in responses.                 | Useful for navigation from AI query results                                                      |

---

This structured documentation enables full understanding, modification, and rebuilding of the "Query n8n Credentials with AI SQL Agent" workflow without referring back to the raw JSON.