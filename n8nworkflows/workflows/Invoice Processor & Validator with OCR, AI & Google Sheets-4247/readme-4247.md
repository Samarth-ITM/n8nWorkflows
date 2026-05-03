Invoice Processor & Validator with OCR, AI & Google Sheets

https://n8nworkflows.xyz/workflows/invoice-processor---validator-with-ocr--ai---google-sheets-4247


# Invoice Processor & Validator with OCR, AI & Google Sheets

---

### 1. Workflow Overview

This workflow automates the processing, validation, and recording of invoice data extracted from local PDF files. It integrates Optical Character Recognition (OCR), AI-powered text extraction, and Google Sheets to enable efficient, accurate handling and verification of invoice line items against master data.

Logical blocks:

- **1.1 Input Reception and File Reading:** Triggering the workflow manually and reading the invoice PDF file from disk.
- **1.2 Invoice Text Extraction:** Extracting raw text from the PDF file using OCR capabilities.
- **1.3 AI-Based Invoice Parsing:** Using an AI language model to convert raw invoice text into structured JSON invoice data.
- **1.4 Post-Processing and JSON Validation:** Cleaning and parsing the AI output to ensure valid JSON structure.
- **1.5 Error Handling and Retry:** Detecting parsing errors and retrying extraction with raw text if needed.
- **1.6 Line Items Splitting and Unique Key Generation:** Splitting extracted line items for individual processing and generating unique identifiers.
- **1.7 Sending Parsed Data to Google Sheets:** Appending extracted invoice details to a dedicated Google Sheet.
- **1.8 Master Data Fetching and Validation:** Retrieving master SKU data from another Google Sheet and validating invoice items using fuzzy matching.
- **1.9 Updating Validation Results:** Updating the Google Sheet with validation outcomes, price discrepancies, and notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Reading

- **Overview:**  
  Initiates the workflow via manual trigger and reads the invoice PDF file stored locally.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Read/Write Files from Disk

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution  
    - Config: No parameters; triggers workflow execution on demand  
    - Inputs: None  
    - Outputs: Triggers next node  

  - **Read/Write Files from Disk**  
    - Type: File Reader  
    - Role: Reads invoice PDF file from local storage  
    - Config: File path set to a specific local invoice PDF (e.g., `E:/SentIImenta AI/.../661.pdf`)  
    - Inputs: Trigger from manual node  
    - Outputs: File binary data to next node  
    - Edge cases: File not found, access permission errors  

---

#### 2.2 Invoice Text Extraction

- **Overview:**  
  Extracts raw textual content from the PDF using built-in PDF extraction.

- **Nodes Involved:**  
  - Extract from File

- **Node Details:**

  - **Extract from File**  
    - Type: PDF Text Extractor  
    - Role: Converts PDF binary data into raw text  
    - Config: Operation set to “pdf” extraction  
    - Inputs: Binary PDF data from file reader  
    - Outputs: JSON containing extracted text  
    - Edge cases: Unsupported PDF formats, extraction inaccuracies  

---

#### 2.3 AI-Based Invoice Parsing

- **Overview:**  
  Uses an AI chat model to parse raw invoice text into a strictly formatted JSON object specifying invoice details and line items.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - Text Extractor

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: AI Language Model (OpenRouter)  
    - Role: Processes raw text with a custom model (`deepseek/deepseek-chat-v3-0324`) to extract structured invoice data  
    - Config: Uses credentials linked to OpenRouter API account “sentiimenta.ai”  
    - Inputs: Raw text from previous node (via Text Extractor)  
    - Outputs: AI-generated raw JSON string  

  - **Text Extractor**  
    - Type: Langchain Agent Node  
    - Role: Applies strict prompt instructions to enforce JSON-only responses, including detailed extraction rules for invoice fields and line items  
    - Config: System message directs extraction of invoice fields (invoice number, vendor name, dates, totals, tax, line items with detailed subfields), includes validation that all line items sum exactly to total amount  
    - Key Expressions: Template uses `{{ $json.text }}` to feed text into prompt  
    - Inputs: Extracted invoice text from previous node  
    - Outputs: Raw JSON string output (still needs cleaning)  
    - Edge cases: AI incomplete or invalid JSON output, missing fields  

---

#### 2.4 Post-Processing and JSON Validation

- **Overview:**  
  Cleans AI output by removing markdown backticks, extracting JSON block, unescaping characters, and performing safe JSON parsing.

- **Nodes Involved:**  
  - Post-Processing  
  - If

- **Node Details:**

  - **Post-Processing**  
    - Type: Code Node (JavaScript)  
    - Role: Extracts valid JSON from AI raw text output, handles common formatting issues (e.g., backticks, escaped characters)  
    - Key Expressions: Parses raw string, attempts double JSON parse if needed, returns error details on failure  
    - Inputs: AI output JSON string  
    - Outputs: Parsed JSON object or error object  
    - Edge cases: Missing JSON block, parsing exceptions  

  - **If**  
    - Type: Conditional Node  
    - Role: Checks if JSON parsing failed or no valid JSON block found  
    - Config: Conditions test if `error` key equals “JSON parsing failed” or “No valid JSON block found”  
    - Inputs: Output from Post-Processing  
    - Outputs: Two branches: error branch (retry) and success branch (continue)  

---

#### 2.5 Error Handling and Retry

- **Overview:**  
  On JSON parsing failure, retries extraction by feeding raw invoice text again into the AI text extractor node.

- **Nodes Involved:**  
  - Send Raw Text Again

- **Node Details:**

  - **Send Raw Text Again**  
    - Type: Set Node  
    - Role: Assigns raw extracted text from the PDF to variable `text` to retry AI extraction  
    - Config: Text set from `'Extract from File'.item.json.text`  
    - Inputs: From If node error branch  
    - Outputs: Routes back to Text Extractor to retry parsing  
    - Edge cases: Infinite loop prevention not shown (may require manual intervention if persistent failure)  

---

#### 2.6 Line Items Splitting and Unique Key Generation

- **Overview:**  
  Splits parsed invoice line items into separate items for individual processing, then generates a unique key for each item combining invoice number and item index.

- **Nodes Involved:**  
  - Split Out  
  - Generate Unique Key

- **Node Details:**

  - **Split Out**  
    - Type: Split Out Node  
    - Role: Decomposes `line_items` array into individual nodes/items for granular processing  
    - Config: Field to split on is `line_items` from parsed JSON  
    - Inputs: Valid JSON from Post-Processing success branch  
    - Outputs: Individual line item JSON objects  

  - **Generate Unique Key**  
    - Type: Set Node  
    - Role: Creates a unique identifier for each line item by concatenating invoice number and item index  
    - Config: Expression `={{ $('If').item.json.invoice_number + "-" + $itemIndex }}`  
    - Inputs: Split Out items  
    - Outputs: Line items enriched with `Unique Key` field  

---

#### 2.7 Sending Parsed Data to Google Sheets

- **Overview:**  
  Appends each invoice line item with all extracted details to a Google Sheet for record-keeping.

- **Nodes Involved:**  
  - Send Invoice Data

- **Node Details:**

  - **Send Invoice Data**  
    - Type: Google Sheets Node (Append)  
    - Role: Appends invoice line items to a dedicated Google Sheet named “Invoice Validation” (Sheet1)  
    - Config: Maps multiple columns including invoice details, PO info, tax rates, quantities, unit prices, HSN, and combined description fields  
    - Credentials: Uses OAuth2 credentials named “Google Sheets account 3”  
    - Inputs: From Generate Unique Key  
    - Outputs: Triggers next step fetching master data  
    - Edge cases: API rate limits, permission errors, schema mismatches  

---

#### 2.8 Master Data Fetching and Validation

- **Overview:**  
  Fetches master SKU and pricing data from a separate Google Sheet and performs detailed validation of invoice items against this data using fuzzy matching techniques.

- **Nodes Involved:**  
  - Fetch Master Data  
  - Validation

- **Node Details:**

  - **Fetch Master Data**  
    - Type: Google Sheets Node (Read)  
    - Role: Retrieves master SKU data from a sheet named “Cost-Priority sheet-GPT” in a different Google Sheet document  
    - Credentials: Same Google Sheets OAuth2 account  
    - Inputs: From Send Invoice Data  
    - Outputs: Supplies rows to validation code  
    - Edge cases: Empty or malformed master data  

  - **Validation**  
    - Type: Code Node (JavaScript)  
    - Role: Performs fuzzy matching and validation of extracted invoice line items against master data using Levenshtein Distance algorithm  
    - Key Logic:  
      - Cleans strings, concatenates code, description, and last character into SKU string  
      - Finds best match in master data based on minimal Levenshtein distance (threshold = 5)  
      - Checks if ASINs and costs match exactly  
      - Prepares detailed validation result messages and flags  
    - Inputs: Items from Split Out, Generate Unique Key, and master data rows  
    - Outputs: Validation results for each line item  
    - Edge cases: No close match found, cost mismatch, ASIN mismatch  

---

#### 2.9 Updating Validation Results

- **Overview:**  
  Updates the original Google Sheet with validation results, price differences, and notes for each invoice line item.

- **Nodes Involved:**  
  - Update Results

- **Node Details:**

  - **Update Results**  
    - Type: Google Sheets Node (Update)  
    - Role: Updates existing rows in “Invoice Validation” Google Sheet with validation status, mismatch notes, and calculated price differences  
    - Config: Uses “Unique Key” to match rows and update columns including mismatch notes, loss/gain calculations, and matched cost/unit price  
    - Credentials: Same Google Sheets OAuth2 account  
    - Inputs: From Validation node  
    - Outputs: End of workflow  
    - Edge cases: Row not found, concurrent edits  

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                            | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                  |
|---------------------------|-------------------------------------|------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                     | Manual workflow start                     | None                             | Read/Write Files from Disk       |                                                                                              |
| Read/Write Files from Disk | File Reader                         | Reads local invoice PDF file              | When clicking ‘Test workflow’    | Extract from File                | ## Reading Invoice's PDF File Locally                                                        |
| Extract from File          | PDF Text Extractor                  | Extracts text from PDF                     | Read/Write Files from Disk       | Text Extractor                  | ## Extracting Details From Invoice PDF                                                      |
| OpenRouter Chat Model      | AI Language Model (OpenRouter)      | Parses raw text into structured JSON      | Text Extractor                  | Text Extractor                  |                                                                                              |
| Text Extractor             | Langchain Agent                    | Enforces JSON-only AI response prompt     | Extract from File / Send Raw Text Again | Post-Processing             |                                                                                              |
| Post-Processing            | Code                              | Cleans and parses AI JSON output           | Text Extractor                  | If                             | ## Processing Output                                                                        |
| If                        | Conditional                       | Checks JSON parsing success or failure    | Post-Processing                 | Send Raw Text Again / Split Out |                                                                                              |
| Send Raw Text Again        | Set                               | Retry extraction by resending raw text     | If (error branch)               | Text Extractor                 | ## Fallback On Error                                                                         |
| Split Out                 | Split Out                         | Splits line items into individual pieces  | If (success branch)             | Generate Unique Key             | ## Extracting Line Items                                                                     |
| Generate Unique Key        | Set                               | Generates unique keys for line items       | Split Out                      | Send Invoice Data              |                                                                                              |
| Send Invoice Data          | Google Sheets Append              | Appends invoice data to Google Sheet       | Generate Unique Key             | Fetch Master Data              | ## Sending Data To G-Sheet                                                                   |
| Fetch Master Data          | Google Sheets Read                | Fetches master SKU/pricing data            | Send Invoice Data              | Validation                    | ## Fetch Master Data & Validating Invoice Extracted Data                                     |
| Validation                | Code                              | Validates invoice items against master data | Fetch Master Data              | Update Results                |                                                                                              |
| Update Results             | Google Sheets Update             | Updates sheet with validation results      | Validation                    | None                         | ## Update Validation Result                                                                  |
| Sticky Note               | Sticky Note                      | Informational notes                        | None                          | None                         | Covers multiple nodes as per positions                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters needed  

2. **Add a File Read node**  
   - Type: `Read/Write Files from Disk`  
   - Name: `Read/Write Files from Disk`  
   - Configure file selector with your local invoice PDF path (e.g., `E:/SentIImenta AI/.../661.pdf`)  
   - Connect output from manual trigger  

3. **Add a PDF Text Extraction node**  
   - Type: `Extract from File`  
   - Name: `Extract from File`  
   - Operation: `pdf`  
   - Connect input from File Read node  

4. **Add an AI Chat Model node (OpenRouter)**  
   - Type: `OpenRouter Chat Model` (Langchain)  
   - Name: `OpenRouter Chat Model`  
   - Select model: `deepseek/deepseek-chat-v3-0324`  
   - Use OpenRouter API credentials (create OAuth2 or API key credentials in n8n)  
   - Connect input from `Text Extractor` node (will be created next)  

5. **Add a Langchain Agent node for Text Extraction**  
   - Type: `Text Extractor` (Langchain Agent)  
   - Name: `Text Extractor`  
   - Configure prompt with strict instructions to extract invoice fields and line items as valid JSON only (copy system message and prompt from workflow)  
   - Use expression `={{ $json.text }}` to pass extracted text from `Extract from File`  
   - Connect input from `Extract from File` node  
   - Connect AI language model input to `OpenRouter Chat Model` node  

6. **Add a Code node for Post-Processing**  
   - Type: `Code` (JavaScript)  
   - Name: `Post-Processing`  
   - Paste JavaScript code to clean AI output, remove markdown, extract JSON block, unescape, and parse JSON safely (see node code above)  
   - Connect input from `Text Extractor`  

7. **Add an If node for JSON parsing error handling**  
   - Type: `If`  
   - Name: `If`  
   - Condition: Check if `$json.error` equals “JSON parsing failed” OR “No valid JSON block found”  
   - Connect input from `Post-Processing`  

8. **Add a Set node to resend raw text on error**  
   - Type: `Set`  
   - Name: `Send Raw Text Again`  
   - Assign `text` field with expression `={{ $('Extract from File').item.json.text }}`  
   - Connect from `If` node error output (true)  
   - Connect output back to `Text Extractor` node to retry AI parsing  

9. **Add a Split Out node to split line items**  
   - Type: `Split Out`  
   - Name: `Split Out`  
   - Field to split: `line_items`  
   - Connect from `If` node success output (false)  

10. **Add a Set node to generate unique keys**  
    - Type: `Set`  
    - Name: `Generate Unique Key`  
    - Assign `Unique Key` with expression: `={{ $('If').item.json.invoice_number + "-" + $itemIndex }}`  
    - Connect from `Split Out`  

11. **Add a Google Sheets Append node to send invoice data**  
    - Type: `Google Sheets` (Append)  
    - Name: `Send Invoice Data`  
    - Connect from `Generate Unique Key`  
    - Configure with target Spreadsheet ID and Sheet Name (e.g., “Invoice Validation” Sheet1)  
    - Map columns for all invoice and line item fields (invoice number, PO number, dates, quantities, HSN, CGST, SGST, etc.)  
    - Use Google Sheets OAuth2 credentials  

12. **Add a Google Sheets Read node to fetch master data**  
    - Type: `Google Sheets` (Read)  
    - Name: `Fetch Master Data`  
    - Connect from `Send Invoice Data`  
    - Configure with Spreadsheet ID and Sheet Name containing master SKU/pricing data (e.g., “Mapping Sheet Final”, sheet “Cost-Priority sheet-GPT”)  
    - Use same Google Sheets credentials  

13. **Add a Code node for validation logic**  
    - Type: `Code` (JavaScript)  
    - Name: `Validation`  
    - Paste validation code implementing Levenshtein distance matching between invoice SKUs and master data, with ASIN and price checks (see node details above)  
    - Connect from `Fetch Master Data`  

14. **Add a Google Sheets Update node to write validation results**  
    - Type: `Google Sheets` (Update)  
    - Name: `Update Results`  
    - Connect from `Validation`  
    - Configure to update rows in the “Invoice Validation” sheet based on `Unique Key` match  
    - Map columns for mismatch notes, price difference, loss/gain, and matched cost/unit price  
    - Use same Google Sheets credentials  

15. **Add Sticky Notes for documentation** (optional)  
    - Add sticky notes near logical blocks for clarity as per original workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| The workflow uses OpenRouter API with model `deepseek/deepseek-chat-v3-0324` for invoice parsing.                                           | Custom AI model optimized for document parsing                                                              |
| The AI prompt enforces strict JSON-only output without markdown or wrappers.                                                               | Ensures downstream nodes receive parsable JSON                                                               |
| Levenshtein distance threshold is set to 5 for fuzzy SKU matching in validation.                                                           | Balances tolerance for minor text discrepancies with accuracy                                               |
| Validation checks both ASIN and cost equality to flag mismatches or confirm matches.                                                       | Critical for financial auditing and vendor price verification                                               |
| Google Sheets credentials must have appropriate access rights to all referenced spreadsheets and sheets.                                    | Prevents permission errors during append/read/update operations                                              |
| The workflow assumes local file access; adapt file read node for cloud storage or other sources as needed.                                  | Enhances portability and automation                                                                           |
| Sticky notes in the workflow provide contextual explanations of each block for maintainers and users.                                       | Improves workflow documentation and comprehension                                                            |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---