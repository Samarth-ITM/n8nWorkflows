Extract Invoice Data from PDFs with Gemini AI to Google Sheets 📄

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-pdfs-with-gemini-ai-to-google-sheets----9261


# Extract Invoice Data from PDFs with Gemini AI to Google Sheets 📄

### 1. Workflow Overview

This workflow automates the extraction of structured invoice data from PDF files using Google Gemini AI and stores the processed data in Google Sheets. It is designed for use cases involving automated bookkeeping, invoice processing, and data entry automation. The workflow listens for new PDF files added to Google Drive, extracts text content, uses AI to parse relevant invoice data, compares with already processed documents to avoid duplicates, and logs the extracted data into Google Sheets. Additionally, it can send email notifications and handle batch processing of multiple invoices.

Logical blocks in the workflow are:

- **1.1 Trigger & File Retrieval:** Detect new PDF invoices in Google Drive and download them.
- **1.2 Preprocessing & Deduplication:** Clean the file list and compare against processed invoices to avoid duplicates.
- **1.3 AI-based Data Extraction:** Use Google Gemini AI and LangChain agents to extract structured invoice data from PDFs.
- **1.4 Data Postprocessing & Logging:** Process extracted data, update Google Sheets, and log processed invoices.
- **1.5 Notification & Looping:** Batch process extracted data rows and optionally send email notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & File Retrieval

**Overview:**  
This block initiates the workflow when a new file is added to Google Drive. It retrieves the list of PDF files and downloads the relevant invoice files for processing.

**Nodes Involved:**  
- Google Drive Trigger1  
- Get list of pdf files1  
- Download file1

**Node Details:**

- **Google Drive Trigger1**  
  - Type: Trigger node for Google Drive  
  - Role: Listens for new or modified files in a specified Google Drive folder (likely configured to target invoice PDFs)  
  - Input: Event from Google Drive (file added/modified)  
  - Output: Metadata about new/changed files  
  - Potential Failures: Authentication errors, permission issues, or event misconfiguration  
  - Notes: Requires Google Drive OAuth2 credentials

- **Get list of pdf files1**  
  - Type: HTTP Request  
  - Role: Retrieves a list of PDF files from Google Drive or an API endpoint  
  - Configuration: Likely configured with request parameters to filter or fetch PDF files  
  - Input: Trigger output  
  - Output: List of PDF file metadata  
  - Failures: API timeouts, incorrect endpoint, or malformed response

- **Download file1**  
  - Type: Google Drive node (Download file)  
  - Role: Downloads the actual PDF file content for further processing  
  - Input: File metadata from previous node  
  - Output: Binary data of PDF file  
  - Failures: File not found, access denied, file corrupted

---

#### 1.2 Preprocessing & Deduplication

**Overview:**  
This block cleans the list of PDF files, reads records of already processed invoices from Google Sheets, and compares datasets to identify new invoices to process.

**Nodes Involved:**  
- Cleans output1  
- Read processed docs1  
- Compare Datasets1  
- Limit1 (disabled)

**Node Details:**

- **Cleans output1**  
  - Type: Code node (JavaScript)  
  - Role: Cleans or transforms the raw list of files retrieved, e.g., filtering by file type or removing invalid entries  
  - Key Logic: Custom JavaScript code to sanitize input data  
  - Input: List of PDFs from HTTP Request node  
  - Output: Filtered/cleaned list  
  - Failures: Expression errors or unexpected data formats

- **Read processed docs1**  
  - Type: Google Sheets (Read rows)  
  - Role: Reads rows from a Google Sheet that tracks invoices already processed to avoid duplication  
  - Input: Trigger output (runs in parallel to file list retrieval)  
  - Output: List of processed document identifiers  
  - Failures: Authorization, incorrect sheet ID or range

- **Compare Datasets1**  
  - Type: Compare Datasets  
  - Role: Compares the cleaned list of files against processed docs to identify new files needing processing  
  - Input: Outputs of Cleans output1 and Read processed docs1  
  - Output: New files to process  
  - Failures: Data structure mismatches, empty inputs

- **Limit1** (Disabled)  
  - Type: Limit node  
  - Role: Limits the number of files processed, useful for testing or throttling  
  - Currently disabled; when enabled, it restricts the batch size

---

#### 1.3 AI-based Data Extraction

**Overview:**  
This block extracts text from the downloaded PDF files, then uses Google Gemini AI and LangChain agents to parse and structure invoice data.

**Nodes Involved:**  
- Extract from File1  
- Google Gemini Chat Model2  
- AI Agent - get targeted elements from text1  
- Google Gemini Chat Model3  
- Structured Output Parser1

**Node Details:**

- **Extract from File1**  
  - Type: Extract from File  
  - Role: Converts the binary PDF into text or extractable data for AI processing  
  - Input: PDF binary data from Download file1  
  - Output: Extracted raw text or structured content  
  - Failures: Unsupported PDF format, extraction failure

- **Google Gemini Chat Model2**  
  - Type: Google Gemini AI language model node  
  - Role: Provides an initial AI chat model interface for processing raw text input  
  - Input: Extracted text from Extract from File1  
  - Output: AI-generated text or response  
  - Configuration: Uses configured OpenAI or Google credentials for Gemini AI  
  - Failures: Authentication errors, API limits, malformed prompts

- **AI Agent - get targeted elements from text1**  
  - Type: LangChain Agent node  
  - Role: Utilizes LangChain agent to parse targeted invoice elements (e.g., invoice number, date, total) from AI model output  
  - Input: Output from Google Gemini Chat Model2 and Google Gemini Chat Model3 (via ai_languageModel and ai_outputParser connections)  
  - Output: Parsed structured data  
  - Failures: Agent misconfiguration, parsing errors

- **Google Gemini Chat Model3**  
  - Type: Google Gemini AI language model node  
  - Role: Secondary AI call, likely for refining or structuring parsed data  
  - Input: Possibly text or context from Agent node  
  - Output: Structured response for parsing  
  - Failures: Similar to Model2

- **Structured Output Parser1**  
  - Type: LangChain structured output parser  
  - Role: Parses AI output into a structured JSON format usable for downstream processing  
  - Input: Output from Google Gemini Chat Model3  
  - Output: Structured invoice data  
  - Failures: Parsing failures, incorrect schema

---

#### 1.4 Data Postprocessing & Logging

**Overview:**  
Processes the structured invoice data, updates Google Sheets with invoice fields, logs the processed invoices, and prepares data for batch operations.

**Nodes Involved:**  
- Loop Over Items1  
- Code in JavaScript1  
- Update invoice Fields1  
- Log the processing of the doc1

**Node Details:**

- **Loop Over Items1**  
  - Type: SplitInBatches  
  - Role: Iterates over each extracted invoice item in batches for controlled processing  
  - Input: Structured data array from AI Agent node  
  - Output: Individual or batch items  
  - Failures: Batch size misconfiguration

- **Code in JavaScript1**  
  - Type: Code node (JavaScript)  
  - Role: Custom logic to transform or prepare invoice data fields before updating sheets  
  - Input: Single item from Loop Over Items1  
  - Output: Processed data for update  
  - Failures: Script errors, data inconsistency

- **Update invoice Fields1**  
  - Type: Google Sheets (Append/Update row)  
  - Role: Updates or appends invoice data fields into a Google Sheet for record keeping  
  - Input: Processed invoice data from code node  
  - Output: Confirmation of update  
  - Failures: Sheet permissions, invalid row data

- **Log the processing of the doc1**  
  - Type: Google Sheets (Append)  
  - Role: Logs metadata of processed invoices to prevent reprocessing  
  - Input: Confirmation or data from Update invoice Fields1  
  - Output: Log entry  
  - Failures: Sheet access, quota limits

---

#### 1.5 Notification & Looping

**Overview:**  
Finalizes batch processing and optionally sends email notifications for processed invoices.

**Nodes Involved:**  
- Send a message1  
- Limit1 (enabled via connection from Compare Datasets1, although disabled in current config)

**Node Details:**

- **Send a message1**  
  - Type: Gmail node  
  - Role: Sends email notifications, possibly with invoice summaries or alerts  
  - Input: Items from SplitInBatches node  
  - Output: Email sending status  
  - Failures: Authentication, SMTP issues, invalid email addresses  
  - Credentials: Requires Gmail OAuth2 credentials

- **Limit1** (see above)  
  - When enabled, controls the number of invoices processed or notified per run

---

### 3. Summary Table

| Node Name                      | Node Type                             | Functional Role                             | Input Node(s)                  | Output Node(s)                   | Sticky Note                          |
|--------------------------------|-------------------------------------|--------------------------------------------|-------------------------------|---------------------------------|------------------------------------|
| Intro Sticky                   | Sticky Note                         | Introductory comment placeholder           |                               |                                 |                                    |
| Setup Sticky                   | Sticky Note                         | Setup instructions placeholder              |                               |                                 |                                    |
| AI Extraction Sticky           | Sticky Note                         | Notes on AI extraction step                  |                               |                                 |                                    |
| Data Split Sticky              | Sticky Note                         | Notes on data splitting/batching             |                               |                                 |                                    |
| Google Sheets Sticky           | Sticky Note                         | Notes related to Google Sheets operations    |                               |                                 |                                    |
| Gmail Sticky                  | Sticky Note                         | Notes on Gmail email sending                  |                               |                                 |                                    |
| Extend Sticky                 | Sticky Note                         | Additional notes or extension points          |                               |                                 |                                    |
| Compare Datasets1             | Compare Datasets                    | Deduplicate new files vs processed docs      | Cleans output1, Read processed docs1 | Limit1                          |                                    |
| Extract from File1            | Extract from File                   | Converts PDF file to extractable text         | Download file1                | AI Agent - get targeted elements from text1 |                                    |
| Google Gemini Chat Model2     | Google Gemini AI Language Model    | AI processing of extracted text               | Extract from File1            | AI Agent - get targeted elements from text1 |                                    |
| Log the processing of the doc1| Google Sheets                      | Log processed invoice metadata                | Update invoice Fields1        | Loop Over Items1                |                                    |
| Cleans output1                | Code (JavaScript)                  | Cleans and filters list of PDF files          | Get list of pdf files1        | Compare Datasets1               |                                    |
| Get list of pdf files1        | HTTP Request                      | Retrieves list of PDFs                        | Google Drive Trigger1         | Cleans output1                 |                                    |
| Read processed docs1          | Google Sheets                     | Reads list of already processed invoices      | Google Drive Trigger1         | Compare Datasets1              |                                    |
| Sticky Note4                 | Sticky Note                       | Placeholder note                              |                               |                                 |                                    |
| Sticky Note5                 | Sticky Note                       | Placeholder note                              |                               |                                 |                                    |
| AI Agent - get targeted elements from text1 | LangChain Agent                | Extracts structured data using AI             | Extract from File1, Google Gemini Chat Model2, Structured Output Parser1 | Loop Over Items1                |                                    |
| Sticky Note7                 | Sticky Note                       | Placeholder note                              |                               |                                 |                                    |
| Google Drive Trigger1         | Google Drive Trigger              | Starts workflow on new file in Google Drive   |                               | Get list of pdf files1, Read processed docs1 |                                    |
| Download file1                | Google Drive (Download file)      | Downloads PDF binary                          | Limit1                       | Extract from File1             |                                    |
| Structured Output Parser1     | LangChain Structured Output Parser | Parses AI output into JSON structured data    | Google Gemini Chat Model3     | AI Agent - get targeted elements from text1 |                                    |
| Google Gemini Chat Model3     | Google Gemini AI Language Model    | Secondary AI model for structured parsing     | AI Agent - get targeted elements from text1 | Structured Output Parser1       |                                    |
| Limit1                       | Limit                            | (Disabled) Limits processed file count        | Compare Datasets1             | Download file1                 |                                    |
| Send a message1              | Gmail                            | Sends email notifications                     | Loop Over Items1              |                                 |                                    |
| Update invoice Fields1       | Google Sheets                    | Updates Google Sheet with invoice data         | Code in JavaScript1           | Log the processing of the doc1 |                                    |
| Loop Over Items1             | SplitInBatches                   | Processes extracted data in batches            | Log the processing of the doc1, AI Agent  | Send a message1, Code in JavaScript1 |                                    |
| Code in JavaScript1          | Code (JavaScript)                | Processes invoice data before sheet update    | Loop Over Items1              | Update invoice Fields1         |                                    |
| Sticky Note                  | Sticky Note                     | Placeholder note                              |                               |                                 |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Configure to monitor the folder where invoices (PDFs) are uploaded  
   - Set OAuth2 credentials for Google Drive

2. **Create HTTP Request node ("Get list of pdf files1")**  
   - Connect input from Google Drive Trigger  
   - Configure to request list of PDF files from Drive or API endpoint filtering PDFs  
   - Use authentication as needed

3. **Create Code node ("Cleans output1")**  
   - Connect input from HTTP Request node  
   - Write JavaScript to filter/clean the list of files (e.g., only PDF mimeType, remove duplicates)  
   - Output cleaned list

4. **Create Google Sheets node ("Read processed docs1")**  
   - Connect input from Google Drive Trigger (runs parallel with Get list of pdf files1)  
   - Configure to read rows from a Google Sheet tracking processed invoices  
   - Set Sheet ID, range, and OAuth2 credentials

5. **Create Compare Datasets node ("Compare Datasets1")**  
   - Connect inputs from Cleans output1 and Read processed docs1  
   - Configure to identify new files by comparing file IDs or unique identifiers  
   - Output list of unprocessed files

6. **Create Limit node ("Limit1")**  
   - Connect input from Compare Datasets1  
   - (Optional) Configure batch size limit to control processing load  
   - In this workflow, the node is disabled by default

7. **Create Google Drive node ("Download file1")**  
   - Connect input from Limit1 (or Compare Datasets1 if Limit disabled)  
   - Configure to download PDF files using file IDs from the previous node  
   - Use Google Drive OAuth2 credentials

8. **Create Extract from File node ("Extract from File1")**  
   - Connect input from Download file1  
   - Configure to extract text content from the PDF binary data

9. **Create Google Gemini Chat Model node ("Google Gemini Chat Model2")**  
   - Connect input from Extract from File1  
   - Configure with Gemini AI credentials and prompt settings to process raw text extraction

10. **Create LangChain Agent node ("AI Agent - get targeted elements from text1")**  
    - Connect input from Google Gemini Chat Model2 (ai_languageModel)  
    - Connect input from Structured Output Parser1 (ai_outputParser) (to be created next)  
    - Configure prompts and agent parameters to extract invoice-specific fields

11. **Create Google Gemini Chat Model node ("Google Gemini Chat Model3")**  
    - Configure a second AI call for parsing or refining output  
    - Connect input from AI Agent node if needed for chaining

12. **Create Structured Output Parser node ("Structured Output Parser1")**  
    - Connect input from Google Gemini Chat Model3  
    - Configure to parse AI responses into structured JSON according to invoice schema

13. **Connect Structured Output Parser1 to AI Agent node's ai_outputParser input**

14. **Create SplitInBatches node ("Loop Over Items1")**  
    - Connect input from AI Agent node's main output  
    - Configure batch size for processing extracted invoice entries

15. **Create Code node ("Code in JavaScript1")**  
    - Connect input from Loop Over Items1  
    - Write code to transform or prepare data for Google Sheets update

16. **Create Google Sheets node ("Update invoice Fields1")**  
    - Connect input from Code in JavaScript1  
    - Configure to append or update rows in the invoice data sheet  
    - Set Sheet ID, range, and OAuth2 credentials

17. **Create Google Sheets node ("Log the processing of the doc1")**  
    - Connect input from Update invoice Fields1  
    - Configure to append rows logging processed documents to a dedicated sheet

18. **Connect Log the processing of the doc1 to Loop Over Items1 to continue processing batches**

19. **Create Gmail node ("Send a message1")**  
    - Connect input from Loop Over Items1  
    - Configure Gmail OAuth2 credentials  
    - Set up email template for notification regarding processed invoices

20. **Add Sticky Notes as needed for documentation or explanations**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                    |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow uses Google Gemini AI within LangChain agents for advanced invoice data extraction.             | Workflow core AI processing                         |
| Google Drive Trigger requires folder path or ID to monitor for new invoices.                                  | Google Drive integration setup                      |
| Google Sheets nodes must be configured with correct Sheet IDs and ranges to track processed invoices and data.| Google Sheets configuration                         |
| Gmail node requires OAuth2 credentials and proper permissions for sending emails.                             | Email notification feature                          |
| Batch processing via SplitInBatches node helps avoid API rate limits and manage large invoice sets.          | Performance and scalability consideration           |
| Disabled Limit node can be enabled for testing or throttling processing volumes.                              | Optional control mechanism                           |
| For AI nodes, ensure API keys and usage limits for Google Gemini are properly managed to avoid interruptions. | AI integration management                            |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.