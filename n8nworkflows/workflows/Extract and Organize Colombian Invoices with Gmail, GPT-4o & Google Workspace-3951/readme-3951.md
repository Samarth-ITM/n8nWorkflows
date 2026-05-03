Extract and Organize Colombian Invoices with Gmail, GPT-4o & Google Workspace

https://n8nworkflows.xyz/workflows/extract-and-organize-colombian-invoices-with-gmail--gpt-4o---google-workspace-3951


# Extract and Organize Colombian Invoices with Gmail, GPT-4o & Google Workspace

### 1. Workflow Overview

This workflow automates the extraction, validation, and organization of **personal electronic invoices from Colombia** received by Gmail, specifically targeting `.zip` attachments containing invoices compliant with DIAN (Colombian tax authority) standards. The workflow processes the invoices by unzipping attachments, extracting key invoice data from PDF and XML files using AI, validating the data, and storing both the original documents and structured data in Google Drive and Google Sheets.

The workflow’s logic is grouped into the following functional blocks:

**1.1 Email Reception and Attachment Filtering**  
Automated polling of Gmail every 30 minutes to detect new emails with `.zip` attachments. Filtering is applied to process only these ZIP files.

**1.2 ZIP Extraction and Document Filtering**  
The ZIP attachments are decompressed, and the extracted files are filtered to isolate only the PDF and XML invoice documents relevant for further processing.

**1.3 Data Extraction from PDF and XML Files**  
The workflow extracts textual content from PDFs and XML data, converting XML to JSON. These data sources are merged and passed to a LangChain agent using OpenAI’s GPT-4o-mini to extract structured invoice details (e.g., invoice number, dates, NITs, amounts).

**1.4 Validation of Financial Data**  
A calculator tool verifies that the Total amount equals Subtotal plus IVA (tax), ensuring data integrity before storage.

**1.5 Storage and Record Management**  
Original PDF invoices are uploaded to Google Drive and renamed following a convention (`YYYY-MM-DD-NUMERO_FACTURA.pdf`). Extracted invoice data are inserted or updated in a Google Sheet, keyed uniquely by `NIT_Emisor + Numero_Factura` to prevent duplicates.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Reception and Attachment Filtering

- **Overview:**  
  This block triggers the workflow based on new Gmail messages containing `.zip` attachments, downloads attachments, and filters only ZIP files for processing.

- **Nodes Involved:**  
  - On Email receipt (Gmail Trigger)  
  - Get Filename and mimeType (Code)  
  - Filter ZIP files only (Filter)

- **Node Details:**

  - **On Email receipt**  
    - *Type:* Gmail Trigger  
    - *Role:* Poll Gmail every 30 minutes for emails with attachments named `.zip`.  
    - *Configuration:* Uses Gmail OAuth2 credentials; filter query `has:attachment filename:zip`; downloads attachments.  
    - *Input/Output:* Trigger node; outputs list of emails with attachments.  
    - *Edge cases:* Gmail API rate limits, OAuth token expiry, no matching emails, corrupted attachments.

  - **Get Filename and mimeType**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts filename and MIME type from binary attachment data to simplify filtering.  
    - *Key Code:* Iterates over binary attachments, returns JSON with `fileName` and `mimeType` plus binary data.  
    - *Input:* Items from Gmail trigger with binary attachments.  
    - *Output:* One item per attachment with filename and MIME type metadata.  
    - *Edge cases:* Missing binary data keys, unexpected attachment formats.

  - **Filter ZIP files only**  
    - *Type:* Filter  
    - *Role:* Passes only files with MIME type ending in "zip" for further processing.  
    - *Input:* Items from previous code node.  
    - *Output:* Only ZIP files forwarded.  
    - *Edge cases:* MIME type mismatch, incorrectly named files.

#### 2.2 ZIP Extraction and Document Filtering

- **Overview:**  
  Extracts all files from ZIPs and filters to retain only PDFs and XMLs for invoice data extraction.

- **Nodes Involved:**  
  - Unzip Invoice (Compression)  
  - Loop Over Items (SplitInBatches)  
  - Just for style (NoOp)  
  - Get filename and mimeType on extracted docs (Code)  
  - Split XML and PDF (Switch)

- **Node Details:**

  - **Unzip Invoice**  
    - *Type:* Compression (Unzip)  
    - *Role:* Extracts all files from the ZIP attachment binary.  
    - *Input:* ZIP binary files from filter node.  
    - *Output:* All extracted files in binary form.  
    - *Edge cases:* Corrupted ZIP files, very large archives.

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes each extracted file individually in sequence for manageable flow control.  
    - *Input:* Multiple extracted files.  
    - *Output:* Single file per execution cycle.  
    - *Edge cases:* Batch size defaults, handling empty batch.

  - **Just for style**  
    - *Type:* NoOp  
    - *Role:* Placeholder node, likely for visual clarity or future extension.  
    - *Input/Output:* Passes data unaltered.

  - **Get filename and mimeType on extracted docs**  
    - *Type:* Code  
    - *Role:* Extracts filename and MIME type from each extracted file’s binary data, preparing for filtering.  
    - *Same logic as previous Get Filename and mimeType node.*

  - **Split XML and PDF**  
    - *Type:* Switch  
    - *Role:* Routes extracted files into two outputs based on MIME type containing `pdf` or `xml`.  
    - *Outputs:* `pdf`, `xml`, or none (fallback).  
    - *Edge cases:* Unrecognized file types, mixed-case MIME types.

#### 2.3 Data Extraction from PDF and XML Files

- **Overview:**  
  Extracts textual content from PDFs and structured data from XMLs, merges them, and uses an AI agent (LangChain + OpenAI GPT-4o-mini) to parse required invoice details.

- **Nodes Involved:**  
  - Extract PDF Data (ExtractFromFile)  
  - Extract XML Data (ExtractFromFile)  
  - Convert to JSON (XML)  
  - Append both Docs (Merge)  
  - Aggregate all Data into 1 list (Aggregate)  
  - Extract Data from PDF and XML (LangChain Agent)  
  - OpenAI Chat Model (AI language model)  
  - Structured Output Parser (AI output parser)

- **Node Details:**

  - **Extract PDF Data**  
    - *Type:* ExtractFromFile (PDF)  
    - *Role:* Extracts all text from the PDF file (joins all pages).  
    - *Input:* PDF file binary.  
    - *Output:* JSON containing extracted text.  
    - *Edge cases:* Complex PDF layouts, encrypted PDFs.

  - **Extract XML Data**  
    - *Type:* ExtractFromFile (XML)  
    - *Role:* Parses XML file into structured data.  
    - *Input:* XML binary file.  
    - *Output:* XML data structure.  
    - *Edge cases:* Invalid XML, namespace issues.

  - **Convert to JSON**  
    - *Type:* XML node  
    - *Role:* Converts XML data into JSON for easier AI consumption.  
    - *Input:* Extracted XML data.  
    - *Output:* JSON object representing XML content.

  - **Append both Docs**  
    - *Type:* Merge  
    - *Role:* Combines PDF text and XML JSON into a single item with two data elements.  
    - *Input:* Outputs from PDF and XML extraction branches.  
    - *Output:* Single item containing both data sources.

  - **Aggregate all Data into 1 list**  
    - *Type:* Aggregate  
    - *Role:* Combines all individual items into a single list for batch AI processing.  
    - *Input:* Merged PDF+XML items.  
    - *Output:* Aggregated list.

  - **Extract Data from PDF and XML**  
    - *Type:* LangChain Agent (AI agent)  
    - *Role:* Uses OpenAI GPT-4o-mini with a system prompt instructing extraction of key invoice fields from the PDF text and XML description. Validates totals using Calculator tool.  
    - *Key Expressions:*  
      - Input text includes extracted PDF text and XML description.  
      - System message specifies required fields and validation logic.  
    - *Input:* Aggregated PDF+XML data list.  
    - *Output:* Parsed structured invoice data.  
    - *Edge cases:* AI misinterpretation, incomplete data, validation failures.

  - **OpenAI Chat Model**  
    - *Type:* LLM Chat model  
    - *Role:* Backend AI model used by the LangChain agent.  
    - *Configuration:* Model `gpt-4o-mini`, authenticated with OpenAI API credentials.  
    - *Edge cases:* API rate limits, authentication errors.

  - **Structured Output Parser**  
    - *Type:* Output parser for AI  
    - *Role:* Parses AI response into a strict JSON schema with expected invoice fields.  
    - *Schema Example:* Includes Tipo, Numero_Factura, Fecha_Emision, CUFE, NIT_Emisor, etc.  
    - *Edge cases:* Parsing errors, unexpected AI output.

#### 2.4 Validation of Financial Data

- **Overview:**  
  Validates that the Total amount equals Subtotal plus IVA using a calculator tool integrated into the LangChain agent workflow.

- **Nodes Involved:**  
  - Calculator (LangChain tool)

- **Node Details:**

  - **Calculator**  
    - *Type:* LangChain tool (Calculator)  
    - *Role:* Enables the AI agent to perform numeric validation (Total = Subtotal + IVA).  
    - *Input:* Called internally by LangChain agent during extraction step.  
    - *Edge cases:* Arithmetic precision issues, malformed inputs.

#### 2.5 Storage and Record Management

- **Overview:**  
  Saves the original PDF to Google Drive with a standardized filename and appends or updates the invoice data in a Google Sheet based on a unique key.

- **Nodes Involved:**  
  - Create initial PDF (Google Drive upload)  
  - Merge both flows (Merge)  
  - Update PDF with actual name (Google Drive update)  
  - Get Current Date (Code, unused)  
  - Create or update row (Google Sheets)  
  - Loop Over Items (SplitInBatches) [reused here]

- **Node Details:**

  - **Create initial PDF**  
    - *Type:* Google Drive  
    - *Role:* Uploads the original PDF file binary to a specified folder in Google Drive.  
    - *Configuration:* Uses OAuth2 credentials, folder ID fixed to user's "Facturas" folder. Initial filename from original attachment.  
    - *Input:* PDF binary from previous extraction branch.  
    - *Output:* Google Drive file metadata including file ID.

  - **Merge both flows**  
    - *Type:* Merge  
    - *Role:* Combines output from AI extracted data and Google Drive file upload for downstream processing.  
    - *Input:* Outputs from "Extract Data from PDF and XML" and "Create initial PDF".  
    - *Output:* Single combined item containing AI data and Drive file info.

  - **Update PDF with actual name**  
    - *Type:* Google Drive (Update)  
    - *Role:* Renames the uploaded PDF file to the format `YYYY-MM-DD-NUMERO_FACTURA.pdf` using extracted invoice data.  
    - *Input:* File ID from previous upload, new filename constructed via expressions.  
    - *Edge cases:* File locking issues, permission errors.

  - **Get Current Date**  
    - *Type:* Code  
    - *Role:* Generates current date in Colombia timezone (America/Bogota) in YYYY-MM-DD format.  
    - *Note:* Marked as not in use currently.  
    - *Edge cases:* Timezone discrepancies if used.

  - **Create or update row**  
    - *Type:* Google Sheets  
    - *Role:* Inserts or updates a row in Google Sheets with extracted invoice data.  
    - *Configuration:* Uses unique `Key` = `NIT_Emisor-Numero_Factura` to avoid duplicates, matches on sheet `gid=0` (Sheet1).  
    - *Fields Mapped:* Key, CUFE, Tipo, Fecha, Total, Factura, Impuesto, Subtotal, NIT Emisor, NIT Receptor, Razón Social, Resumen Compra.  
    - *Input:* Merged flow data.  
    - *Output:* Confirmation of sheet update.  
    - *Edge cases:* Google Sheets API limits, data type mismatches, concurrent updates.

  - **Loop Over Items** (reused here)  
    - *Role:* Processes each invoice record individually for Google Sheets update.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                                    | Input Node(s)                             | Output Node(s)                             | Sticky Note                                                                                                    |
|----------------------------------|----------------------------------|---------------------------------------------------|------------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Sticky Note                      | Sticky Note                      | Documentation and workflow summary                 | None                                     | None                                       | # 🧾 Colombian electronic invoices processing... (workflow summary and purpose)                                |
| On Email receipt                 | Gmail Trigger                   | Trigger workflow on new Gmail emails with ZIPs     | None                                     | Get Filename and mimeType                   | Executed every 30 minutes as it's for personal invoices, one can wait                                         |
| Get Filename and mimeType        | Code                            | Extract filename and MIME type from email attachments | On Email receipt                        | Filter ZIP files only                       |                                                                                                                |
| Filter ZIP files only            | Filter                          | Allow only ZIP files                                | Get Filename and mimeType                 | Unzip Invoice                              |                                                                                                                |
| Unzip Invoice                   | Compression (Unzip)             | Extract files from ZIP                              | Filter ZIP files only                     | Loop Over Items                            |                                                                                                                |
| Loop Over Items                 | SplitInBatches                  | Process each extracted file individually            | Unzip Invoice                            | Just for style, Get filename and mimeType on extracted docs |                                                                                                                |
| Just for style                  | NoOp                            | Visual placeholder                                  | Loop Over Items                          | Get filename and mimeType on extracted docs |                                                                                                                |
| Get filename and mimeType on extracted docs | Code                   | Extract filename and MIME type from extracted files | Loop Over Items                          | Split XML and PDF                          |                                                                                                                |
| Split XML and PDF              | Switch                         | Route files by MIME type to PDF or XML pipelines   | Get filename and mimeType on extracted docs | Create initial PDF, Extract PDF Data, Extract XML Data |                                                                                                                |
| Extract PDF Data               | ExtractFromFile (PDF)           | Extract text from PDF                               | Split XML and PDF (pdf output)            | Append both Docs                           |                                                                                                                |
| Extract XML Data               | ExtractFromFile (XML)           | Extract XML content                                | Split XML and PDF (xml output)            | Convert to JSON                            |                                                                                                                |
| Convert to JSON               | XML                            | Convert XML to JSON                                | Extract XML Data                         | Append both Docs                           |                                                                                                                |
| Append both Docs              | Merge                          | Combine PDF text and XML JSON data                  | Extract PDF Data, Convert to JSON         | Aggregate all Data into 1 list             |                                                                                                                |
| Aggregate all Data into 1 list | Aggregate                      | Aggregate all merged items into one list            | Append both Docs                        | Extract Data from PDF and XML              |                                                                                                                |
| Extract Data from PDF and XML  | LangChain Agent                | AI extraction of structured invoice data            | Aggregate all Data into 1 list           | Merge both flows                           |                                                                                                                |
| OpenAI Chat Model             | LangChain LLM Chat Model        | OpenAI GPT-4o-mini used by LangChain agent          | Extract Data from PDF and XML (ai_tool)  | Extract Data from PDF and XML (ai_languageModel) |                                                                                                                |
| Structured Output Parser      | LangChain Output Parser         | Parse AI output into structured JSON                 | OpenAI Chat Model                       | Extract Data from PDF and XML (ai_outputParser) |                                                                                                                |
| Create initial PDF            | Google Drive                   | Upload original PDF to Drive                         | Split XML and PDF (pdf output)            | Merge both flows                           |                                                                                                                |
| Merge both flows             | Merge                          | Combine AI data and Google Drive file info           | Extract Data from PDF and XML, Create initial PDF | Update PDF with actual name, Create or update row |                                                                                                                |
| Update PDF with actual name  | Google Drive (Update)          | Rename uploaded PDF file with invoice info           | Merge both flows                        | Get Current Date                           |                                                                                                                |
| Get Current Date             | Code                            | Generate current date in Colombia timezone (unused) | Update PDF with actual name              | Create or update row                       | Not in use actually...                                                                                          |
| Create or update row         | Google Sheets                  | Insert or update invoice data in Google Sheets       | Merge both flows                        | Loop Over Items                            |                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node ("On Email receipt")**  
   - Type: Gmail Trigger  
   - Credentials: Gmail OAuth2  
   - Parameters:  
     - Filter query: `has:attachment filename:zip`  
     - Poll every 30 minutes  
     - Download attachments enabled

2. **Add Code node ("Get Filename and mimeType")**  
   - Purpose: Extract filename and MIME type from attachments  
   - Code (JS): Iterate over binary attachments and output fileName and mimeType with binary data.

3. **Add Filter node ("Filter ZIP files only")**  
   - Condition: MIME type ends with `zip`  
   - Connect input from previous Code node

4. **Add Compression node ("Unzip Invoice")**  
   - Operation: Unzip  
   - Input: Filter node output (ZIP files)

5. **Add SplitInBatches node ("Loop Over Items")**  
   - Purpose: Process each extracted file individually  
   - Input: Unzip node output

6. **Add NoOp node ("Just for style")**  
   - Purpose: Visual placeholder  
   - Connect input from SplitInBatches node

7. **Add Code node ("Get filename and mimeType on extracted docs")**  
   - Same JS code as first Code node to extract metadata from extracted files  
   - Input from both "Just for style" and "Loop Over Items" nodes

8. **Add Switch node ("Split XML and PDF")**  
   - Conditions:  
     - Output `pdf` if MIME type contains `pdf`  
     - Output `xml` if MIME type contains `xml`  
   - Input: Previous Code node output

9. **Add ExtractFromFile node ("Extract PDF Data")**  
   - Operation: PDF text extraction (join pages)  
   - Input: Switch node `pdf` output

10. **Add ExtractFromFile node ("Extract XML Data")**  
    - Operation: XML extraction  
    - Input: Switch node `xml` output

11. **Add XML node ("Convert to JSON")**  
    - Converts XML output to JSON  
    - Input: Extract XML Data node output

12. **Add Merge node ("Append both Docs")**  
    - Mode: Merge by index (append)  
    - Inputs: Extract PDF Data and Convert to JSON nodes

13. **Add Aggregate node ("Aggregate all Data into 1 list")**  
    - Aggregate all item data into one list  
    - Input: Append both Docs output

14. **Add LangChain Agent node ("Extract Data from PDF and XML")**  
    - Model: OpenAI GPT-4o-mini (configured in next step)  
    - Input: Aggregated list  
    - Parameters:  
      - System message instructing extraction of invoice fields and validation  
      - Input prompt combining PDF text and XML description  
    - Enable output parser

15. **Add LangChain LLM Chat Model node ("OpenAI Chat Model")**  
    - Model: `gpt-4o-mini`  
    - Credentials: OpenAI API key  
    - Input: Connected to LangChain Agent node (ai_languageModel)

16. **Add LangChain Output Parser node ("Structured Output Parser")**  
    - JSON schema specifying extracted invoice fields  
    - Connected as AI output parser to LangChain Agent

17. **Add LangChain Calculator node ("Calculator")**  
    - For numeric validation inside LangChain agent (Total = Subtotal + IVA)

18. **Add Google Drive node ("Create initial PDF")**  
    - Operation: Upload file  
    - Credentials: Google Drive OAuth2  
    - Parameters:  
      - Drive: My Drive  
      - Folder ID: target folder for invoices  
      - File name: original file name from attachment  
      - Input: PDF file binary from Switch node `pdf` output

19. **Add Merge node ("Merge both flows")**  
    - Mode: Combine all  
    - Inputs: LangChain Agent output and Google Drive upload output

20. **Add Google Drive node ("Update PDF with actual name")**  
    - Operation: Update file metadata (rename)  
    - Credentials: Google Drive OAuth2  
    - Parameters:  
      - File ID: from Google Drive upload  
      - New file name: `{{Fecha_Emision}}-{{Numero_Factura}}.pdf` from parsed data

21. **Add Code node ("Get Current Date")** *(Optional, unused)*  
    - Generate current date in Colombia timezone (America/Bogota) in `YYYY-MM-DD`  
    - Not connected to main flow, can be omitted

22. **Add Google Sheets node ("Create or update row")**  
    - Operation: Append or update row  
    - Credentials: Google Sheets OAuth2  
    - Parameters:  
      - Document ID and Sheet Name (gid=0)  
      - Mapping columns to extracted fields: Key, CUFE, Tipo, Fecha, Total, Factura, Impuesto, Subtotal, NIT Emisor, NIT Receptor, Razón Social, Resumen Compra  
      - Unique key column: `Key` = `NIT_Emisor-Numero_Factura`

23. **Connect Loop Over Items node output to Google Sheets node**  
    - To process each invoice record individually for sheet update

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow is designed for personal use with minimal latency tolerance and prioritizes automation reliability.             | Workflow description                                                                                         |
| The workflow follows DIAN (Colombian tax authority) standards for electronic invoice handling, including invoice data fields. | Workflow description                                                                                         |
| For Google Drive and Google Sheets nodes, ensure OAuth2 credentials have proper scopes for file upload, rename, and edit.     | Credential setup requirement                                                                                 |
| OpenAI GPT-4o-mini is used as the language model for AI extraction, requiring a valid OpenAI API key with usage enabled.      | OpenAI Chat Model node setup                                                                                 |
| The AI agent uses LangChain with Calculator tool integration to validate arithmetic consistency in invoices.                  | Extract Data from PDF and XML node configuration                                                             |
| Gmail API quotas and OAuth refresh tokens must be managed to avoid interruptions in email polling.                            | On Email receipt node considerations                                                                         |
| The Google Sheets `Key` column uniquely identifies invoices to avoid duplicate entries using `NIT_Emisor + Numero_Factura`.   | Google Sheets node; prevents duplication                                                                     |
| Some nodes (e.g., "Get Current Date") are currently unused but may be activated for date-related functionalities if needed.  | Code node note                                                                                               |

---

This completes the detailed technical documentation and reference for reproducing and understanding the "Extract and Organize Colombian Invoices with Gmail, GPT-4o & Google Workspace" n8n workflow.