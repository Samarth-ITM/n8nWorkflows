Transform Excel Data into AI-Ready Vectors with OpenAI and Supabase

https://n8nworkflows.xyz/workflows/transform-excel-data-into-ai-ready-vectors-with-openai-and-supabase-8557


# Transform Excel Data into AI-Ready Vectors with OpenAI and Supabase

### 1. Workflow Overview

This workflow titled **"Transform Excel Data into AI-Ready Vectors with OpenAI and Supabase"** is designed to automate the process of extracting data from an Excel worksheet, cleaning and normalizing it, generating AI embeddings for text fields using OpenAI’s API, and storing the enriched data into a Supabase PostgreSQL database with vector search capabilities.

The key target use case is to prepare structured question-and-answer data from Excel for advanced retrieval-augmented generation (RAG) or semantic search applications by converting textual data into vector embeddings and storing them alongside original metadata.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception and Data Retrieval**: Triggering the workflow manually and fetching rows from Excel and existing Supabase records.
- **1.2 Data Cleaning and Preparation**: Removing HTML artifacts, normalizing date formats, and flagging records for downstream branching.
- **1.3 Batch Processing and Conditional Routing**: Splitting data into manageable batches, then routing records based on whether they contain questions or answers.
- **1.4 Text Processing and Embedding Generation**: Preparing the text for embedding, calling OpenAI API to generate embeddings separately for questions and answers.
- **1.5 Data Merging and Database Writing**: Merging original data with generated embeddings and writing the enriched records into the Supabase database.
- **1.6 Database Setup and Maintenance (Optional/Disabled)**: SQL node to create and maintain the target table schema in Supabase with vector and full-text search support.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Retrieval

**Overview:**  
This block starts the workflow manually and retrieves data from the Excel worksheet and existing rows from the Supabase database for merging.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get rows from sheet (Microsoft Excel)  
- Retrieve existing rows (Supabase)  
- Merge import Data (Merge)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: ManualTrigger  
  - Role: Initiates workflow execution manually  
  - Configuration: Default trigger without parameters  
  - Inputs: None  
  - Outputs: Starts the chain to retrieve data  
  - Edge cases: None; manual trigger only

- **Get rows from sheet**  
  - Type: MicrosoftExcel  
  - Role: Reads rows from a specified Excel workbook and worksheet  
  - Configuration:  
    - Resource: worksheet  
    - Operation: readRows  
    - Workbook ID: `01VJX45VT23GD6IE3HAVD3GBYREFIPIRAO`  
    - Worksheet ID: `{94B58D19-7964-4A20-BFF2-BBE769717A5B}`  
  - Inputs: Manual trigger output  
  - Outputs: Array of JSON rows extracted from Excel  
  - Edge cases: Excel permission errors, empty sheets, malformed data  

- **Retrieve existing rows**  
  - Type: Supabase  
  - Role: Fetches all existing records from the `excel_records` table to enable merging with new data  
  - Configuration:  
    - Table: `excel_records`  
    - Operation: getAll  
    - ReturnAll: true  
    - FilterType: none  
  - Inputs: Manual trigger output  
  - Outputs: List of existing database records  
  - Edge cases: Database connection/auth errors, large dataset performance  

- **Merge import Data**  
  - Type: Merge  
  - Role: Combines new Excel rows and existing Supabase records on matching UUIDs to keep all Excel rows with updated info if present  
  - Configuration:  
    - Mode: combine  
    - Join Mode: keepNonMatches (retain all from Excel, merge existing matches)  
    - Fields to Match: UUID  
    - Output Data From: input1 (Excel data)  
  - Inputs: Excel rows and Supabase rows  
  - Outputs: Merged dataset combining fresh and existing data  
  - Edge cases: UUID mismatches, data type inconsistencies  

---

#### 1.2 Data Cleaning and Preparation

**Overview:**  
Clean HTML from text fields, normalize Excel dates, create cleaned text fields, and add boolean flags indicating presence of sufficient text for questions and answers.

**Nodes Involved:**  
- Remove HTML

**Node Details:**

- **Remove HTML**  
  - Type: Code (JavaScript)  
  - Role: Removes HTML tags/entities from "Question" and "Answer" fields, converts Excel date numbers to ISO date strings, and flags records if they have valid question/answer texts  
  - Configuration: Custom JavaScript code:  
    - Converts Excel serial date to ISO date for 'Creation Date' and 'Closed on'  
    - Cleans HTML tags/entities from 'Question' and 'Answer' fields producing 'Question_clean' and 'Answer_clean'  
    - Adds boolean flags `has_Question` and `has_Answer` if text length > 10  
  - Inputs: Merged dataset from previous block  
  - Outputs: Cleaned and enriched JSON records  
  - Edge cases: Missing fields, NaN dates, malformed HTML, unexpected data types  

---

#### 1.3 Batch Processing and Conditional Routing

**Overview:**  
Splits the cleaned dataset into batches for processing efficiency and routes each item based on whether it contains question or answer data, enabling parallel embedding generation for each.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Switch  
- Merge

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes items one by one (or batch-wise) to manage API call limits and performance  
  - Configuration: reset = false (does not reset batch counter)  
  - Inputs: Cleaned data  
  - Outputs: Items sent individually or in batches for processing  
  - Edge cases: Large batches may cause timeout or API rate limits  

- **Switch**  
  - Type: Switch  
  - Role: Routes each item to "Question" or "Answer" output paths based on boolean flags  
  - Configuration:  
    - Output "Question" if `has_Question` is true  
    - Output "Answer" if `has_Answer` is true (non-empty)  
    - allMatchingOutputs: true (an item can be routed to both if both flags true)  
  - Inputs: Single item from batch  
  - Outputs: One or two paths for question and answer embedding requests  
  - Edge cases: Items with neither flag true are dropped downstream  

- **Merge**  
  - Type: Merge  
  - Role: Combines three input streams: question embeddings, answer embeddings, and original data for later merging  
  - Configuration: numberInputs = 3 (expects three inputs)  
  - Inputs: From question embeddings, answer embeddings, and original data paths  
  - Outputs: Combined data array for merging embeddings into records  
  - Edge cases: Missing inputs cause incomplete merges  

---

#### 1.4 Text Processing and Embedding Generation

**Overview:**  
Prepare the text fields for embedding: conditionally output only question or answer text to OpenAI, call OpenAI embeddings API for each, and receive numeric vector embeddings.

**Nodes Involved:**  
- Code "Question"  
- Code "Answer"  
- Embeddings OpenAI Question (HTTP Request)  
- Embeddings OpenAI Answer (HTTP Request)

**Node Details:**

- **Code "Question"**  
  - Type: Code (JavaScript)  
  - Role: Checks if question text is valid, prepares JSON with `record_id`, cleaned question text (`text_to_embed`), and original item for embedding  
  - Inputs: Items from Switch node "Question" output  
  - Outputs: JSON with text to embed or empty if no valid question  
  - Edge cases: Missing fields cause empty output, which blocks downstream calls  

- **Code "Answer"**  
  - Type: Code (JavaScript)  
  - Role: Similar to "Question" code but for the answer text, outputs just cleaned answer text for embedding  
  - Inputs: Items from Switch node "Answer" output  
  - Outputs: JSON with `text_to_embed` or empty array if invalid  
  - Edge cases: Missing fields cause empty output  

- **Embeddings OpenAI Question**  
  - Type: HTTP Request  
  - Role: Sends question text to OpenAI embeddings API and receives vector embedding  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.openai.com/v1/embeddings`  
    - JSON body includes model: "text-embedding-3-small", input text, encoding_format: "float"  
    - Authentication: configured with OpenAI API credentials ("openAiApi")  
  - Inputs: JSON from "Code Question" node  
  - Outputs: JSON with embeddings array  
  - Edge cases: API key invalid, rate limits, malformed input, network errors  

- **Embeddings OpenAI Answer**  
  - Type: HTTP Request  
  - Role: Same as above but for answer text embeddings  
  - Configuration: Same as question embeddings node  
  - Inputs: JSON from "Code Answer" node  
  - Outputs: Embeddings JSON  
  - Edge cases: Same as question embeddings node  

---

#### 1.5 Data Merging and Database Writing

**Overview:**  
Merge original data with received embeddings from OpenAI, then write the enriched record into the Supabase database table `excel_records`.

**Nodes Involved:**  
- Merge fields for database insert (Code)  
- Write row to database (Supabase)

**Node Details:**

- **Merge fields for database insert**  
  - Type: Code (JavaScript)  
  - Role: Aggregates the original record and the embeddings for question and answer into one JSON object for database insertion  
  - Configuration: JavaScript code that:  
    - Scans all inputs  
    - Assigns original data, question embedding, and answer embedding based on presence  
    - Constructs a JSON object merging all fields with embeddings under `Question_embedding` and `Answer_embedding`  
  - Inputs: All three merged streams: original data, question embedding, answer embedding  
  - Outputs: Single JSON object ready for database insertion  
  - Edge cases: Missing embeddings, input ordering issues, null UUIDs  

- **Write row to database**  
  - Type: Supabase  
  - Role: Inserts or updates the merged record into Supabase table `excel_records`  
  - Configuration:  
    - Table ID: `excel_records`  
    - Fields mapped from JSON keys to table columns (UUID, Projectcode, boolean flags, handlers, dates, emails, questions, answers, and embeddings)  
    - Uses Supabase credentials with appropriate permissions  
  - Inputs: Output from "Merge fields for database insert"  
  - Outputs: Confirmation or error of database write operation  
  - Edge cases: Constraint violations (unique UUID), credential/auth errors, network issues  

---

#### 1.6 Database Setup and Maintenance (Disabled Node)

**Overview:**  
Provides SQL commands to create the target table in Supabase with necessary extensions, columns, indexes, triggers for full-text search and vector indexing.

**Nodes Involved:**  
- Build table in Supabase (Postgres) [Disabled]

**Node Details:**

- **Build table in Supabase**  
  - Type: Postgres (Execute Query)  
  - Role: Defines and recreates the `excel_records` table schema with columns for original data, embeddings as `vector(1536)`, and full-text search vectors  
  - Configuration:  
    - SQL script includes:  
      - Enabling extensions `pgcrypto` and `vector`  
      - Table creation with all fields and constraints  
      - Triggers for updating timestamps and full-text search columns  
      - Indexes for full-text search and vector similarity (IVFFLAT)  
      - Row-level security policies  
  - Inputs: None (manual execution)  
  - Outputs: Success or failure of schema setup  
  - Edge cases: Requires Supabase PostgreSQL with extensions available, permissions for schema changes  

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                                   | Input Node(s)                      | Output Node(s)                       | Sticky Note                                                                                                               |
|----------------------------|-------------------------|-------------------------------------------------|----------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | ManualTrigger           | Start the workflow manually                       | None                             | Get rows from sheet, Retrieve existing rows | Overview sticky note: Simple upload from Excel to Supabase with RAG                                                        |
| Get rows from sheet         | MicrosoftExcel          | Retrieve rows from Excel worksheet                | When clicking ‘Execute workflow’ | Merge import Data                   | Sticky Note - Get rows from sheet                                                                                          |
| Retrieve existing rows      | Supabase                | Retrieve existing rows from Supabase table        | When clicking ‘Execute workflow’ | Merge import Data                   | Sticky Note - Retrieve existing rows                                                                                       |
| Merge import Data           | Merge                   | Combine Excel rows with existing Supabase data    | Get rows from sheet, Retrieve existing rows | Remove HTML                       | Sticky Note - Merge import Data                                                                                            |
| Remove HTML                | Code (JavaScript)       | Clean HTML/normalize dates, add flags             | Merge import Data                | Loop Over Items                    | Sticky Note - Remove HTML                                                                                                  |
| Loop Over Items            | SplitInBatches          | Process items in batches or individually           | Remove HTML                     | Switch, Merge                      | Sticky Note - Loop Over Items                                                                                              |
| Switch                    | Switch                  | Route data to question or answer processing paths | Loop Over Items                 | Code "Question", Code "Answer", Merge | Sticky Note - Switch                                                                                                       |
| Code "Question"            | Code (JavaScript)       | Prepare question text for embedding                | Switch (Question output)         | Embeddings OpenAI Question         | Sticky Note - Code "Question"                                                                                             |
| Code "Answer"              | Code (JavaScript)       | Prepare answer text for embedding                  | Switch (Answer output)           | Embeddings OpenAI Answer           | Sticky Note - Code "Answer"                                                                                               |
| Embeddings OpenAI Question | HTTP Request            | Call OpenAI API for question embeddings            | Code "Question"                 | Merge                             | Sticky Note - Embeddings OpenAI Question                                                                                   |
| Embeddings OpenAI Answer   | HTTP Request            | Call OpenAI API for answer embeddings              | Code "Answer"                   | Merge                             | Sticky Note - Embeddings OpenAI Question                                                                                   |
| Merge                     | Merge                   | Combine embeddings and original data               | Embeddings OpenAI Question, Embeddings OpenAI Answer, Loop Over Items (original data) | Merge fields for database insert | Sticky Note - Merge                                                                                                        |
| Merge fields for database insert | Code (JavaScript)       | Merge original record with embeddings for DB insert | Merge                           | Write row to database             | Sticky Note - Merge fields for database insert                                                                             |
| Write row to database      | Supabase                | Insert enriched record into Supabase database      | Merge fields for database insert | Loop Over Items                   | Sticky Note - Write row to database                                                                                        |
| Build table in Supabase    | Postgres (SQL)          | (Disabled) Setup database schema and indexes       | None                           | None                            | Sticky Note: SQL and schema setup instructions                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To start the workflow manually.

2. **Add Microsoft Excel Node to Get Rows**  
   - Name: `Get rows from sheet`  
   - Resource: `worksheet`  
   - Operation: `readRows`  
   - Configure with your Excel Workbook ID and Worksheet ID.  
   - Connect output from manual trigger node.

3. **Add Supabase Node to Retrieve Existing Rows**  
   - Name: `Retrieve existing rows`  
   - Operation: `getAll`  
   - Table ID: `excel_records`  
   - Return All: `true`  
   - Connect output from manual trigger node.

4. **Add Merge Node to Combine Excel and Supabase Data**  
   - Name: `Merge import Data`  
   - Mode: `combine`  
   - Join Mode: `keepNonMatches`  
   - Fields to Match: `UUID`  
   - Output Data From: `input1` (Excel)  
   - Connect inputs from `Get rows from sheet` and `Retrieve existing rows`.

5. **Add Code Node to Clean HTML and Normalize Dates**  
   - Name: `Remove HTML`  
   - Paste JavaScript code that:  
     - Converts Excel numeric dates to ISO strings  
     - Removes HTML tags/entities from `Question` and `Answer` fields  
     - Creates `Question_clean` and `Answer_clean`  
     - Adds boolean flags `has_Question` and `has_Answer` if text length > 10  
   - Connect output from `Merge import Data`.

6. **Add SplitInBatches Node to Process Items Individually**  
   - Name: `Loop Over Items`  
   - Options: Reset = false  
   - Connect output from `Remove HTML`.

7. **Add Switch Node to Route Items Based on Content**  
   - Name: `Switch`  
   - Rules:  
     - Output "Question" if `has_Question == true`  
     - Output "Answer" if `has_Answer == true` (non-empty)  
   - allMatchingOutputs enabled to send to both if conditions met  
   - Connect output from `Loop Over Items`.

8. **Add Two Code Nodes to Prepare Text for Embeddings**  
   - Name: `Code "Question"`  
     - JavaScript: Return JSON with `record_id`, `text_to_embed` from `Question_clean`, and original item if valid; else empty array.  
   - Name: `Code "Answer"`  
     - JavaScript: Return JSON with `text_to_embed` from `Answer_clean` if valid; else empty array.  
   - Connect outputs of `Switch` accordingly.

9. **Add Two HTTP Request Nodes for OpenAI Embeddings**  
   - Name: `Embeddings OpenAI Question` and `Embeddings OpenAI Answer`  
   - Method: POST  
   - URL: `https://api.openai.com/v1/embeddings`  
   - Authentication: Setup OpenAI API credentials (credential named `openAiApi`)  
   - JSON Body:  
     ```json
     {
       "model": "text-embedding-3-small",
       "input": "{{ $json.text_to_embed }}",
       "encoding_format": "float"
     }
     ```  
   - Connect outputs from respective Code nodes.

10. **Add Merge Node to Combine Embeddings and Original Data**  
    - Name: `Merge`  
    - Number Inputs: 3  
    - Connect outputs from:  
      - `Embeddings OpenAI Question` (input 1)  
      - `Embeddings OpenAI Answer` (input 2)  
      - `Loop Over Items` original data (input 3)

11. **Add Code Node to Merge Fields for Database Insert**  
    - Name: `Merge fields for database insert`  
    - JavaScript: Combine original data and embeddings from inputs into one JSON object with fields: `UUID`, original data, `Question_embedding`, `Answer_embedding`.  
    - Connect output from `Merge`.

12. **Add Supabase Node to Write Row to Database**  
    - Name: `Write row to database`  
    - Operation: Upsert or Insert into table `excel_records`  
    - Map all relevant fields (UUID, Projectcode, booleans, handler, status, dates, email, city, questions, answers, embeddings) from JSON data  
    - Ensure Supabase credentials with write permissions are configured  
    - Connect output from `Merge fields for database insert`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Adjust the column schema in both Supabase and Excel to match your requirements. The example uses UUID, Projectcode, Bool 1, Bool 2, Bool 3, Handler, Status, Creation Date, Closed on, E-mailadres, City, Question, and Answer fields.                                                                                                                                                                                                                                                                                      | Sticky Note referencing SQL schema and Excel columns                                                  |
| The SQL script for creating the `excel_records` table includes enabling `pgcrypto` and `vector` extensions, full-text search columns and triggers, vector indexes for cosine similarity, and row-level security policies. Adjust embedding dimension if using a different OpenAI model.                                                                                                                                                                                                                                  | SQL schema node and sticky note                                                                        |
| When using OpenAI API, monitor your API usage and quota to avoid hitting rate limits or incurring unexpected costs. Ensure that the text fields processed are valid strings without nulls to prevent API errors.                                                                                                                                                                                                                                                                                                         | Sticky Notes on OpenAI embeddings nodes                                                               |
| Ensure Supabase credentials have the appropriate permissions for data insertion and retrieval. Validate email fields to match the database constraint pattern.                                                                                                                                                                                                                                                                                                                                                         | Sticky Notes on Supabase nodes                                                                         |
| The workflow processes question and answer texts separately to generate distinct embeddings, allowing fine-grained semantic search over both fields in the database.                                                                                                                                                                                                                                                                                                                                                    | Overall workflow logic                                                                                 |
| For large datasets, consider limiting batch sizes in the SplitInBatches node to avoid timeouts and rate limits.                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note on Loop Over Items                                                                         |
| The workflow assumes that the Excel input fields are consistently named and populated. Any missing or malformed data may cause failures or empty outputs in downstream nodes.                                                                                                                                                                                                                                                                                                                                           | Sticky Notes on Remove HTML and Code nodes                                                            |
| The disabled SQL node can be used once to initialize or reset the Supabase table before running the workflow for the first time or after schema changes.                                                                                                                                                                                                                                                                                                                                                                | Disabled "Build table in Supabase" node                                                               |

---

**Disclaimer**: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.