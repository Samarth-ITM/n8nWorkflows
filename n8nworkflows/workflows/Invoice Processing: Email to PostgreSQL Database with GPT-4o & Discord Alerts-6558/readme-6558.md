Invoice Processing: Email to PostgreSQL Database with GPT-4o & Discord Alerts

https://n8nworkflows.xyz/workflows/invoice-processing--email-to-postgresql-database-with-gpt-4o---discord-alerts-6558


# Invoice Processing: Email to PostgreSQL Database with GPT-4o & Discord Alerts

### 1. Workflow Overview

This workflow automates invoice processing by extracting invoice data from email attachments (PDFs), analyzing the content with an AI language model (GPT-4o), and storing verified invoice and company data into a PostgreSQL database. It also sends alerts summarizing new invoices to a Discord channel.

Logical blocks:

- **1.1 Input Reception**: Fetches emails and extracts PDF attachments.
- **1.2 PDF Text Extraction**: Converts PDF files to text for AI processing.
- **1.3 AI Processing & Parsing**: Uses GPT-4o to analyze text and extract structured invoice details.
- **1.4 Data Validation & Conditional Logic**: Filters and verifies extracted data, ensuring it is a valid invoice.
- **1.5 Database Operations**: Checks and inserts company and invoice data into PostgreSQL, handling duplicates.
- **1.6 Notification**: Sends invoice summary alerts to Discord.
- **1.7 Metadata Merging**: Combines email metadata with AI-extracted data for enriched records.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block reads incoming emails, filters for PDF attachments, and prepares those files for processing.

**Nodes Involved:**  
- Email Trigger (IMAP) [Disabled]  
- If  
- Code

**Node Details:**  

- **Email Trigger (IMAP)**  
  - Type: Email trigger node (IMAP)  
  - Role: Watches an email inbox for new messages with attachments (PDFs).  
  - Configuration: Downloads attachments; uses custom IMAP credentials.  
  - Input/Output: No input; outputs email data with attachments.  
  - Disabled: Yes (node is currently disabled).  
  - Edge cases: Authentication errors, no new emails, emails without PDFs.  
  - Sticky Note: Explains setup of IMAP credentials and filtering for PDFs.

- **If**  
  - Type: Conditional node  
  - Role: Checks if incoming email has binary attachments (PDFs).  
  - Key condition: Checks if `$binary` data is not empty.  
  - Input: Email Trigger output  
  - Output: True branch leads to Code node.  
  - Edge cases: Empty attachments, corrupted files.

- **Code**  
  - Type: Code node (JavaScript)  
  - Role: Filters attachments to only keep PDFs, restructuring data to one PDF per item.  
  - Key logic: Loops through binary attachments, checks MIME type `application/pdf`, returns new array of items with PDF files.  
  - Input: If node’s true output (emails with attachments)  
  - Output: Array of PDF files  
  - Edge cases: Attachments not PDF, files with incorrect or missing MIME types.

---

#### 1.2 PDF Text Extraction

**Overview:**  
Extracts text content from the filtered PDF files, preparing for AI analysis.

**Nodes Involved:**  
- If1  
- Extract from File1

**Node Details:**

- **If1**  
  - Type: Conditional node  
  - Role: Validates presence of binary data (PDF content) before extraction.  
  - Condition: Checks non-empty binary data.  
  - Input: Code node output (filtered PDFs)  
  - Output: True branch leads to Extract from File1.  
  - Edge cases: Missing or corrupted binary data.

- **Extract from File1**  
  - Type: ExtractText (PDF) node  
  - Role: Extracts text content from PDF binary data.  
  - Configuration: Keeps original binary data after extraction.  
  - Input: If1 true output (PDF files)  
  - Output: JSON with extracted text and original binary.  
  - Edge cases: PDFs with scanned images (no text), extraction errors.

---

#### 1.3 AI Processing & Parsing

**Overview:**  
Sends extracted text to an AI language model (GPT-4o) to determine if the document is an invoice and to extract structured invoice details.

**Nodes Involved:**  
- Basic LLM Chain  
- OpenAI Chat Model  
- Structured Output Parser  
- Merge

**Node Details:**

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain node  
  - Role: Sends prompt with extracted text to AI model; expects structured invoice extraction.  
  - Configuration: Custom prompt requesting invoice extraction with fields like invoice number, items, payment due date, bank details, totals, currency.  
  - Input: Extract from File1 (text extracted from PDF) and OpenAI Chat Model (AI inference).  
  - Output: AI raw response parsed by Structured Output Parser.  
  - Edge cases: AI response format errors, incomplete data extraction, language or format variations.  
  - Sticky Note: Advises prompt customization if invoices are not detected properly.

- **OpenAI Chat Model**  
  - Type: AI language model node (OpenAI GPT)  
  - Role: Executes AI inference using GPT-4o-mini model.  
  - Configuration: Model set to `gpt-4o-mini`, credentials required (OpenAI API key).  
  - Input: Receives prompt from Basic LLM Chain.  
  - Output: AI-generated text response.  
  - Edge cases: API key invalid, rate limits, network timeouts.  
  - Sticky Note: Instructions for setting up OpenAI credentials.

- **Structured Output Parser**  
  - Type: LangChain output parser node  
  - Role: Parses AI text response into structured JSON following a defined schema.  
  - Configuration: JSON schema example provided for invoice details.  
  - Input: AI output from Basic LLM Chain.  
  - Output: Structured JSON with invoice details.  
  - Edge cases: Parsing failures if AI response deviates from schema.

- **Merge**  
  - Type: Merge node  
  - Role: Combines AI invoice extraction output and original email data (metadata).  
  - Input 1: AI output  
  - Input 2: Email metadata (not shown directly in connections but implied)  
  - Configuration: Combine by position/index.  
  - Output: Combined JSON data for further processing.  
  - Sticky Note: Explains usage of merge node for metadata enrichment.

---

#### 1.4 Data Validation & Conditional Logic

**Overview:**  
Filters out non-invoices and prepares valid invoice data for database insertion.

**Nodes Involved:**  
- If2  
- Discord  
- Prepare Company Data

**Node Details:**

- **If2**  
  - Type: Conditional node  
  - Role: Checks if AI detected document is an invoice (`$json.output.isInvoice == true`).  
  - Input: Merge node output (combined data)  
  - Output: True branch triggers Discord notification and company data preparation.  
  - Edge cases: False negatives where real invoices are missed.

- **Discord**  
  - Type: Discord node (Webhook)  
  - Role: Sends a formatted alert to Discord summarizing invoice details.  
  - Configuration: Uses webhook URL; message content template includes invoice number, buyer/seller details, bank, due date, total amount, currency.  
  - Input: If2 true branch output (invoice data)  
  - Output: Notification sent to Discord channel.  
  - Edge cases: Webhook URL misconfiguration, Discord API rate limits.  
  - Sticky Note: Setup instructions for Discord webhook.

- **Prepare Company Data**  
  - Type: Set node  
  - Role: Extracts and formats company issuer data (tax number, name, address) for database queries.  
  - Input: If2 true output (invoice data)  
  - Output: JSON with prepared company fields for PostgreSQL queries.

---

#### 1.5 Database Operations

**Overview:**  
Checks for existing company and invoice records; inserts new ones as needed.

**Nodes Involved:**  
- Check Company  
- IF Company Does Not Exist  
- Add Company  
- Set New Company ID  
- Set Existing Company ID  
- Merge Company ID  
- Check Invoice  
- IF Invoice Does Not Exist  
- Add Invoice  
- Invoice Exists  
- Merge End

**Node Details:**

- **Check Company**  
  - Type: PostgreSQL node  
  - Role: Queries company table to find a company by tax number.  
  - Query: `SELECT id FROM company WHERE tax_number = '{{ $json.issuer_tax_number }}';`  
  - Input: Prepare Company Data output  
  - Output: Query result with company id if found.  
  - Edge cases: DB connection failure, missing tax number.

- **IF Company Does Not Exist**  
  - Type: Conditional node  
  - Role: Checks if company query returned no results (empty array).  
  - Output True: Add Company node (new company insertion)  
  - Output False: Set Existing Company ID node  
  - Edge cases: Unexpected query response format.

- **Add Company**  
  - Type: PostgreSQL insert node  
  - Role: Inserts new company record into `company` table with extracted issuer data.  
  - Input: IF Company Does Not Exist true branch  
  - Output: New company record JSON with assigned `id`.  
  - Edge cases: DB insert errors, constraint violations.  
  - Sticky Note: PostgreSQL credential reminder.

- **Set New Company ID**  
  - Type: Set node  
  - Role: Sets `company_id` field with newly inserted company id (`$json.id`).  
  - Input: Add Company output  
  - Output: JSON with `company_id` for invoice insertion.

- **Set Existing Company ID**  
  - Type: Set node  
  - Role: Sets `company_id` from existing company query result.  
  - Input: IF Company Does Not Exist false branch  
  - Output: JSON with `company_id`.

- **Merge Company ID**  
  - Type: Merge node  
  - Role: Merges outputs from Set New Company ID and Set Existing Company ID into one stream for invoice check.  
  - Input: From both Set nodes  
  - Output: Combined JSON with `company_id`.

- **Check Invoice**  
  - Type: PostgreSQL node  
  - Role: Queries invoice table to check if invoice number already exists for the company.  
  - Query: `SELECT id FROM invoice WHERE invoice_number = '{{ $json.invoiceDetails.invoiceNumber }}' AND company_id = {{ $json.company_id }};`  
  - Input: Merge Company ID output  
  - Output: Query result with invoice id if found.  
  - Edge cases: DB connectivity, missing invoice number.

- **IF Invoice Does Not Exist**  
  - Type: Conditional node  
  - Role: Checks if invoice query returned no results (length=0).  
  - Output True: Add Invoice node  
  - Output False: Invoice Exists node

- **Add Invoice**  
  - Type: PostgreSQL insert node  
  - Role: Inserts new invoice record into `invoice` table using extracted invoice details and `company_id`.  
  - Input: IF Invoice Does Not Exist true branch  
  - Output: Inserted invoice record.

- **Invoice Exists**  
  - Type: Set node  
  - Role: Sets a status message `"Invoice already exists"` for duplicate detection handling.  
  - Input: IF Invoice Does Not Exist false branch  
  - Output: Status message JSON.

- **Merge End**  
  - Type: Merge node  
  - Role: Combines both Add Invoice and Invoice Exists outputs as workflow end point.  
  - Input: From Add Invoice and Invoice Exists

---

#### 1.6 Notification

This block is integrated within 1.4 (If2 node triggering Discord).

See node **Discord** in section 1.4.

---

#### 1.7 Metadata Merging

See node **Merge** in section 1.3.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                                 | Input Node(s)               | Output Node(s)                       | Sticky Note                                                                                      |
|-------------------------|--------------------------------|------------------------------------------------|-----------------------------|------------------------------------|-------------------------------------------------------------------------------------------------|
| Email Trigger (IMAP)     | emailReadImap                   | Watches inbox, fetches emails with attachments | None                        | If                                 | Explains IMAP credentials setup and filtering for PDFs                                         |
| If                      | if                             | Checks if attachments exist                     | Email Trigger (IMAP)         | Code                               |                                                                                                 |
| Code                    | code                           | Filters PDF files from attachments              | If                          | If1                                |                                                                                                 |
| If1                     | if                             | Checks if PDF binary data is present             | Code                        | Extract from File1                  |                                                                                                 |
| Extract from File1       | extractFromFile                | Extracts text content from PDFs                   | If1                         | Basic LLM Chain, Merge              |                                                                                                 |
| Basic LLM Chain          | langchain.chainLlm              | Sends text to AI for invoice extraction          | Extract from File1, OpenAI Chat Model | Merge                       | Advises prompt customization for improved invoice detection                                    |
| OpenAI Chat Model        | langchain.lmChatOpenAi          | Executes GPT-4o inference                         | Basic LLM Chain              | Basic LLM Chain                    | Describes OpenAI API key setup and model selection                                             |
| Structured Output Parser | langchain.outputParserStructured| Parses AI output into structured JSON             | Basic LLM Chain              | Basic LLM Chain                    |                                                                                                 |
| Merge                   | merge                          | Combines AI output with email metadata            | Basic LLM Chain, Extract from File1 | If2                           | Explains merging invoice data with email metadata                                              |
| If2                     | if                             | Validates if data is a confirmed invoice          | Merge                       | Discord, Prepare Company Data      |                                                                                                 |
| Discord                 | discord                        | Sends invoice summary notification to Discord    | If2                         | None                             | Explains Discord webhook setup                                                                 |
| Prepare Company Data     | set                            | Prepares company info fields for DB queries       | If2                         | Check Company                     |                                                                                                 |
| Check Company            | postgres                       | Checks if company exists in DB                      | Prepare Company Data         | IF Company Does Not Exist          | PostgreSQL credential setup reminder                                                           |
| IF Company Does Not Exist| if                             | Branches logic based on company existence          | Check Company                | Add Company, Set Existing Company ID |                                                                                                 |
| Add Company              | postgres                       | Inserts new company record                          | IF Company Does Not Exist    | Set New Company ID                 | PostgreSQL credential setup reminder                                                           |
| Set New Company ID       | set                            | Sets company_id from newly inserted company        | Add Company                 | Merge Company ID                   |                                                                                                 |
| Set Existing Company ID  | set                            | Sets company_id from existing company               | IF Company Does Not Exist    | Merge Company ID                   |                                                                                                 |
| Merge Company ID         | merge                          | Merges new and existing company_id branches        | Set New Company ID, Set Existing Company ID | Check Invoice             |                                                                                                 |
| Check Invoice            | postgres                       | Checks if invoice exists for company                 | Merge Company ID             | IF Invoice Does Not Exist          | PostgreSQL credential setup reminder                                                           |
| IF Invoice Does Not Exist| if                             | Branches logic based on invoice existence           | Check Invoice                | Add Invoice, Invoice Exists        |                                                                                                 |
| Add Invoice              | postgres                       | Inserts new invoice record                           | IF Invoice Does Not Exist    | Merge End                        | PostgreSQL credential setup reminder                                                           |
| Invoice Exists           | set                            | Sets status message if invoice already exists        | IF Invoice Does Not Exist    | Merge End                        |                                                                                                 |
| Merge End                | merge                          | Combines invoice insert and duplicate status outcomes| Add Invoice, Invoice Exists  | None                             |                                                                                                 |
| Sticky Note              | stickyNote                     | Various instructional notes                         | None                        | None                             | See individual sticky notes for context                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Email Trigger (IMAP) Node**  
   - Type: `EmailReadImap`  
   - Configuration: Set IMAP credentials with host, port, username, password. Enable attachment download.  
   - Trigger: Receive new emails with attachments.

2. **Add If Node (Check Attachments)**  
   - Type: `If`  
   - Condition: Check if `$binary` is not empty (attachments exist).  
   - Connect Email Trigger output to this node.

3. **Add Code Node (Filter PDFs)**  
   - Type: `Code`  
   - Paste JS code filtering attachments by MIME type `application/pdf`, creating separate items per PDF.  
   - Connect If node’s true output here.

4. **Add If1 Node (Check PDF Binary Data)**  
   - Type: `If`  
   - Condition: Check if `$binary` is not empty.  
   - Connect Code node output.

5. **Add Extract from File Node**  
   - Type: `ExtractFromFile`  
   - Operation: PDF text extraction. Enable "keep source" to retain binary.  
   - Connect If1 true output.

6. **Add OpenAI Chat Model Node**  
   - Type: `langchain.lmChatOpenAi`  
   - Credentials: Configure OpenAI API key.  
   - Model: Select `gpt-4o-mini`.  
   - Connect to Basic LLM Chain node’s AI language model input.

7. **Add Basic LLM Chain Node**  
   - Type: `langchain.chainLlm`  
   - Parameters: Set prompt explaining invoice extraction requirements (invoice number, items, payment date, bank, totals, currency).  
   - Connect Extract from File output and OpenAI Chat Model node output as inputs.

8. **Add Structured Output Parser Node**  
   - Type: `langchain.outputParserStructured`  
   - Provide JSON schema example matching expected invoice output format.  
   - Connect AI output from Basic LLM Chain to this parser.

9. **Add Merge Node (Invoice Data + Email Metadata)**  
   - Type: `Merge`  
   - Merge strategy: "Combine by Position"  
   - Connect Structured Output Parser output and email metadata source (if available).

10. **Add If2 Node (Check if Data is Invoice)**  
    - Type: `If`  
    - Condition: `$json.output.isInvoice` equals `true`.  
    - Connect Merge node output.

11. **Add Discord Node**  
    - Type: `discord`  
    - Configure Discord webhook URL.  
    - Message content: Template with invoice summary fields.  
    - Connect If2 true output.

12. **Add Set Node (Prepare Company Data)**  
    - Type: `Set`  
    - Set variables: `issuer_tax_number`, `issuer_name`, `issuer_address` from invoiceDetails.  
    - Connect If2 true output.

13. **Add Check Company Node**  
    - Type: `Postgres` (Execute Query)  
    - Query: `SELECT id FROM company WHERE tax_number = '{{ $json.issuer_tax_number }}';`  
    - Configure PostgreSQL credentials.  
    - Connect Prepare Company Data output.

14. **Add IF Company Does Not Exist Node**  
    - Type: `If`  
    - Condition: Check if query result length is 0 (no company found).  
    - Connect Check Company output.

15. **Add Add Company Node**  
    - Type: `Postgres` (Insert)  
    - Table: company  
    - Map issuer data fields to columns.  
    - Connect IF Company Does Not Exist true branch.

16. **Add Set New Company ID Node**  
    - Type: `Set`  
    - Set `company_id` from inserted company record’s `id`.  
    - Connect Add Company output.

17. **Add Set Existing Company ID Node**  
    - Type: `Set`  
    - Set `company_id` from Check Company query result.  
    - Connect IF Company Does Not Exist false branch.

18. **Add Merge Company ID Node**  
    - Type: `Merge`  
    - Merge Set New Company ID and Set Existing Company ID outputs.  
    - Connect both Set nodes.

19. **Add Check Invoice Node**  
    - Type: `Postgres` (Execute Query)  
    - Query: `SELECT id FROM invoice WHERE invoice_number = '{{ $json.invoiceDetails.invoiceNumber }}' AND company_id = {{ $json.company_id }};`  
    - Connect Merge Company ID output.

20. **Add IF Invoice Does Not Exist Node**  
    - Type: `If`  
    - Condition: Query result length equals 0.  
    - Connect Check Invoice output.

21. **Add Add Invoice Node**  
    - Type: `Postgres` (Insert)  
    - Table: invoice  
    - Map invoice details fields and `company_id`.  
    - Connect IF Invoice Does Not Exist true branch.

22. **Add Invoice Exists Node**  
    - Type: `Set`  
    - Set field `status` to `"Invoice already exists"`.  
    - Connect IF Invoice Does Not Exist false branch.

23. **Add Merge End Node**  
    - Type: `Merge`  
    - Merge Add Invoice and Invoice Exists outputs.  
    - Connect both nodes.

24. **Verify all PostgreSQL nodes use the same valid credentials with access to required tables.**

25. **Optionally enable and configure Email Trigger (IMAP) to activate workflow on new incoming emails.**

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| 📥 Email Trigger (IMAP): Create IMAP credentials with host, port, email, and password. Filter emails to those with PDFs. | Sticky Note near Email Trigger node                            |
| 🤖 AI Data Extraction: Use clear prompt specifying all invoice fields for reliable extraction. Customize prompt as needed. | Sticky Note near Basic LLM Chain node                          |
| 🔑 OpenAI API Setup: Generate API key on https://platform.openai.com, create n8n OpenAI credentials, select appropriate model. | Sticky Note near OpenAI Chat Model                             |
| 📤 Discord Webhook: Create webhook in Discord Server Settings → Integrations; paste URL in Discord node.        | Sticky Note near Discord node                                  |
| 🗃️ PostgreSQL Insert: Ensure correct DB schema and permissions; map all invoice/company fields properly.        | Sticky Note near PostgreSQL nodes                              |
| ❗ PostgreSQL Credentials: Select valid credentials in all PostgreSQL nodes to avoid workflow failures.          | Sticky Note near PostgreSQL nodes                              |
| 🔀 Merge Node: Used to attach metadata (e.g., email sender) to AI-extracted data for enriched records.          | Sticky Note near Merge node                                    |

---

**Disclaimer:**  
The text provided is exclusively derived from a workflow automated with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.