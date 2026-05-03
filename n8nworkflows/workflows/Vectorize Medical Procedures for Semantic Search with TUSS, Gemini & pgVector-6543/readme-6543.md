Vectorize Medical Procedures for Semantic Search with TUSS, Gemini & pgVector

https://n8nworkflows.xyz/workflows/vectorize-medical-procedures-for-semantic-search-with-tuss--gemini---pgvector-6543


# Vectorize Medical Procedures for Semantic Search with TUSS, Gemini & pgVector

### 1. Workflow Overview

This workflow, titled **"Vectorize Medical Procedures for Semantic Search with TUSS, Gemini & pgVector"**, is designed to transform medical procedure data from an Oracle database into vector embeddings stored in a PostgreSQL database using the pgVector extension. The workflow targets use cases involving semantic search and natural language processing of medical procedure descriptions, leveraging Google Gemini embeddings for vectorization.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception and Data Extraction**: Manual trigger initiates the process, then connects to an Oracle database to fetch medical procedure data.
- **1.2 Data Preparation and Formatting**: The raw database rows are extracted and prepared for vectorization.
- **1.3 Batch Processing and Vectorization**: Data is split into manageable batches, converted into vector embeddings using Google Gemini, and then stored in a PostgreSQL vector store with pgVector.
- **1.4 Data Loader and Metadata Assignment**: Prepares documents with metadata for accurate embedding and storage, ensuring semantic search utility.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Extraction

- **Overview:**  
This block is responsible for initiating the workflow and extracting the medical procedures data from an Oracle database. It fetches two key columns: procedure code (`CD_ITEM`) and description (`DS_ITEM`).

- **Nodes Involved:**  
  - VECTORIZE TUSS TABLE (Manual Trigger)  
  - ORACLE DATABASE CONNECTION (Oracle DB Query)  
  - COLLECTION OF MEDICAL PROCEDURES (Code Node)

- **Node Details:**

  - **VECTORIZE TUSS TABLE**  
    - Type: Manual Trigger  
    - Configuration: No parameters; starts workflow manually  
    - Input: None  
    - Output: Triggers "ORACLE DATABASE CONNECTION"  
    - Potential Failures: Trigger not initiated by user

  - **ORACLE DATABASE CONNECTION**  
    - Type: Oracle Database Parameterized Query  
    - Configuration: Runs SQL query to select `CD_ITEM` and `DS_ITEM` from specified table (user must replace placeholder `{sua tabela aqui}` with actual table name)  
    - Credentials: Oracle credentials (`PROTHEUS - PRODUÇÃO (APPN8N)`)  
    - Input: Trigger from manual node  
    - Output: Query results passed to "COLLECTION OF MEDICAL PROCEDURES"  
    - Edge Cases: Incorrect table name, query errors, DB connection timeouts or authentication failures

  - **COLLECTION OF MEDICAL PROCEDURES**  
    - Type: Code Node (JavaScript)  
    - Configuration: Extracts the `.rows` array from the Oracle query result to simplify downstream processing  
    - Input: Output from Oracle DB query node  
    - Output: Array of procedure objects passed to batch splitter  
    - Key Expression: `return $input.first().json.rows;`  
    - Potential Failures: Input data structure changes, empty or null results

---

#### 2.2 Batch Processing and Vectorization

- **Overview:**  
This block splits the large dataset into batches, converts each medical procedure text into embeddings using Google Gemini, and inserts the resulting vectors into a PostgreSQL database with pgVector.

- **Nodes Involved:**  
  - FOR - MEDICAL PROCEDURES (Split in Batches)  
  - Embeddings Google Gemini (Embedding Generation)  
  - Postgres PGVector Store2 (Vector Storage)

- **Node Details:**

  - **FOR - MEDICAL PROCEDURES**  
    - Type: SplitInBatches  
    - Configuration: Default batch settings; splits input array into smaller chunks for processing  
    - Input: Array of medical procedures from "COLLECTION OF MEDICAL PROCEDURES"  
    - Output: Individual batches sent to Embeddings node and also triggers vector insertion node  
    - Edge Cases: Very large dataset causing memory issues, batch size adjustments might be required  

  - **Embeddings Google Gemini**  
    - Type: Google Gemini Embeddings Node  
    - Configuration: Uses Google Gemini API to generate vector embeddings for input text  
    - Credentials: Google Palm API credentials (`GEMINI - PRODUÇÃO`)  
    - Input: Batches of medical procedure data (text)  
    - Output: Vector embeddings passed to PostgreSQL storage  
    - Edge Cases: API rate limits, invalid or malformed text input, network timeouts  

  - **Postgres PGVector Store2**  
    - Type: LangChain PGVector Store Node  
    - Configuration: Insert mode enabled; target PostgreSQL table name placeholder `{your_table_name_here}` to be replaced by user  
    - Credentials: PostgreSQL credentials (`N8N`)  
    - Input: Embeddings from Gemini node  
    - Output: Confirmation or error; triggers batch splitter for next batch  
    - Edge Cases: Database connection failure, schema mismatches, duplicate entries, insertion errors

---

#### 2.3 Data Loader and Metadata Assignment (Inactive)

- **Overview:**  
This block is designed to format each medical procedure with metadata (procedure code and description) before vectorization, enhancing semantic search precision. However, this block is currently set as inactive in the workflow.

- **Nodes Involved:**  
  - Data Loader (LangChain Document Data Loader)  
  - Token Splitter (Text Splitter)  
  - Sticky Note2 (Documentation)

- **Node Details:**

  - **Data Loader**  
    - Type: LangChain Document Default Data Loader  
    - Configuration: Formats input into structured JSON with metadata fields `cd_item` and `procedureName` extracted from each record; textual content includes procedure code and description  
    - Input: Output of Token Splitter (which is currently not connected)  
    - Output: Document objects prepared for embedding  
    - Edge Cases: Expression evaluation errors, metadata missing, inactive status means no runtime effect  

  - **Token Splitter**  
    - Type: Text Splitter (Token-based)  
    - Configuration: Splits text into chunks of 100 tokens for better embedding handling  
    - Input: (Not connected)  
    - Output: Intended for Data Loader node input  
    - Edge Cases: Unused due to inactive status  

  - **Sticky Note2**  
    - Content: Explains the role of data loader in preparing and formatting textual data with metadata for semantic search and vectorization

---

#### 2.4 Sticky Notes and Documentation

Sticky notes provide guidance on key aspects of the workflow:

- **Sticky Note (at [0,0])**:  
  - Advises user to provide the TUSS table with columns `CD_ITEM` and `DS_ITEM` in the database.

- **Sticky Note1 (at [1200,48])**:  
  - Explains that vectorization is performed in a PostgreSQL database using pgVector.

- **Sticky Note2 (at [1616,480])**:  
  - Details the data loader's role in preparing textual data and metadata to generate precise vectors for semantic search.

---

### 3. Summary Table

| Node Name                 | Node Type                                           | Functional Role                           | Input Node(s)            | Output Node(s)              | Sticky Note                                                                                       |
|---------------------------|----------------------------------------------------|-----------------------------------------|--------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| VECTORIZE TUSS TABLE       | Manual Trigger                                     | Workflow start trigger                   | None                     | ORACLE DATABASE CONNECTION   |                                                                                                 |
| ORACLE DATABASE CONNECTION | Oracle Database with Parameterization              | Fetch medical procedures from Oracle DB | VECTORIZE TUSS TABLE      | COLLECTION OF MEDICAL PROCEDURES |                                                                                                 |
| COLLECTION OF MEDICAL PROCEDURES | Code Node (JavaScript)                              | Extracts rows array from DB query result | ORACLE DATABASE CONNECTION | FOR - MEDICAL PROCEDURES      |                                                                                                 |
| FOR - MEDICAL PROCEDURES   | SplitInBatches                                     | Processes data in batches                | COLLECTION OF MEDICAL PROCEDURES | Embeddings Google Gemini, Postgres PGVector Store2 |                                                                                                 |
| Embeddings Google Gemini   | Google Gemini Embeddings                           | Generates vector embeddings              | FOR - MEDICAL PROCEDURES  | Postgres PGVector Store2      |                                                                                                 |
| Postgres PGVector Store2   | LangChain PGVector Store                           | Stores embeddings in PostgreSQL          | Embeddings Google Gemini, FOR - MEDICAL PROCEDURES | FOR - MEDICAL PROCEDURES      | Sticky Note1: "## VECTORIZATION\n\nHERE THE VECTORIZATION WILL BE PERFORMED IN A POSTGRES DATABASE USING PGVECTOR" |
| Token Splitter             | Text Splitter (Token-based)                        | Splits text into token chunks            | None (inactive)           | Data Loader (inactive)        | Sticky Note2: "## DATA LOADER\n\nResponsible for preparing and formatting the textual data before vectorization..." |
| Data Loader               | LangChain Document Data Loader                     | Prepares documents with metadata         | Token Splitter (inactive) | Postgres PGVector Store2 (connected but inactive) | Sticky Note2 (same as above)                                                                    |
| Sticky Note               | Sticky Note                                        | Documentation and guidance               | None                     | None                        | "## DATABASE AND/OR FILE CONNECTION\n\n### HERE YOU SHOULD PROVIDE THE TUSS TABLE, WITH ONE COLUMN NAMED CD_ITEM AND ANOTHER NAMED DS_ITEM" |
| Sticky Note1              | Sticky Note                                        | Documentation on vectorization method    | None                     | None                        | "## VECTORIZATION\n\nHERE THE VECTORIZATION WILL BE PERFORMED IN A POSTGRES DATABASE USING PGVECTOR" |
| Sticky Note2              | Sticky Note                                        | Documentation on data loader role        | None                     | None                        | "## DATA LOADER\n\nResponsible for preparing and formatting the textual data before vectorization, ensuring that each piece of information has the correct structure and metadata—such as the medical procedure code and its description—to generate precise and useful vectors for semantic search." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `VECTORIZE TUSS TABLE`  
   - Type: Manual Trigger  
   - No parameters needed

2. **Create Oracle Database Node**  
   - Name: `ORACLE DATABASE CONNECTION`  
   - Type: Oracle Database Parameterization Node  
   - Set credentials: Oracle credentials for your environment (e.g., `PROTHEUS - PRODUÇÃO (APPN8N)`)  
   - Query: `SELECT CD_ITEM, DS_ITEM FROM your_table_name` (replace placeholder)  
   - Connect manual trigger output to this node input

3. **Create Code Node to Extract Rows**  
   - Name: `COLLECTION OF MEDICAL PROCEDURES`  
   - Type: Code Node (JavaScript)  
   - JavaScript code: `return $input.first().json.rows;`  
   - Connect Oracle DB node output to this node input

4. **Create SplitInBatches Node**  
   - Name: `FOR - MEDICAL PROCEDURES`  
   - Type: SplitInBatches  
   - Default batch size or adjust as needed for dataset size  
   - Connect Code node output to this node input

5. **Create Embeddings Node for Google Gemini**  
   - Name: `Embeddings Google Gemini`  
   - Type: LangChain Google Gemini Embeddings  
   - Set credentials: Google Palm API credentials (e.g., `GEMINI - PRODUÇÃO`)  
   - Connect SplitInBatches node output to this node input (for embedding generation)

6. **Create PostgreSQL PGVector Store Node**  
   - Name: `Postgres PGVector Store2`  
   - Type: LangChain PGVector Store for PostgreSQL  
   - Set credentials: PostgreSQL credentials (e.g., `N8N`)  
   - Configure mode: Insert  
   - Set target table name (replace `{your_table_name_here}` with actual table)  
   - Connect Embeddings node output to this node input  
   - Connect Postgres node output back to SplitInBatches node to continue batch processing

7. **Create Sticky Notes for Documentation**  
   - Add a sticky note near the start explaining the required database table and columns (`CD_ITEM`, `DS_ITEM`)  
   - Add another sticky note near the PGVector node describing that vectorization happens in PostgreSQL  
   - Add a sticky note near the Data Loader nodes explaining their role (optional, as these are inactive)

8. **(Optional) Data Loader and Token Splitter Nodes**  
   - Create Token Splitter node (Text Splitter, chunk size 100 tokens)  
   - Create Data Loader node (LangChain Document Default Data Loader)  
   - Configure metadata mapping of `cd_item` and `procedureName` fields from JSON expressions  
   - Connect Token Splitter output to Data Loader input  
   - Connect Data Loader output to Postgres PGVector Store node (if activating this flow)  
   - Note: These nodes are inactive in current workflow and can be omitted unless needed for enhanced data preparation

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow requires replacing placeholder table names with actual tables in Oracle and PostgreSQL databases. | See Sticky Note at workflow start for database requirements.                                           |
| Vectorization leverages Google Gemini embeddings and stores vectors in PostgreSQL using pgVector extension.    | Sticky Note1 explains this vectorization approach.                                                    |
| The Data Loader and Token Splitter nodes are designed for better metadata handling but are currently inactive.  | Sticky Note2 describes their purpose for semantic search accuracy.                                    |
| Oracle database credentials and Google Palm API credentials need to be properly configured in n8n credentials. | Ensure correct OAuth or API key setup in n8n credential manager.                                       |
| PostgreSQL must have pgVector extension enabled and target table schema compatible with vector data insertion. | Consult pgVector documentation for table schema setup: https://github.com/pgvector/pgvector            |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.