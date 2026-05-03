Categorize Revolut Transactions Automatically with GPT-4 and Supabase

https://n8nworkflows.xyz/workflows/categorize-revolut-transactions-automatically-with-gpt-4-and-supabase-8807


# Categorize Revolut Transactions Automatically with GPT-4 and Supabase

### 1. Workflow Overview

This workflow automates the categorization of Revolut bank transactions using GPT-4 AI and Supabase as a backend database. It is designed for users who want to process CSV extracts of Revolut statements, normalize the data, enrich it by AI-powered categorization, and store the results in a structured database for further analysis or reporting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preparation**: Manually triggered start, downloading CSV files from Google Drive, and parsing them into structured data.
- **1.2 Data Normalization and Hashing**: Clean and standardize raw transaction data fields and generate unique hashes to avoid duplicates.
- **1.3 Merchant Extraction**: Apply custom logic to extract and normalize merchant names from transaction descriptions.
- **1.4 Category Retrieval from Supabase**: Fetch the list of allowed expense categories from Supabase to guide AI categorization.
- **1.5 AI Categorization with GPT-4**: Send each transaction data and category list to GPT-4 to classify transactions, detect subscriptions, internal transfers, and return structured metadata.
- **1.6 Data Merge and Insert into Supabase**: Combine enriched data and insert new transaction records into Supabase, handling duplicates gracefully.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

- **Overview:** Starts the workflow manually and downloads a CSV file from Google Drive representing Revolut transactions. It extracts and parses the CSV content for further processing.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Get many rows (Supabase)  
  - Aggregate  
  - Download extract (Google Drive)  
  - Extract from File

- **Node Details:**

1. **When clicking ‘Execute workflow’**  
   - Type: Manual trigger  
   - Role: Manual entry point to start the workflow  
   - Configuration: Default manual trigger, no parameters  
   - Connections: Outputs to "Get many rows"  
   - Edge cases: User must manually trigger or replace with schedule/webhook for automation

2. **Get many rows**  
   - Type: Supabase node  
   - Role: Retrieve all categories from Supabase table `categories`  
   - Configuration: Operation `getAll`, returns all rows  
   - Connections: Outputs to "Aggregate"  
   - Credentials: Supabase connection configured  
   - Edge cases: Supabase connection failure, empty category table

3. **Aggregate**  
   - Type: Aggregate node  
   - Role: Aggregate `normalized_name` field from categories for AI prompt  
   - Configuration: Aggregate on field `normalized_name`  
   - Input: Categories from Supabase  
   - Output: To "Download extract"  
   - Edge cases: No categories or missing fields

4. **Download extract**  
   - Type: Google Drive node  
   - Role: Downloads the CSV file from Google Drive by file ID  
   - Configuration: Uses a hardcoded file ID of the Revolut CSV file  
   - Credentials: Google Drive OAuth2 configured  
   - Output: Sends file binary to "Extract from File"  
   - Edge cases: File not found, permission denied, API limits

5. **Extract from File**  
   - Type: Extract from File node  
   - Role: Parses the CSV file content into JSON items for processing  
   - Configuration: Default CSV extraction  
   - Input: File binary from "Download extract"  
   - Output: Structured JSON per row to "Normalize content"  
   - Edge cases: Malformed CSV, encoding issues

---

#### 2.2 Data Normalization and Hashing

- **Overview:** Normalizes transaction fields to consistent formats, cleans descriptions, and creates a unique hash per transaction to identify duplicates.
- **Nodes Involved:**  
  - Normalize content  
  - Uniq Hash

- **Node Details:**

1. **Normalize content**  
   - Type: Set node  
   - Role: Maps raw CSV columns to normalized field names and formats, e.g., date strings to ISO8601 UTC, description cleaning, uppercase currencies  
   - Configuration: Uses dynamic expressions referencing CSV columns by position  
   - Fields created: type, product, started_date, completed_date, description_original, description_clean (lowercase, trimmed), amount, fee, currency (uppercase), state (uppercase), balance, raw JSON, uniq_hash (string concatenation of key fields)  
   - Output: To "Uniq Hash"  
   - Edge cases: Missing or malformed data fields, date parsing errors, empty strings

2. **Uniq Hash**  
   - Type: Crypto node  
   - Role: Creates a cryptographic hash of the `uniq_hash` string field for deduplication  
   - Configuration: Input is the concatenated string from Normalize content, output stored as `uniq_hash` field  
   - Output: To "Loop Over Items"  
   - Edge cases: Hashing failure (rare), input string empty

---

#### 2.3 Merchant Extraction

- **Overview:** Applies a JavaScript function to refine and normalize the merchant name candidate extracted from the cleaned description field.
- **Nodes Involved:**  
  - Loop Over Items  
  - Extract merchant  
  - Category extraction (after merchant extraction)

- **Node Details:**

1. **Loop Over Items**  
   - Type: SplitInBatches  
   - Role: Processes transactions one by one (batch size default 1), enabling AI calls per item  
   - Input: From "Uniq Hash"  
   - Outputs: Main output to "Create a row" (for completed data), second output to "Extract merchant"  
   - Edge cases: Large batch sizes may overload AI API or slow processing

2. **Extract merchant**  
   - Type: Code node (JavaScript)  
   - Role: Cleans and normalizes merchant names using regex rules and canonical replacements  
   - Configuration:  
     - Normalizes string by removing accents and special chars  
     - Removes noise terms and common business suffixes  
     - Matches known brands (Amazon, Apple, Spotify, Netflix, Uber, Bolt, Carrefour, Lidl, Aldi, Delhaize) for canonical names  
     - Picks first 3 relevant words as fallback merchant name  
   - Input: Transaction JSON with `description_clean`  
   - Output: Adds `merchant_candidate` normalized string to JSON  
   - Edge cases: Unrecognized merchants, empty descriptions, possible misclassification

---

#### 2.4 Category Retrieval from Supabase

- **Overview:** Retrieves the list of allowed categories from Supabase to feed as a prompt context into the AI categorizer.
- **Nodes Involved:** (Covered in 2.1 block)  
  - Get many rows  
  - Aggregate

- **Comments:** These nodes are upstream and supply the category list to the AI prompt dynamically.

---

#### 2.5 AI Categorization with GPT-4

- **Overview:** Sends each normalized transaction with merchant candidate and raw data, plus the allowed categories, to GPT-4 for classification into a single category and flags.
- **Nodes Involved:**  
  - Category extraction (Langchain LLM chain node)  
  - gpt-4.1-mini (OpenAI GPT-4 model)  
  - Output Parser

- **Node Details:**

1. **Category extraction**  
   - Type: Langchain LLM Chain node  
   - Role: Formulates prompt including raw transaction JSON and allowed categories, requests GPT-4 to classify transaction  
   - Configuration:  
     - Prompt instructs GPT-4 to output a JSON object with merchant_name, category, is_subscription, is_internal, confidence (0.0–1.0), and rule_reason  
     - Uses categories from the Aggregate node as allowed categories  
     - Expects a structured JSON output only  
   - Input: Raw transaction JSON + categories  
   - Output: Parsed JSON classification to "Merge"  
   - Edge cases: API errors, malformed responses, rate limits, confidence out of range, unexpected output format

2. **gpt-4.1-mini**  
   - Type: Langchain OpenAI LM Chat node  
   - Role: Executes GPT-4 model with configured prompt and temperature 0.3 for deterministic output  
   - Credentials: OpenAI with valid API key  
   - Output: Passed to "Category extraction" node for parsing  
   - Edge cases: OpenAI API failures, model unavailability, token limits

3. **Output Parser**  
   - Type: Langchain output parser (structured)  
   - Role: Validates and parses GPT-4 JSON output to structured fields  
   - Configuration: JSON schema example provided for validation  
   - Output: To "Category extraction" node main output  
   - Edge cases: Parsing failures if output not valid JSON or schema mismatch

---

#### 2.6 Data Merge and Insert into Supabase

- **Overview:** Combines AI classification with original transaction data and inserts the enriched record into the Supabase `transactions` table. Handles duplicates by continuing on errors.
- **Nodes Involved:**  
  - Merge  
  - Create a row

- **Node Details:**

1. **Merge**  
   - Type: Merge node  
   - Role: Combines the AI classification output with the original transaction data into one JSON object  
   - Configuration: Mode `combineAll` to merge all fields  
   - Inputs: From "Extract merchant" and "Category extraction"  
   - Output: To "Create a row"  
   - Edge cases: Mismatched data keys, missing fields

2. **Create a row**  
   - Type: Supabase node  
   - Role: Inserts the final enriched transaction record into `transactions` table in Supabase  
   - Configuration: Maps all fields including dates, descriptions, merchant_name, category, flags, amounts, currency, user ID, and unique hash  
   - Credentials: Supabase API with write permissions  
   - On Error: Set to continue on error (e.g., duplicate key insertion)  
   - Edge cases: DB errors, unique constraint violations, missing credentials

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                          | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                                                |
|-------------------------------|---------------------------------|----------------------------------------|----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Workflow start                         | -                                | Get many rows                    |                                                                                                                                            |
| Get many rows                 | Supabase                        | Fetch categories from Supabase         | When clicking ‘Execute workflow’ | Aggregate                       | Get Categories from Supabase: Retrieve categories from Supabase to feed into later LLM categorization phases.                              |
| Aggregate                    | Aggregate                       | Aggregate category names                | Get many rows                    | Download extract                |                                                                                                                                            |
| Download extract             | Google Drive                   | Download CSV transaction extract       | Aggregate                       | Extract from File               | Download and Transform extraction: Download file from Google Drive, parse CSV, normalize, generate unique hash.                            |
| Extract from File            | Extract from File               | Parse CSV to JSON                      | Download extract                | Normalize content              |                                                                                                                                            |
| Normalize content            | Set                            | Normalize and clean transaction fields | Extract from File               | Uniq Hash                      |                                                                                                                                            |
| Uniq Hash                   | Crypto                         | Generate unique hash for deduplication | Normalize content               | Loop Over Items                |                                                                                                                                            |
| Loop Over Items             | SplitInBatches                 | Iterate over transactions              | Uniq Hash                      | Extract merchant (second output), Create a row (main output) |                                                                                                                                           |
| Extract merchant            | Code                           | Extract and normalize merchant name    | Loop Over Items (2nd output)     | Category extraction, Merge     | LLM Categorizer: Iterate over transactions, send to LLM with categories, extract category and flags.                                        |
| Category extraction         | Langchain LLM Chain            | AI classify transaction with GPT-4     | Extract merchant, gpt-4.1-mini, Output Parser | Merge                          |                                                                                                                                            |
| gpt-4.1-mini                | Langchain LM Chat OpenAI       | GPT-4 model call                       | Category extraction (prompt input) | Category extraction (AI output parser) |                                                                                                                                            |
| Output Parser              | Langchain Output Parser Structured | Parse AI JSON output                    | gpt-4.1-mini                   | Category extraction            |                                                                                                                                            |
| Merge                      | Merge                          | Combine enriched AI data with transaction | Extract merchant, Category extraction | Create a row                  |                                                                                                                                            |
| Create a row               | Supabase                       | Insert transaction record into database | Merge                         | -                              | Insert into supabase: Then insert into supabase avoiding duplicates.                                                                        |
| Sticky Note                | Sticky Note                    | Documentation / explanation             | -                              | -                              | Revolut Extracts Analyzer: Explains the workflow purpose, use cases, and requirements.                                                     |
| Sticky Note1               | Sticky Note                    | Documentation on categories retrieval   | -                              | -                              | Get Categories from Supabase: Retrieve categories from Supabase to feed into later LLM categorization phases.                              |
| Sticky Note2               | Sticky Note                    | Documentation on download & transform   | -                              | -                              | Download and Transform extraction: Download the file from Google Drive, extract and parse the CSV data, normalize and standardize content.|
| Sticky Note3               | Sticky Note                    | Documentation on AI categorization      | -                              | -                              | LLM Categorizer: Iterate over all transactions, send each to the LLM with category list, extract category plus flags.                      |
| Sticky Note4               | Sticky Note                    | Documentation on insertion               | -                              | -                              | Insert into supabase: Then insert into supabase avoiding duplicates.                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on demand

2. **Add Supabase Node to Get Categories**  
   - Type: Supabase  
   - Operation: `getAll` on table `categories`  
   - Credentials: Setup Supabase API credentials  
   - Output: Pass data to an Aggregate node

3. **Add Aggregate Node**  
   - Type: Aggregate  
   - Operation: Aggregate on field `normalized_name` from categories  
   - Output: Connect to Google Drive node

4. **Add Google Drive Node to Download CSV**  
   - Type: Google Drive  
   - Operation: Download file by `fileId` (set to your Revolut CSV file ID)  
   - Credentials: Google Drive OAuth2 credentials  
   - Output: Connect to Extract from File node

5. **Add Extract from File Node**  
   - Type: Extract from File  
   - Configuration: Default CSV extraction  
   - Input: File from Google Drive  
   - Output: Connect to Set node for normalization

6. **Add Set Node to Normalize Content**  
   - Type: Set  
   - Create fields with expressions:  
     - Map CSV columns by position to fields like `type`, `product`, `started_date` (convert to ISO8601), `completed_date` (ISO8601), `description_original`, `description_clean` (lowercase, trimmed, single spaces), `amount`, `fee`, `currency` (uppercase), `state` (uppercase), `balance`, `raw` (full JSON), `uniq_hash` (concatenate key fields)  
   - Output: Connect to Crypto node

7. **Add Crypto Node to Generate Unique Hash**  
   - Type: Crypto  
   - Operation: Hash the `uniq_hash` string field  
   - Output: Connect to SplitInBatches node

8. **Add SplitInBatches Node**  
   - Type: SplitInBatches  
   - Purpose: Process transactions one by one for AI calls  
   - Output 1 (main): Connect to Supabase insert node  
   - Output 2 (secondary): Connect to Code node for merchant extraction

9. **Add Code Node to Extract Merchant**  
   - Type: Code (JavaScript)  
   - Script: Normalize and clean `description_clean` field, remove noise, canonicalize known merchants, pick first relevant words as fallback  
   - Output: Connect to Langchain LLM Chain node (Category extraction)  
   - Also connect to Merge node (second input)

10. **Add Langchain LLM Chain Node for Category Extraction**  
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
    - Provider: OpenAI GPT-4 (via `gpt-4.1-mini` node)  
    - Prompt: Include raw transaction JSON and allowed categories from Aggregate node  
    - Expected output: JSON with merchant_name, category, is_subscription, is_internal, confidence, rule_reason  
    - Output: Connect to Merge node

11. **Add Langchain LM Chat Node (gpt-4.1-mini)**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Model: `gpt-4.1-mini`  
    - Temperature: 0.3  
    - Credentials: OpenAI API key  
    - Input: Prompt from Category extraction node  
    - Output: Connect to Output Parser node

12. **Add Langchain Output Parser Node**  
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - Configuration: JSON schema example matching expected category extraction JSON  
    - Output: Connect back to Category extraction node

13. **Add Merge Node**  
    - Type: Merge  
    - Mode: Combine all fields from merchant extraction and category AI output  
    - Output: Connect to Supabase Insert node

14. **Add Supabase Node to Insert Transaction**  
    - Type: Supabase  
    - Operation: Insert into `transactions` table  
    - Map fields: completed_date, started_date, description_original, description_clean, merchant_name (from AI), category_name (from AI), type, state, fee, currency, balance, is_subscription, is_internal, amount, uniq_hash, user_id (optional)  
    - On Error: Continue on error to avoid stopping on duplicates  
    - Credentials: Supabase API with write permissions

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                               | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow processes Revolut CSV extracts, automatically categorizes transactions using GPT-4, and stores results in Supabase for dashboarding and financial analysis.                                                                                  | Sticky Note on Workflow overview node                                                                           |
| Requires Google Drive to store CSV files and Supabase tables `categories` and `transactions` correctly configured.                                                                                                                                         | Sticky Note on categories retrieval and data insert nodes                                                      |
| The AI prompt uses a strict JSON output format to ensure reliable parsing and integration. Failures to parse GPT output may require prompt tuning or fallback handling.                                                                                   | Category extraction node prompt details                                                                          |
| Supabase credentials must have read access to `categories` and write access to `transactions`.                                                                                                                                                              | Supabase nodes configuration                                                                                     |
| Consider scheduling the manual trigger or replacing it with a webhook or cron trigger for automation.                                                                                                                                                      | Manual Trigger node comment                                                                                       |
| For best AI accuracy, maintain an updated and comprehensive category list in Supabase.                                                                                                                                                                      | Aggregate node usage                                                                                              |
| The merchant extraction code uses normalization and common noise removal but may need customization for local merchants or languages.                                                                                                                     | Extract merchant code node                                                                                         |
| Documentation and workflow logic inspired by best practices in combining AI with structured financial data workflows.                                                                                                                                    | Sticky Notes content and overall workflow design                                                                 |

---

This detailed reference document provides a complete understanding of the workflow structure, node roles, configurations, and instructions to reproduce or modify the workflow, ensuring robust integration of AI-driven financial transaction categorization.