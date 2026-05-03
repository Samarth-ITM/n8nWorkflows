Synchronize MySQL database schemas to Pinecone with OpenAI embeddings

https://n8nworkflows.xyz/workflows/synchronize-mysql-database-schemas-to-pinecone-with-openai-embeddings-11971


# Synchronize MySQL database schemas to Pinecone with OpenAI embeddings

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow synchronizes **MySQL table schema definitions** into **Pinecone** as **OpenAI embeddings**, storing **one vector per table** so the full schema context can be retrieved later (RAG / schema reasoning). It is designed to be **idempotent**: it avoids duplicates and only re-indexes when the schema changes.

**Target use cases**
- Keeping a vector index of database schemas for AI-assisted SQL generation, data discovery, and schema Q&A.
- Detecting schema drift and refreshing embeddings automatically.
- Maintaining a metadata log for what has been embedded and when it needs re-embedding.

### Logical Blocks
**1.1 Setup & Schema Discovery**  
Manual trigger → load configuration → list all tables → iterate table-by-table → fetch each table’s `CREATE TABLE` statement.

**1.2 Schema Normalization & Fingerprinting**  
Generate a deterministic `vector_id` and compute a hash of the schema text used to detect changes.

**1.3 Vector Existence & Change Detection**  
Look up metadata for the vector_id in an n8n Data Table, decide whether to insert new or delete+reinsert.

**1.4 Cleanup, Embedding & Indexing (Pinecone)**  
If changed: delete old vector + delete metadata row. Then build a document, generate embeddings, insert into Pinecone, and upsert metadata.

---

## 2. Block-by-Block Analysis

### 2.1 Setup & Schema Discovery
**Overview:** Initializes runtime configuration, discovers all base tables in the current MySQL database, and loops over them to fetch each table’s schema definition.

**Nodes involved:**
- Sync DB Schema to Vector Store
- Load Global Configuration
- Fetch All Database Tables
- Loop Over executable queries
- Set Table Schema Context
- Fetch Table Schema Definition

#### Node: **Sync DB Schema to Vector Store**
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point.
- **Configuration:** No parameters; starts execution manually.
- **Outputs:** To **Load Global Configuration**.
- **Failure modes:** None (unless workflow disabled).

#### Node: **Load Global Configuration**
- **Type / role:** Set node (`n8n-nodes-base.set`) — defines global constants used across nodes.
- **Configuration choices (interpreted):**
  - Outputs a JSON object with keys:
    - `vector_namespace`, `embedding_model`, `chunk_size`, `chunk_overlap`
    - `pinecone_index`, `mysql_database_name`, `vector_index_host`
    - `pinecone_apikey`, `dataTable_Id`
  - Values are empty strings in the template and must be filled.
- **Key expressions/variables:** Referenced later via `$('Load Global Configuration').item.json.<key>`.
- **Outputs:** To **Fetch All Database Tables**.
- **Failure modes / edge cases:**
  - Missing/empty values lead to auth errors (Pinecone), invalid model selection (OpenAI), or Data Table not found.

#### Node: **Fetch All Database Tables**
- **Type / role:** MySQL node (`n8n-nodes-base.mySql`) — enumerates tables and builds per-table `SHOW CREATE TABLE` queries.
- **Configuration choices:**
  - Operation: *Execute Query*
  - SQL:
    - Reads from `information_schema.tables`
    - Filters to current `DATABASE()` and `BASE TABLE`
    - Returns:
      - `queries`: `SHOW CREATE TABLE \`TABLE_NAME\`;`
      - `tbl`: table name
      - `db`: current database
- **Credentials:** `mySql Credential`
- **Input/Output:** Input from config; output to **Loop Over executable queries**.
- **Failure modes:**
  - MySQL auth/host errors, insufficient privileges on `information_schema`, or non-default database selection issues.

#### Node: **Loop Over executable queries**
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates tables one at a time.
- **Configuration:** `reset: false` (continues batching until done).
- **Connections:**
  - Receives list from **Fetch All Database Tables**.
  - Main output (index 1) goes to **Set Table Schema Context**.
  - Loop-back connections later return here to continue.
- **Failure modes / edge cases:**
  - If upstream returns 0 items, loop will not process any table.
  - If downstream nodes error mid-loop, run stops and remaining tables won’t sync.

#### Node: **Set Table Schema Context**
- **Type / role:** Set node — defines deterministic vector identity and basic metadata.
- **Key configuration:**
  - Produces:
    - `vector_id`: `mysql_schema::<db>::<tbl>` using the current loop item
    - `table_name`: from `tbl`
    - `source_type`: `"mysql_schema"`
  - Uses expressions referencing the loop node:
    - `$node['Loop Over executable queries'].json.db`
    - `$node['Loop Over executable queries'].json.tbl`
- **Output:** To **Fetch Table Schema Definition**.
- **Failure modes:**
  - If the loop item doesn’t have `db`/`tbl` fields, expressions fail.

#### Node: **Fetch Table Schema Definition**
- **Type / role:** MySQL node — executes the per-table `SHOW CREATE TABLE ...;`.
- **Configuration:**
  - Query expression: `{{ $node['Loop Over executable queries'].json.queries }}`
  - Operation: *Execute Query*
- **Credentials:** `mySql Credential`
- **Output:** To **Generate Schema Hash**.
- **Failure modes / edge cases:**
  - Table names requiring special quoting are already backticked; still, permission issues may occur.
  - Output shape depends on MySQL driver; the `CREATE TABLE` text is expected in field **`Create Table`**.

---

### 2.2 Schema Normalization & Fingerprinting
**Overview:** Computes a hash of the schema text to detect changes and carries forward the `vector_id`/metadata for later steps.

**Nodes involved:**
- Generate Schema Hash

#### Node: **Generate Schema Hash**
- **Type / role:** Code node (`n8n-nodes-base.code`) — hashes schema text and enriches the item.
- **Configuration (interpreted):**
  - Reads schema text: `$input.first().json['Create Table']`
  - Uses a DJB2-like rolling hash, returns:
    - All original JSON fields
    - `vector_id` from **Set Table Schema Context**
    - `source_type` from **Set Table Schema Context**
    - `schema_hash` as string
- **Input/Output:**
  - Input from **Fetch Table Schema Definition**
  - Output to **Check Existing Vector Metadata**
- **Failure modes / edge cases:**
  - If `'Create Table'` field name differs (driver/localization), hashing will fail or hash empty/undefined.
  - Large schema text is fine for hashing but later embedding may hit token limits depending on model.
  - Hash collisions are possible (though unlikely); could skip a needed re-index.

---

### 2.3 Vector Existence & Change Detection
**Overview:** Checks if the table’s vector was previously stored (via metadata table). If present, compares stored hash vs current hash to decide whether to re-index.

**Nodes involved:**
- Check Existing Vector Metadata
- Vector Exists?
- Schema Changed?

#### Node: **Check Existing Vector Metadata**
- **Type / role:** Data Table (`n8n-nodes-base.dataTable`) — lookup row(s) for this `vector_id`.
- **Configuration:**
  - Operation: **get**, `returnAll: true`
  - Filter: `vector_id == {{ $node['Generate Schema Hash'].json.vector_id }}`
  - DataTableId set by expression: `$('Load Global Configuration').item.json.dataTable_Id`
  - `alwaysOutputData: true` (important for IF evaluation)
- **Output:** To **Vector Exists?**
- **Failure modes:**
  - Wrong `dataTable_Id` → “not found” / permission error.
  - If multiple rows match unexpectedly, downstream assumptions may break (should be 0 or 1).

#### Node: **Vector Exists?**
- **Type / role:** IF node — branches based on whether metadata lookup returned anything.
- **Configuration (interpreted):**
  - Condition checks whether `$node["Check Existing Vector Metadata"].json` is **not empty**.
  - **True branch (vector exists):** goes to **Schema Changed?**
  - **False branch (vector does not exist):** goes directly to **Insert Schema Vector to Pinecone**
- **Output connections:**
  - Output 0 → Insert Schema Vector to Pinecone
  - Output 1 → Schema Changed?
- **Failure modes / edge cases:**
  - Data Table node output structure nuances: if the “get” returns an array vs object, an “empty” check may behave unexpectedly.
  - If `alwaysOutputData` + empty payload still produces an object wrapper, IF logic may misroute.

#### Node: **Schema Changed?**
- **Type / role:** IF node — compares stored hash vs current hash.
- **Configuration:**
  - `leftValue`: `{{ $node['Vector Exists?'].json.schema_hash }}`
  - `rightValue`: `{{ $node['Generate Schema Hash'].json.schema_hash }}`
  - Operator: string `notEquals`
  - **True branch:** delete old vector and metadata
  - **False branch:** skip re-index and continue loop
- **Connections:**
  - True → **Delete Existing Vector (Pinecone)**
  - False → **Loop Over executable queries** (continue)
- **Failure modes / edge cases:**
  - If the “Vector Exists?” branch doesn’t actually output a `schema_hash` field (because it outputs the current item, not the metadata row), the comparison can be invalid.
  - Type validation is `strict`; null/undefined may cause unexpected results.

---

### 2.4 Vector Cleanup, Embedding & Indexing (Pinecone)
**Overview:** When needed, removes the existing vector from Pinecone and deletes the metadata record, then recreates the vector by embedding the schema text and inserting it into Pinecone. Finally, upserts metadata for future runs.

**Nodes involved:**
- Delete Existing Vector (Pinecone)
- Delete Vector Metadata Record
- Split Schema Text (No Chunking)
- Prepare Schema Document
- Generate Schema Embeddings
- Insert Schema Vector to Pinecone
- Upsert Vector Metadata Record

#### Node: **Delete Existing Vector (Pinecone)**
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Pinecone REST delete endpoint.
- **Configuration:**
  - Method: POST
  - URL: `{{ vector_index_host }}/vectors/delete`
  - Header: `Api-Key: {{ pinecone_apikey }}`
  - JSON body:
    - `filter.vector_id.$eq = <current vector_id>`
    - `namespace = <vector_namespace>`
  - `allowUnauthorizedCerts: true` (permissive TLS; generally not recommended in production)
- **Input/Output:** From **Schema Changed?** true → output to **Delete Vector Metadata Record**
- **Failure modes:**
  - Incorrect host (must be the Pinecone index host URL), invalid API key, wrong namespace.
  - Pinecone API schema differences: some setups use `ids` instead of filter; this workflow assumes filter delete is supported.
  - Network timeouts / 4xx/5xx responses.

#### Node: **Delete Vector Metadata Record**
- **Type / role:** Data Table — deletes the metadata row(s) for this vector_id.
- **Configuration:**
  - Operation: `deleteRows`
  - Filter: `vector_id == {{ $('Generate Schema Hash').item.json.vector_id }}`
  - DataTableId is **hard-coded** to a list value named `rag_embedding_log` (not using global config here).
- **Output:** To **Insert Schema Vector to Pinecone**
- **Failure modes / edge cases:**
  - If you change DataTable_Id in config but not here, deletions and upserts may target different tables.
  - Deleting 0 rows is possible if metadata drifted; that’s usually safe but indicates inconsistency.

#### Node: **Split Schema Text (No Chunking)**
- **Type / role:** LangChain Text Splitter (`@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`)
- **Configuration:**
  - `chunkSize` and `chunkOverlap` come from global config.
  - Despite the name “No Chunking”, this node can chunk if `chunk_size` is set smaller than the schema text length.
  - `splitCode: markdown`
- **Connections:** Its `ai_textSplitter` output feeds **Prepare Schema Document**.
- **Failure modes:**
  - Non-numeric `chunk_size`/`chunk_overlap` strings may error or coerce unexpectedly.
  - If chunking occurs, you may create multiple documents/vectors (depending on downstream behavior).

#### Node: **Prepare Schema Document**
- **Type / role:** LangChain Document Loader (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`) — builds the document text + metadata.
- **Configuration:**
  - Document content: `$('Generate Schema Hash').item.json['Create Table']`
  - Metadata fields:
    - `table_name`: `{{ $('Generate Schema Hash').item.json.Table }}`
    - `vector_id`, `source_type`, `schema_hash`
- **Connections:** Sends `ai_document` to **Insert Schema Vector to Pinecone**.
- **Failure modes / edge cases:**
  - **Potential bug:** uses `json.Table` (capital T) but earlier nodes provide `tbl` and/or `table_name`. In MySQL `SHOW CREATE TABLE`, the table name field might be `Table` (capitalized) depending on driver—verify actual output. If absent, metadata and later upsert may store blank table name.

#### Node: **Generate Schema Embeddings**
- **Type / role:** OpenAI Embeddings (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`)
- **Configuration:**
  - Model from config: `embedding_model`
- **Credentials:** `openAiApi Credential`
- **Connections:** `ai_embedding` to **Insert Schema Vector to Pinecone**
- **Failure modes:**
  - Invalid model name, missing OpenAI credentials, rate limits, timeouts.
  - Payload too large for embedding model context window (large schemas).

#### Node: **Insert Schema Vector to Pinecone**
- **Type / role:** Pinecone Vector Store (`@n8n/n8n-nodes-langchain.vectorStorePinecone`) — inserts vectors.
- **Configuration:**
  - Mode: `insert`
  - Pinecone index: selected list value `dbrag`
  - Namespace: `{{ $node['Load Global Configuration'].json.vector_namespace }}`
  - Receives both:
    - `ai_document` from **Prepare Schema Document**
    - `ai_embedding` from **Generate Schema Embeddings**
- **Credentials:** `pineconeApi Credential`
- **Output:** To **Upsert Vector Metadata Record**
- **Failure modes / edge cases:**
  - Pinecone index mismatch (node uses a picked index “dbrag” rather than config `pinecone_index`).
  - Namespace mismatch with the deletion HTTP call (must be identical).
  - Dimension mismatch between embeddings and Pinecone index dimension.

#### Node: **Upsert Vector Metadata Record**
- **Type / role:** Data Table — persists (vector_id, table_name, schema_hash).
- **Configuration:**
  - Operation: `upsert`
  - Filter: `vector_id == {{ Generate Schema Hash.vector_id }}`
  - Writes columns:
    - `vector_id`: ok
    - `table_name`: `{{ $node['Generate Schema Hash'].json.Table }}`
    - `schema_hash`: ok
  - DataTableId: hard-coded list value `rag_embedding_log` (same as deletion node).
- **Connections:** Output returns to **Loop Over executable queries** to continue.
- **Failure modes / edge cases:**
  - Same **potential bug**: `json.Table` may not exist → table_name stored blank.
  - Schema of Data Table must match column IDs (`vector_id`, `table_name`, `schema_hash`).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation | — | — | ## Workflow Overview - MySQL Schema Embeddings in Pinecone (full overview text) |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## How to Use This Workflow<br>@[youtube](elykvVRjVL0) |
| Sync DB Schema to Vector Store | Manual Trigger | Start workflow manually | — | Load Global Configuration | ## Setup Workflow & Schema Discovery<br>Initialize configuration, discover database tables, and generate schema fingerprints |
| Load Global Configuration | Set | Define global config variables | Sync DB Schema to Vector Store | Fetch All Database Tables | ## Setup Workflow & Schema Discovery<br>Initialize configuration, discover database tables, and generate schema fingerprints |
| Fetch All Database Tables | MySQL | List tables and generate SHOW CREATE queries | Load Global Configuration | Loop Over executable queries | ## Setup Workflow & Schema Discovery<br>Initialize configuration, discover database tables, and generate schema fingerprints |
| Loop Over executable queries | Split In Batches | Iterate tables one-by-one | Fetch All Database Tables; Schema Changed? (false); Upsert Vector Metadata Record | Set Table Schema Context | ## Setup Workflow & Schema Discovery<br>Initialize configuration, discover database tables, and generate schema fingerprints |
| Set Table Schema Context | Set | Build deterministic vector_id and metadata | Loop Over executable queries | Fetch Table Schema Definition | ## Setup Workflow & Schema Discovery<br>Initialize configuration, discover database tables, and generate schema fingerprints |
| Fetch Table Schema Definition | MySQL | Run SHOW CREATE TABLE for current table | Set Table Schema Context | Generate Schema Hash | ## Setup Workflow & Schema Discovery<br>Initialize configuration, discover database tables, and generate schema fingerprints |
| Generate Schema Hash | Code | Hash schema text to detect changes | Fetch Table Schema Definition | Check Existing Vector Metadata | ## Setup Workflow & Schema Discovery<br>Initialize configuration, discover database tables, and generate schema fingerprints |
| Check Existing Vector Metadata | Data Table | Lookup metadata for vector_id | Generate Schema Hash | Vector Exists? | ## Vector Existence & Change Detection<br>Duplicate prevention and schema drift logic |
| Vector Exists? | IF | Branch: insert new vs compare hash | Check Existing Vector Metadata | Insert Schema Vector to Pinecone; Schema Changed? | ## Vector Existence & Change Detection<br>Duplicate prevention and schema drift logic |
| Schema Changed? | IF | Branch: delete+reindex vs skip | Vector Exists? | Delete Existing Vector (Pinecone); Loop Over executable queries | ## Vector Existence & Change Detection<br>Duplicate prevention and schema drift logic |
| Delete Existing Vector (Pinecone) | HTTP Request | Delete prior vector from Pinecone | Schema Changed? (true) | Delete Vector Metadata Record | ## Vector Cleanup & Indexing<br>Safe deletion and re-indexing pipeline |
| Delete Vector Metadata Record | Data Table | Delete old metadata row(s) | Delete Existing Vector (Pinecone) | Insert Schema Vector to Pinecone | ## Vector Cleanup & Indexing<br>Safe deletion and re-indexing pipeline |
| Split Schema Text (No Chunking) | Recursive Character Text Splitter | Optional chunking before doc creation | — | Prepare Schema Document | ## Vector Cleanup & Indexing<br>Safe deletion and re-indexing pipeline |
| Prepare Schema Document | Document Loader | Create document with metadata | Split Schema Text (No Chunking) | Insert Schema Vector to Pinecone (ai_document) | ## Vector Cleanup & Indexing<br>Safe deletion and re-indexing pipeline |
| Generate Schema Embeddings | OpenAI Embeddings | Generate embeddings for schema text | — | Insert Schema Vector to Pinecone (ai_embedding) | ## Vector Cleanup & Indexing<br>Safe deletion and re-indexing pipeline |
| Insert Schema Vector to Pinecone | Pinecone Vector Store | Insert document vectors into Pinecone | Vector Exists? (false); Delete Vector Metadata Record; Prepare Schema Document; Generate Schema Embeddings | Upsert Vector Metadata Record | ## Vector Cleanup & Indexing<br>Safe deletion and re-indexing pipeline |
| Upsert Vector Metadata Record | Data Table | Record vector_id + schema_hash for idempotency | Insert Schema Vector to Pinecone | Loop Over executable queries | ## Vector Cleanup & Indexing<br>Safe deletion and re-indexing pipeline |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Template - MySQL Schema Embeddings in Pinecone** (or your choice).
- Ensure execution order is default (this workflow uses `settings.executionOrder = v1`).

2) **Add trigger**
- Add **Manual Trigger** node named **Sync DB Schema to Vector Store**.

3) **Add configuration Set node**
- Add **Set** node named **Load Global Configuration**.
- Mode: **Raw**
- JSON output with keys (fill values as needed):
  - `vector_namespace`: e.g. `"mysql-schemas"`
  - `embedding_model`: e.g. `"text-embedding-3-small"` (must match your OpenAI account)
  - `chunk_size`: set large if you truly want one chunk (e.g. `100000`), or set a smaller chunk size if you accept multiple vectors
  - `chunk_overlap`: e.g. `0`
  - `pinecone_index`: (optional in this template; Pinecone node uses a picked index)
  - `mysql_database_name`: informational (this template uses `DATABASE()` in SQL)
  - `vector_index_host`: your Pinecone index host URL (e.g. `https://<index>-<project>.svc.<region>.pinecone.io`)
  - `pinecone_apikey`: your Pinecone API key
  - `dataTable_Id`: the n8n Data Table ID for the metadata log (if you choose to parameterize it everywhere)

4) **Connect** Manual Trigger → Load Global Configuration.

5) **Add MySQL “list tables” node**
- Add **MySQL** node named **Fetch All Database Tables**.
- Operation: **Execute Query**
- SQL:
  ```sql
  SELECT CONCAT('SHOW CREATE TABLE `', TABLE_NAME, '`;') AS queries,
         TABLE_NAME as tbl,
         DATABASE() as db
  FROM information_schema.tables
  WHERE table_schema = DATABASE()
    AND TABLE_TYPE = 'BASE TABLE';
  ```
- Configure **MySQL credentials** (host/user/password or connection string).
- Connect Load Global Configuration → Fetch All Database Tables.

6) **Add loop node**
- Add **Split In Batches** named **Loop Over executable queries**.
- Keep `reset = false`.
- Connect Fetch All Database Tables → Loop Over executable queries.

7) **Add Set node for vector context**
- Add **Set** named **Set Table Schema Context**, Mode **Raw**.
- JSON output:
  - `vector_id`: `mysql_schema::{{ $node['Loop Over executable queries'].json.db }}::{{ $node['Loop Over executable queries'].json.tbl }}`
  - `table_name`: `{{ $node['Loop Over executable queries'].json.tbl }}`
  - `source_type`: `mysql_schema`
- Connect Loop Over executable queries (main output that yields items) → Set Table Schema Context.

8) **Add MySQL “SHOW CREATE TABLE” node**
- Add **MySQL** node named **Fetch Table Schema Definition**.
- Operation: Execute Query
- Query: `{{ $node['Loop Over executable queries'].json.queries }}`
- Connect Set Table Schema Context → Fetch Table Schema Definition.

9) **Add Code node for hashing**
- Add **Code** node named **Generate Schema Hash**.
- Use JS hashing of `$input.first().json['Create Table']` and return `schema_hash`, plus `vector_id` and `source_type` from the Set node.
- Connect Fetch Table Schema Definition → Generate Schema Hash.

10) **Create a Data Table for metadata**
- In n8n, create a Data Table (example name: `rag_embedding_log`) with columns:
  - `vector_id` (string)
  - `table_name` (string)
  - `schema_hash` (string)
- Note its ID for configuration.

11) **Add Data Table lookup node**
- Add **Data Table** node named **Check Existing Vector Metadata**.
- Operation: **get**
- Return all: enabled
- Filter: `vector_id` equals `{{ $node['Generate Schema Hash'].json.vector_id }}`
- DataTableId: use expression to `dataTable_Id` (recommended) or pick the table directly.
- Enable **Always Output Data**.
- Connect Generate Schema Hash → Check Existing Vector Metadata.

12) **Add IF node “Vector Exists?”**
- Add **IF** named **Vector Exists?**
- Condition: check whether lookup output is empty/non-empty (based on your actual Data Table “get” output shape).
- Connect Check Existing Vector Metadata → Vector Exists?
- False (doesn’t exist) → will go to Pinecone insert.
- True (exists) → will go to hash comparison.

13) **Add IF node “Schema Changed?”**
- Add **IF** named **Schema Changed?**
- Condition: stored `schema_hash` != current `schema_hash`.
  - Stored hash must come from the metadata row; ensure you reference the correct field from **Check Existing Vector Metadata** output.
- Connect Vector Exists? (true) → Schema Changed?
- Connect Schema Changed? (false) → Loop Over executable queries (to continue).

14) **Add Pinecone deletion HTTP request (optional but used here)**
- Add **HTTP Request** named **Delete Existing Vector (Pinecone)**.
- Method: POST
- URL: `{{ $('Load Global Configuration').item.json.vector_index_host }}/vectors/delete`
- Headers: `Api-Key = {{ $('Load Global Configuration').item.json.pinecone_apikey }}`
- JSON body:
  - `namespace = {{ vector_namespace }}`
  - filter on `vector_id == current vector_id`
- Connect Schema Changed? (true) → Delete Existing Vector (Pinecone).

15) **Add Data Table delete node**
- Add **Data Table** named **Delete Vector Metadata Record**
- Operation: `deleteRows`
- Filter: `vector_id == {{ current vector_id }}`
- DataTableId: the same metadata table as step 10.
- Connect Delete Existing Vector (Pinecone) → Delete Vector Metadata Record.

16) **Add text splitter (optional)**
- Add **Recursive Character Text Splitter** named **Split Schema Text (No Chunking)**.
- chunkSize / overlap from global config.
- Connect it into the LangChain document pipeline (see next steps).

17) **Add document loader**
- Add **Default Data Loader (Document)** named **Prepare Schema Document**
- Text/content: the `Create Table` string.
- Metadata: `table_name`, `vector_id`, `source_type`, `schema_hash`.
- Connect Splitter (`ai_textSplitter`) → Prepare Schema Document (`ai_document` output later goes to Pinecone node).

18) **Add OpenAI embeddings node**
- Add **OpenAI Embeddings** named **Generate Schema Embeddings**
- Model: from config
- Configure **OpenAI API credentials**.
- Its `ai_embedding` output must connect to Pinecone vector store node.

19) **Add Pinecone Vector Store node**
- Add **Pinecone Vector Store** named **Insert Schema Vector to Pinecone**
- Mode: **insert**
- Select your Pinecone index in node settings.
- Namespace: from config.
- Configure **Pinecone API credentials**.
- Connect:
  - Prepare Schema Document → Insert Schema Vector to Pinecone (ai_document)
  - Generate Schema Embeddings → Insert Schema Vector to Pinecone (ai_embedding)
- Also connect:
  - Vector Exists? (false) → Insert Schema Vector to Pinecone
  - Delete Vector Metadata Record → Insert Schema Vector to Pinecone (to reinsert after deletion)

20) **Add Data Table upsert node**
- Add **Data Table** named **Upsert Vector Metadata Record**
- Operation: `upsert`
- Filter: `vector_id == current vector_id`
- Set columns: `vector_id`, `table_name`, `schema_hash`
- Connect Insert Schema Vector to Pinecone → Upsert Vector Metadata Record
- Connect Upsert Vector Metadata Record → Loop Over executable queries (to continue).

21) **Run manually**
- Execute the manual trigger to perform initial indexing.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Workflow Overview - MySQL Schema Embeddings in Pinecone (explains controlled sync, idempotency, hashing, deletion + reindex, one vector per table, limitations) | Embedded workflow sticky note |
| ## How to Use This Workflow — @[youtube](elykvVRjVL0) | Video link from sticky note |

### Notable implementation cautions (cross-cutting)
- **Potential field mismatch:** multiple nodes reference `$('Generate Schema Hash').item.json.Table` (capital “T”). Validate whether your MySQL node outputs the table name as `Table`; otherwise switch to `tbl` or the earlier `table_name`.
- **Two sources of truth for metadata table:** one node reads `dataTable_Id` from config, while delete/upsert nodes are hard-wired to a specific Data Table selection. For portability, parameterize all Data Table nodes consistently.
- **Pinecone deletion vs insertion configuration:** deletion uses `vector_index_host` + API key header, insertion uses Pinecone credentials + selected index. Ensure they target the same index/namespace.