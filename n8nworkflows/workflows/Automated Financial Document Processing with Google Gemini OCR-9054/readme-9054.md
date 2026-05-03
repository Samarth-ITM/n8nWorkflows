Automated Financial Document Processing with Google Gemini OCR

https://n8nworkflows.xyz/workflows/automated-financial-document-processing-with-google-gemini-ocr-9054


# Automated Financial Document Processing with Google Gemini OCR

---

## 1. Workflow Overview

This workflow automates the processing of financial documents using Google Gemini OCR and AI-driven natural language models, integrating with Google Drive and Google Sheets. It is designed for organizations needing to automatically extract, categorize, and store financial data from multiple document types such as invoices, expense receipts, and bank statements. Additionally, it provides an AI-powered financial analysis agent for querying the aggregated data.

### Logical Blocks:

- **1.1 Invoice Processing (Mark - Accountant)**  
  Handles invoice uploads via chat interface, extracts structured invoice data using Google Gemini, saves data to Google Sheets, renames files, and confirms completion.

- **1.2 Expense Processing (Donna - Accountant)**  
  Monitors a Google Drive folder for new expense receipts, downloads files, extracts expense data with Google Gemini, categorizes expenses using OpenRouter LLM, saves to Google Sheets.

- **1.3 Bank Statement Processing (Victor - Controller)**  
  Monitors a Google Drive folder for new bank statements, downloads files, extracts transaction data with Google Gemini, parses multiple transactions via JavaScript, saves to Google Sheets.

- **1.4 Financial Analysis Agent (Andrew - CFO)**  
  Manual trigger to query and analyze financial data across invoices, expenses, and transactions using an AI agent connected to all relevant Google Sheets.

- **1.5 Supporting Nodes**  
  Includes triggers, file storage, data parsing, error handling, and UI sticky notes for documentation and instructions.

---

## 2. Block-by-Block Analysis

### 1.1 Invoice Processing (Mark - Accountant)

**Overview:**  
Manages invoice documents uploaded by users via a chat interface. The workflow saves the invoice to Google Drive, sends it to Google Gemini for invoice-specific OCR and data extraction, parses the JSON response, splits line items, saves the data to a Google Sheet, renames the file, and sends a confirmation message.

**Nodes Involved:**  
- When chat message received  
- Save Invoice  
- Upload PDF to Google Gemini  
- Download Data from Google Gemini  
- Parsed Data  
- Split Out  
- Table ERP (Google Sheets)  
- Merge  
- Update File Name  
- Confirmation  
- Multiple Sticky Notes (instructions & documentation)

**Node Details:**

- **When chat message received**  
  - Type: Langchain chat trigger  
  - Role: Entry trigger accepting chat messages with file uploads (any MIME type)  
  - Config: Allows file uploads, no MIME restriction but expects PDFs  
  - Outputs: Passes uploaded file binary data and metadata

- **Save Invoice**  
  - Type: Google Drive node  
  - Role: Saves uploaded invoice PDF to "Invoices" Google Drive folder  
  - Config: Uses binary data from chat node, original filename, folder ID specified  
  - Outputs: File metadata including file ID for later use

- **Upload PDF to Google Gemini**  
  - Type: HTTP Request  
  - Role: Uploads PDF binary data to Google Gemini File Upload API  
  - Config: POST multipart with Content-Type: application/pdf, uses Google PaLM API credentials  
  - Input: Binary PDF from chat upload  
  - Output: File URI for subsequent AI processing

- **Download Data from Google Gemini**  
  - Type: HTTP Request  
  - Role: Sends prompt to Gemini AI to extract invoice fields or summarize if not invoice  
  - Config: POST JSON with prompt describing required fields, includes uploaded file URI  
  - Output: AI response with extracted data or summary text

- **Parsed Data**  
  - Type: Set node  
  - Role: Cleans AI response text removing markdown code blocks, parses JSON to object  
  - Config: Uses expression to strip markdown and parse JSON from AI content  
  - Features: Retry on failure for robustness

- **Split Out**  
  - Type: Split Out  
  - Role: Splits "Line Items" array from parsed invoice data into individual items for row-wise processing

- **Table ERP (Google Sheets)**  
  - Type: Google Sheets append operation  
  - Role: Appends invoice line items and header data to "Test Invoice Records" spreadsheet  
  - Mapping: Maps parsed invoice fields and individual line item details to columns

- **Merge**  
  - Type: Merge  
  - Role: Combines original file metadata with spreadsheet data for further processing

- **Update File Name**  
  - Type: Google Drive update  
  - Role: Renames saved invoice file using pattern "{Vendor Name} - {Invoice Number}"  
  - Error Handling: Continues gracefully if rename fails

- **Confirmation**  
  - Type: Set node  
  - Role: Sets confirmation message "All invoices have been processed" for user feedback

- **Sticky Notes:**  
  Provide detailed instructions on each step, role descriptions, AI prompt details, and configuration guidelines.

**Edge Cases & Failure Points:**  
- Upload failures due to network/API issues  
- AI returning invalid or non-JSON output (handled with retry and error messages)  
- Missing or malformed "Line Items" array  
- Google Sheets append errors (e.g., schema mismatches)  
- File renaming failures (non-critical)  

---

### 1.2 Expense Processing (Donna - Accountant)

**Overview:**  
Monitors the "Expense Receipts" Google Drive folder for new PDF files every minute. Downloads each file, sends it to Google Gemini for expense data extraction, uses OpenRouter LLM for automatic categorization into 17 predefined categories, saves the structured data to the "Expenses Recording" Google Sheet, and confirms processing.

**Nodes Involved:**  
- Google Drive Trigger (Expense Receipts folder)  
- Google Drive (download)  
- Upload PDF to Google Gemini1  
- Download Data from Google Gemini1  
- Parsed Data1  
- Split Out1  
- Basic LLM Chain  
- OpenRouter Chat Model  
- Structured Output Parser  
- Google Sheets  
- Edit Fields1  
- Sticky Notes (instructions & documentation)

**Node Details:**

- **Google Drive Trigger**  
  - Type: Google Drive trigger  
  - Role: Watches "Expense Receipts" folder for newly created files every minute  
  - Output: Metadata of new expense receipt files

- **Google Drive (download)**  
  - Type: Google Drive file download  
  - Role: Downloads binary PDF data for processing  
  - Input: File ID from trigger

- **Upload PDF to Google Gemini1**  
  - Same as invoice upload but for expense receipts

- **Download Data from Google Gemini1**  
  - Sends prompt tailored to expense receipts, requesting fields such as Merchant Name, Transaction Date, Payment Method, Tax Amount, and line items if available

- **Parsed Data1**  
  - Cleans and parses AI JSON response as in invoice block

- **Split Out1**  
  - Splits line items array for row-wise processing

- **Basic LLM Chain**  
  - Uses OpenRouter LLM to classify each expense line item into one of 17 predefined categories  
  - Includes a structured output parser to enforce JSON response

- **Google Sheets**  
  - Appends categorized expense data to "Expenses Recording" spreadsheet with mapped columns

- **Edit Fields1**  
  - Sets a confirmation message "Recorded!" after successful save

- **Sticky Notes:**  
  Document workflow steps, AI categorization details, and expense categories list.

**Edge Cases & Failure Points:**  
- File download or Google Drive trigger latency  
- AI misclassification or invalid JSON response (structured output parser mitigates)  
- Missing line items or partial data  
- Google Sheets append errors  
- Expense category ambiguity handled by LLM fallback

---

### 1.3 Bank Statement Processing (Victor - Controller)

**Overview:**  
Monitors the "Bank Statements" Google Drive folder for new PDF bank statements. Downloads each file, sends to Google Gemini to extract multiple transaction entries, parses and normalizes the transaction JSON via custom JavaScript code, appends transactions to "Bank Transactions Record" Google Sheet, and confirms processing.

**Nodes Involved:**  
- Google Drive Trigger1 (Bank Statements folder)  
- Google Drive1 (download)  
- Upload PDF to Google Gemini2  
- Download Data from Google Gemini2  
- Code (custom JavaScript parser)  
- Google Sheets1  
- Edit Fields  
- Sticky Notes (instructions & documentation)

**Node Details:**

- **Google Drive Trigger1**  
  - Watches "Bank Statements" folder every minute for new files

- **Google Drive1**  
  - Downloads PDF binary data

- **Upload PDF to Google Gemini2**  
  - Uploads bank statement PDF to Google Gemini

- **Download Data from Google Gemini2**  
  - Sends prompt to extract bank statement fields including multiple transactions

- **Code**  
  - JavaScript node parses AI response text to extract clean JSON array of transactions  
  - Handles multiple response formats, cleans invalid characters, and throws readable errors if parsing fails

- **Google Sheets1**  
  - Appends each transaction with mapped columns to "Bank Transactions Record" spreadsheet

- **Edit Fields**  
  - Sets confirmation message "Recorded!" after data appended

- **Sticky Notes:**  
  Detail the parsing logic, responsibilities of the Victor persona, and bank statement specifics.

**Edge Cases & Failure Points:**  
- AI returning complex or malformed JSON requiring robust parsing  
- Large bank statements exceeding token limits (maxOutputTokens set high)  
- Google Sheets append errors  
- File download or trigger delays

---

### 1.4 Financial Analysis Agent (Andrew - CFO)

**Overview:**  
A manual trigger workflow that uses a Langchain AI agent configured as a CFO persona. It connects to the three Google Sheets (Invoices, Expenses, Transactions) as tools and can answer complex natural language queries about financial data, generate reports, and provide insights.

**Nodes Involved:**  
- When Executed by Another Workflow (disabled by default)  
- AI Agent (Langchain agent)  
- Invoices (Google Sheets Tool)  
- Expenses (Google Sheets Tool)  
- Transactions (Google Sheets Tool)  
- Edit Fields2  
- OpenRouter Chat Model1  
- Sticky Notes (instructions & documentation)

**Node Details:**

- **When Executed by Another Workflow**  
  - Manual trigger node, disabled by default for security

- **AI Agent**  
  - Langchain agent configured with a system prompt defining its role as FinanceDataBot  
  - Connected to three Google Sheets tools for data access  
  - Processes natural language queries, returns formatted data and analysis

- **Invoices, Expenses, Transactions**  
  - Google Sheets Tool nodes that provide AI agent access to respective financial spreadsheets

- **Edit Fields2**  
  - Returns AI agent response as "Response" string

- **OpenRouter Chat Model1**  
  - Language model node used by AI agent for processing queries

- **Sticky Notes:**  
  Provide detailed role description, usage instructions, and query examples.

**Edge Cases & Failure Points:**  
- Query ambiguity leading to agent requesting clarification  
- Large or complex queries may time out or exceed token limits  
- Access permissions to Google Sheets must be correctly configured  
- Disabled trigger requires manual enablement to function

---

### 1.5 Supporting Nodes and Documentation

- Multiple **Sticky Note** nodes provide extensive in-workflow documentation, instructions, role descriptions, setup requirements, usage guidelines, and AI persona explanations.

- Sticky notes cover: Workflow overview, setup instructions, AI prompt details, error handling guidance, category lists, and visualization of team roles.

- Credentials used:  
  - Google Drive OAuth2  
  - Google Sheets OAuth2  
  - Google PaLM API (Google Gemini)  
  - OpenRouter API (for LLM categorization and agent chat)

---

## 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                               | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                         |
|-------------------------------|----------------------------------|-----------------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------|
| When chat message received     | Langchain chat trigger           | Entry trigger for invoice upload via chat     | -                                | Upload PDF to Google Gemini, Save Invoice | # Mark - Accountant (role info, image)                                                               |
| Save Invoice                  | Google Drive                     | Saves uploaded invoice PDF to Drive folder    | When chat message received        | Merge                           | # 💾 Save File Instructions (upload & storage details)                                            |
| Upload PDF to Google Gemini    | HTTP Request                    | Uploads PDF to Google Gemini AI                | When chat message received / Google Drive | Download Data from Google Gemini | # 🤖 Upload to AI Instructions (upload config)                                                     |
| Download Data from Google Gemini| HTTP Request                   | Requests data extraction from Google Gemini AI | Upload PDF to Google Gemini       | Parsed Data, Upload PDF to Google Gemini | # 🧠 Data Extraction Instructions (prompt details)                                                 |
| Parsed Data                   | Set                              | Cleans and parses AI JSON response              | Download Data from Google Gemini  | Split Out                      | # 📋 JSON Parsing Instructions (clean and parse JSON)                                             |
| Split Out                    | Split Out                        | Splits invoice line items array                  | Parsed Data                      | Table ERP                     | # 📑 Split Items Instructions (line item processing)                                              |
| Table ERP                    | Google Sheets                   | Appends invoice data to Google Sheet            | Split Out                       | Merge                         | # 📊 Save to Sheet Instructions (mapping invoice fields)                                          |
| Merge                        | Merge                            | Combines file metadata with invoice data        | Save Invoice, Table ERP          | Update File Name               | # 🔗 Merge Instructions (combine data)                                                           |
| Update File Name             | Google Drive                    | Renames invoice file with vendor & invoice no. | Merge                          | Confirmation                  | # 📝 Rename Instructions (file renaming details)                                                 |
| Confirmation                | Set                              | Final confirmation message for user             | Update File Name                | -                             | # ✅ Confirmation Instructions (completion message)                                              |
| Google Drive Trigger         | Google Drive Trigger             | Watches "Expense Receipts" folder                | -                              | Google Drive                  | # 📁 Expense Trigger (folder monitoring details)                                                  |
| Google Drive                | Google Drive                    | Downloads expense receipt PDF                     | Google Drive Trigger            | Upload PDF to Google Gemini1  |                                                                                                  |
| Upload PDF to Google Gemini1 | HTTP Request                    | Uploads expense PDF to Google Gemini             | Google Drive                   | Download Data from Google Gemini1 |                                                                                                  |
| Download Data from Google Gemini1| HTTP Request               | Requests expense data extraction                  | Upload PDF to Google Gemini1   | Parsed Data1                 |                                                                                                  |
| Parsed Data1                | Set                              | Cleans and parses expense AI JSON response       | Download Data from Google Gemini1| Split Out1                  |                                                                                                  |
| Split Out1                  | Split Out                        | Splits expense line items array                   | Parsed Data1                   | Basic LLM Chain             |                                                                                                  |
| Basic LLM Chain             | Langchain LLM Chain             | Categorizes expense line items into 17 categories| Split Out1                    | Google Sheets               | # 🏷️ Categorization Instructions (AI categorization)                                             |
| Google Sheets               | Google Sheets                   | Appends categorized expenses to Google Sheet     | Basic LLM Chain               | Edit Fields1                |                                                                                                  |
| Edit Fields1               | Set                              | Sets confirmation message after saving           | Google Sheets                 | -                           |                                                                                                  |
| Google Drive Trigger1       | Google Drive Trigger             | Watches "Bank Statements" folder                  | -                            | Google Drive1               | # 🏦 Bank Statement Processing Documentation (folder monitoring)                                 |
| Google Drive1              | Google Drive                    | Downloads bank statement PDF                       | Google Drive Trigger1          | Upload PDF to Google Gemini2 |                                                                                                  |
| Upload PDF to Google Gemini2 | HTTP Request                    | Uploads bank statement PDF                        | Google Drive1                 | Download Data from Google Gemini2 |                                                                                                  |
| Download Data from Google Gemini2| HTTP Request               | Requests bank statement data extraction            | Upload PDF to Google Gemini2  | Code                       |                                                                                                  |
| Code                       | Code                             | Parses AI JSON response into transaction array     | Download Data from Google Gemini2| Google Sheets1             | # 🔄 Bank Parser Instructions (custom parsing code)                                              |
| Google Sheets1             | Google Sheets                   | Appends bank transactions to Google Sheet          | Code                        | Edit Fields                 |                                                                                                  |
| Edit Fields                | Set                              | Confirmation message after saving                   | Google Sheets1               | -                           |                                                                                                  |
| When Executed by Another Workflow | Execute Workflow Trigger     | Manual trigger for financial analysis agent        | -                          | AI Agent                   |                                                                                                  |
| AI Agent                   | Langchain Agent                 | Processes natural language financial queries        | When Executed by Another Workflow | Edit Fields2               | # 📈 Financial Analysis Documentation (role and usage)                                          |
| Invoices                   | Google Sheets Tool              | Provides invoice data to AI Agent                     | AI Agent (tool input)         | AI Agent                   |                                                                                                  |
| Expenses                   | Google Sheets Tool              | Provides expense data to AI Agent                     | AI Agent (tool input)         | AI Agent                   |                                                                                                  |
| Transactions               | Google Sheets Tool              | Provides bank transaction data to AI Agent            | AI Agent (tool input)         | AI Agent                   |                                                                                                  |
| Edit Fields2               | Set                              | Sets AI agent response as output                       | AI Agent                    | -                           |                                                                                                  |
| OpenRouter Chat Model      | Langchain LLM Chat Model       | Provides LLM chat model for Basic LLM Chain           | Basic LLM Chain (language model) | Basic LLM Chain           |                                                                                                  |
| OpenRouter Chat Model1     | Langchain LLM Chat Model       | Provides LLM chat model for AI Agent                   | AI Agent (language model)    | AI Agent                   |                                                                                                  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**  
   - Google Drive OAuth2 credentials  
   - Google Sheets OAuth2 credentials  
   - Google PaLM API credentials (for Google Gemini)  
   - OpenRouter API credentials (for LLM categorization and agent)

2. **Invoice Processing Block:**

   2.1 Add **When chat message received** node:  
       - Configure to allow file uploads (all MIME types)  

   2.2 Add **Save Invoice** (Google Drive) node:  
       - Operation: Upload file  
       - Folder: Select "Invoices" folder by ID  
       - Input: Binary data from chat upload (field: data0)  

   2.3 Add **Upload PDF to Google Gemini** (HTTP Request) node:  
       - Method: POST  
       - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files`  
       - Content-Type: application/pdf  
       - Auth: Google PaLM API credentials  
       - Binary Data field: data0 from chat upload  

   2.4 Add **Download Data from Google Gemini** (HTTP Request) node:  
       - Method: POST  
       - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-04-17:generateContent`  
       - JSON body includes user prompt requesting invoice fields and the file URI from upload node  
       - Auth: Google PaLM API credentials  

   2.5 Add **Parsed Data** (Set) node:  
       - Expression to remove markdown and parse JSON from AI response text  
       - Enable retry on failure  

   2.6 Add **Split Out** node:  
       - Field to split out: `parsedData['Line Items']`  

   2.7 Add **Table ERP** (Google Sheets) node:  
       - Operation: Append  
       - Document: "Test Invoice Records" Google Sheet  
       - Sheet: Default or "Sheet1"  
       - Map fields: Vendor Name, Invoice Number, Invoice Date, Due Date, Total Amount, VAT Amount, Line Item Description, Quantity, Unit Price, Total Price from parsedData and split items  
       - Cell format: USER_ENTERED  

   2.8 Add **Merge** node:  
       - Merge data from Save Invoice and Table ERP  

   2.9 Add **Update File Name** (Google Drive) node:  
       - Operation: Update  
       - FileId: From Save Invoice node  
       - New filename: `{{$('Table ERP').first().json['Vendor Name']}} - {{$('Table ERP').first().json['Invoice Number']}}`  

   2.10 Add **Confirmation** (Set) node:  
       - Assign string variable `confirmation` with value "All invoices have been processed"  

   2.11 Connect nodes in order:  
       - When chat message received → Save Invoice & Upload PDF to Google Gemini (parallel)  
       - Upload PDF to Google Gemini → Download Data from Google Gemini  
       - Download Data from Google Gemini → Parsed Data  
       - Parsed Data → Split Out  
       - Split Out → Table ERP  
       - Table ERP → Merge  
       - Save Invoice → Merge  
       - Merge → Update File Name  
       - Update File Name → Confirmation  

3. **Expense Processing Block:**

   3.1 Add **Google Drive Trigger** node:  
       - Monitor "Expense Receipts" folder every minute for new files  

   3.2 Add **Google Drive** node:  
       - Operation: Download  
       - FileId: From trigger  

   3.3 Add **Upload PDF to Google Gemini1** (HTTP Request) node:  
       - Same config as invoice upload but input binary field "data" from download node  

   3.4 Add **Download Data from Google Gemini1** (HTTP Request) node:  
       - Similar to invoice data request, tailored prompt for expense receipts  

   3.5 Add **Parsed Data1** (Set) node:  
       - Clean and parse AI JSON response  
       - Retry on failure  

   3.6 Add **Split Out1** node:  
       - Split field: `parsedData['Line Items']`  

   3.7 Add **OpenRouter Chat Model** node:  
       - Model: OpenRouter API  
       - To be used by LLM categorization chain  

   3.8 Add **Structured Output Parser** node:  
       - JSON schema example for category output  

   3.9 Add **Basic LLM Chain** node:  
       - Text input: Line item description and parsed expense data  
       - Use OpenRouter Chat Model and Structured Output Parser  
       - Purpose: Categorize expense line items into 17 categories  

   3.10 Add **Google Sheets** node:  
       - Append to "Expenses Recording" spreadsheet  
       - Map fields including AI category, merchant info, line items  

   3.11 Add **Edit Fields1** (Set) node:  
       - Assign confirmation message "Recorded!"  

   3.12 Connect nodes sequentially as per flow:  
       - Google Drive Trigger → Google Drive (download) → Upload PDF to Google Gemini1 → Download Data from Google Gemini1 → Parsed Data1 → Split Out1 → Basic LLM Chain → Google Sheets → Edit Fields1  

4. **Bank Statement Processing Block:**

   4.1 Add **Google Drive Trigger1** node:  
       - Monitor "Bank Statements" folder every minute  

   4.2 Add **Google Drive1** node:  
       - Download file from trigger  

   4.3 Add **Upload PDF to Google Gemini2** node:  
       - Upload bank statement PDF  

   4.4 Add **Download Data from Google Gemini2** node:  
       - Request bank statement data extraction with prompt  

   4.5 Add **Code** node:  
       - JavaScript code to parse multi-transaction JSON from AI response  

   4.6 Add **Google Sheets1** node:  
       - Append to "Bank Transactions Record" spreadsheet  
       - Map transaction fields accordingly  

   4.7 Add **Edit Fields** node:  
       - Confirmation message "Recorded!"  

   4.8 Connect nodes:  
       - Google Drive Trigger1 → Google Drive1 → Upload PDF to Google Gemini2 → Download Data from Google Gemini2 → Code → Google Sheets1 → Edit Fields  

5. **Financial Analysis Agent Block:**

   5.1 Add **When Executed by Another Workflow** node (disabled by default)  

   5.2 Add **AI Agent** node:  
       - System message defines CFO persona and data access rules  
       - Add three tools: Invoices, Expenses, Transactions (Google Sheets Tool nodes)  
       - Connect AI Agent output to **Edit Fields2** node to set response  

   5.3 Add **OpenRouter Chat Model1** node:  
       - Used as language model by AI Agent  

   5.4 Add **Invoices**, **Expenses**, **Transactions** nodes:  
       - Google Sheets Tool nodes configured to access relevant spreadsheets  

   5.5 Connect nodes:  
       - When Executed by Another Workflow → AI Agent → Edit Fields2  
       - AI Agent uses OpenRouter Chat Model1 as language model  
       - AI Agent uses the three Google Sheets Tools as tools  

---

## 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| AutoSolutions.ai - AI Consulting Services branding with Didac Fernandez Girona logo                                                    | Workflow branding and credits                                                                                                |
| Setup Requirements: Google Drive API, Google Sheets API, Google Gemini API, OpenRouter API credentials                                | Workflow prerequisites for API integration                                                                                   |
| Required Google Drive folders: "Invoices", "Expense Receipts", "Bank Statements"                                                       | Folder structure to be created and permissions set                                                                           |
| Required Google Sheets with specified column headers for invoices, expenses, and transactions                                        | Spreadsheet setup for data storage                                                                                            |
| AI Personas: Mark (Invoices), Donna (Expenses), Victor (Bank Statements), Andrew (Financial Analysis)                                 | Clear role definitions for AI agents                                                                                         |
| Expense Categories: 17 predefined categories used for AI classification                                                              | Expense categorization taxonomy                                                                                              |
| Error handling: Workflow continues processing despite individual document failures; logs errors                                        | Robustness features                                                                                                          |
| Supported file types: PDF only; ensure clear, legible documents for AI extraction                                                     | Input file format requirements                                                                                               |
| Query examples for financial analysis agent: "last 10 bank transactions", "total value of invoices for December 2023", etc.           | Illustrative queries for AI CFO persona                                                                                      |
| https://postimg.cc links to images used in sticky notes for visual aids                                                                | Visual documentation resources                                                                                              |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created using n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---