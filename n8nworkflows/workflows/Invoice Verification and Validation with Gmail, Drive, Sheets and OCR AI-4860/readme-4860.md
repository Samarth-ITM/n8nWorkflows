Invoice Verification and Validation with Gmail, Drive, Sheets and OCR AI

https://n8nworkflows.xyz/workflows/invoice-verification-and-validation-with-gmail--drive--sheets-and-ocr-ai-4860


# Invoice Verification and Validation with Gmail, Drive, Sheets and OCR AI

### 1. Workflow Overview

This workflow automates the **extraction, validation, and processing of invoice data** received via Gmail, leveraging Google Drive and Google Sheets for storage and record-keeping, and AI-powered OCR for text extraction. It is designed for organizations that receive invoices by email and need to automate their verification and data entry processes.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Watches Gmail for incoming invoice emails, reads messages and attachments.
- **1.2 Google Drive Folder Management:** Ensures appropriate folder hierarchy (by month and day) exists in Google Drive for storing invoices.
- **1.3 Attachment Handling & Extraction:** Retrieves attachments, uploads them to Drive, extracts raw text using OCR/AI agents.
- **1.4 Data Processing & AI Post-Processing:** Processes extracted text with AI models to parse invoice data and validate it.
- **1.5 Data Validation & Google Sheets Integration:** Fetches master data, validates extracted information, writes results and updates totals in Sheets.
- **1.6 Notifications & Looping:** Handles batch processing and sends notifications after processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers when a new email arrives in Gmail, reads the email and extracts attachments for processing.

- **Nodes Involved:**  
  Gmail Trigger, Date From Email Subject, Read Message, Merge, Get Attachments, Extract from File

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Config: Listens for new incoming emails; execution once true to avoid duplicate processing initially.  
    - Inputs: None (trigger)  
    - Outputs: Email metadata and message data  
    - Edge Cases: Email format variations, missing attachments, Gmail API limits.

  - **Date From Email Subject**  
    - Type: Code node  
    - Role: Parses the date from the email subject line, likely for folder organization or metadata.  
    - Inputs: Gmail Trigger output  
    - Outputs: Parsed date information  
    - Edge Cases: Malformed or missing date in subject causing parsing failures.

  - **Read Message**  
    - Type: Gmail node  
    - Role: Reads full email content including bodies and attachments.  
    - Config: Executes once, fetches message content.  
    - Inputs: Date From Email Subject output  
    - Outputs: Full message data  
    - Edge Cases: Email size limits, Gmail API errors.

  - **Merge**  
    - Type: Merge node  
    - Role: Combines data streams from Gmail Trigger and Read Message for unified processing.  
    - Inputs: Gmail Trigger (secondary), Read Message (primary)  
    - Outputs: Combined data  
    - Edge Cases: Mismatched data sets or timing issues.

  - **Get Attachments**  
    - Type: Code node  
    - Role: Extracts attachments from the merged email data for further processing.  
    - Inputs: Merge output  
    - Outputs: Attachment files  
    - Edge Cases: Missing or unsupported attachment types.

  - **Extract from File**  
    - Type: Extract from File node (likely for extracting metadata or text)  
    - Role: Processes attachments to prepare for OCR or AI extraction.  
    - Inputs: Get Attachments output  
    - Outputs: File extraction data  
    - Edge Cases: Corrupt files, unsupported file formats.

#### 2.2 Google Drive Folder Management

- **Overview:**  
  This block ensures that Google Drive folders for the current month and day exist, creating them if needed, organizing invoice storage.

- **Nodes Involved:**  
  Search Month Folder, Month Folder Found?, Create Month Folder, Search Day Folder, Day Folder Found?, Create Day Folder, Get Parent Folder ID, Get Day Folder ID

- **Node Details:**

  - **Search Month Folder**  
    - Type: Google Drive node  
    - Role: Searches for a folder named after the current month.  
    - Inputs: Read Message output (date context)  
    - Outputs: Folder info if found  
    - Edge Cases: Folder not found, Drive API errors.

  - **Month Folder Found?**  
    - Type: If node  
    - Role: Checks if the month folder exists, branches accordingly.  
    - Inputs: Search Month Folder output  
    - Outputs: True branch (folder found), False branch (folder not found)  
    - Edge Cases: False negatives due to API delay.

  - **Create Month Folder**  
    - Type: Google Drive node  
    - Role: Creates month folder if not found.  
    - Inputs: Month Folder Found? false branch  
    - Outputs: New folder info  
    - Edge Cases: Permissions errors, folder name conflicts.

  - **Search Day Folder**  
    - Type: Google Drive node  
    - Role: Searches for folder named by day inside the month folder.  
    - Inputs: Month Folder Found? true branch or Create Month Folder output  
    - Outputs: Folder info if found  
    - Edge Cases: Similar to month folder.

  - **Day Folder Found?**  
    - Type: If node  
    - Role: Checks if the day folder exists.  
    - Inputs: Search Day Folder output  
    - Outputs: True (exists), False (create needed)  
    - Edge Cases: As above.

  - **Create Day Folder**  
    - Type: Google Drive node  
    - Role: Creates day folder if missing.  
    - Inputs: Day Folder Found? false branch  
    - Outputs: New folder info  
    - Edge Cases: Similar permissions or naming conflicts.

  - **Get Parent Folder ID**  
    - Type: Google Drive node  
    - Role: Retrieves parent folder ID for folder creation hierarchy.  
    - Inputs: Day Folder Found? true branch  
    - Outputs: Parent folder ID metadata  
    - Edge Cases: API retrieval failures.

  - **Get Day Folder ID**  
    - Type: Google Drive node  
    - Role: Retrieves the day folder ID for attachment uploads.  
    - Inputs: Create Day Folder output or Day Folder Found? true branch  
    - Outputs: Folder ID  
    - Edge Cases: Sync or API issues.

#### 2.3 Attachment Handling & Extraction

- **Overview:**  
  Retrieves attachments, uploads them into Drive folders, and extracts text using AI-powered OCR or language models.

- **Nodes Involved:**  
  Upload Invoices, Loop Over Items, Text Extractor, OpenRouter Chat Model, Post-Processing, If, Send Raw Text Again, Split Out, Generate Unique Key

- **Node Details:**

  - **Upload Invoices**  
    - Type: Google Drive node  
    - Role: Uploads invoice attachments into the appropriate Drive folder.  
    - Inputs: Get Attachments output  
    - Outputs: Uploaded file metadata  
    - Edge Cases: Upload failure, quota limits.

  - **Loop Over Items**  
    - Type: Split In Batches node  
    - Role: Processes each uploaded invoice file individually to manage workload.  
    - Inputs: Upload Invoices output  
    - Outputs: Single file per iteration  
    - Edge Cases: Large batch sizes causing timeouts.

  - **Text Extractor**  
    - Type: LangChain Agent node  
    - Role: Extracts raw text from invoice files using AI-based OCR and NLP.  
    - Inputs: Loop Over Items output  
    - Outputs: Raw extracted text  
    - Edge Cases: OCR inaccuracies, file compatibility.

  - **OpenRouter Chat Model**  
    - Type: LangChain OpenRouter Chat Model  
    - Role: Provides AI language model processing for text extraction and understanding.  
    - Inputs: Text Extractor AI language model input  
    - Outputs: Processed text or parsed data  
    - Edge Cases: Model API errors, rate limits.

  - **Post-Processing**  
    - Type: Code node  
    - Role: Cleans and formats extracted text for validation.  
    - Inputs: Text Extractor output  
    - Outputs: Processed text ready for validation  
    - Edge Cases: Code exceptions, unexpected text formats.

  - **If**  
    - Type: If node  
    - Role: Branches workflow based on validation or content presence.  
    - Inputs: Post-Processing output  
    - Outputs: True (valid), False (retry)  
    - Edge Cases: False negatives or positives.

  - **Send Raw Text Again**  
    - Type: Set node  
    - Role: Resets or prepares raw text to retry extraction or further processing.  
    - Inputs: If false branch  
    - Outputs: Data for reprocessing  
    - Edge Cases: Infinite retry loops.

  - **Split Out**  
    - Type: Split Out node  
    - Role: Splits output data into smaller parts for detailed processing or parallelism.  
    - Inputs: If true branch  
    - Outputs: Split data items  
    - Edge Cases: Data loss or incorrect splitting.

  - **Generate Unique Key**  
    - Type: Set node  
    - Role: Generates unique identifiers for each invoice processed, used for tracking and storage.  
    - Inputs: Split Out output  
    - Outputs: Data with unique keys  
    - Edge Cases: Key collisions (rare but possible).

#### 2.4 Data Validation & Google Sheets Integration

- **Overview:**  
  Validates extracted invoice data against master records and writes validated results and totals into Google Sheets.

- **Nodes Involved:**  
  Send Invoice Data, Fetch Master Data, Validation, Update Results, Get last Index, Update Totals

- **Node Details:**

  - **Send Invoice Data**  
    - Type: Google Sheets node  
    - Role: Sends structured invoice data to a Google Sheets spreadsheet for further reference.  
    - Inputs: Generate Unique Key output  
    - Outputs: Confirmation of write operation  
    - Edge Cases: Sheet access errors.

  - **Fetch Master Data**  
    - Type: Google Sheets node  
    - Role: Retrieves master data (e.g., vendor info, pricing rules) to assist validation.  
    - Inputs: Send Invoice Data output  
    - Outputs: Master data rows  
    - Edge Cases: Permissions, sheet structure changes.

  - **Validation**  
    - Type: Code node  
    - Role: Performs logical checks and data validation comparing invoice data to master data.  
    - Inputs: Fetch Master Data output  
    - Outputs: Validation results  
    - Edge Cases: Logic bugs, unexpected data formats.

  - **Update Results**  
    - Type: Google Sheets node  
    - Role: Updates the results of validation back into the Google Sheets record.  
    - Inputs: Validation output  
    - Outputs: Updated sheet status  
    - Edge Cases: Partial write failures.

  - **Get last Index**  
    - Type: Code node  
    - Role: Determines the last row index for updating totals or appending data.  
    - Inputs: Update Results output  
    - Outputs: Index number  
    - Edge Cases: Empty sheets or corrupted index data.

  - **Update Totals**  
    - Type: Google Sheets node  
    - Role: Updates cumulative totals or summary fields in the sheet.  
    - Inputs: Get last Index output  
    - Outputs: Updated totals  
    - Edge Cases: Formula disruption or concurrency conflicts.

#### 2.5 Notifications & Looping

- **Overview:**  
  Manages batch processing loops and sends email notifications after processing invoices.

- **Nodes Involved:**  
  Wait, Notify

- **Node Details:**

  - **Wait**  
    - Type: Wait node  
    - Role: Provides delay or throttling between batches or steps to manage load.  
    - Inputs: Update Totals output  
    - Outputs: Delayed trigger to Loop Over Items  
    - Edge Cases: Long delays causing timeouts in workflows.

  - **Notify**  
    - Type: Gmail node  
    - Role: Sends notification emails to stakeholders or systems regarding processing status.  
    - Inputs: Loop Over Items outputs (post-processing)  
    - Outputs: Email sent confirmation  
    - Edge Cases: SMTP or Gmail API errors.

---

### 3. Summary Table

| Node Name             | Node Type                           | Functional Role                       | Input Node(s)          | Output Node(s)          | Sticky Note                 |
|-----------------------|-----------------------------------|------------------------------------|------------------------|-------------------------|-----------------------------|
| Gmail Trigger         | Gmail Trigger                     | Triggers on new email               | -                      | Date From Email Subject, Merge |                             |
| Date From Email Subject| Code                             | Parses date from email subject      | Gmail Trigger          | Read Message            |                             |
| Read Message          | Gmail                            | Reads full email content            | Date From Email Subject| Search Month Folder      |                             |
| Merge                 | Merge                            | Merges Gmail trigger and message    | Gmail Trigger, Read Message | Get Attachments       |                             |
| Get Attachments       | Code                             | Extracts attachments from email     | Merge                  | Upload Invoices, Extract from File |                             |
| Extract from File     | Extract From File                 | Extracts data from files             | Get Attachments        | Loop Over Items         |                             |
| Upload Invoices       | Google Drive                     | Uploads attachments to Drive        | Get Attachments        | Loop Over Items         |                             |
| Loop Over Items       | Split In Batches                 | Processes items in batches           | Extract from File, Upload Invoices | Notify, Text Extractor |                             |
| Text Extractor        | LangChain Agent                  | Extracts text using AI OCR/NLP      | Loop Over Items        | Post-Processing         |                             |
| OpenRouter Chat Model | LangChain OpenRouter Chat Model  | AI language model for text extraction | Text Extractor (ai_languageModel) | Text Extractor         |                             |
| Post-Processing       | Code                            | Cleans/formats extracted text       | Text Extractor         | If                      |                             |
| If                    | If                              | Branches based on validation        | Post-Processing        | Send Raw Text Again, Split Out |                             |
| Send Raw Text Again   | Set                             | Resets text for reprocessing        | If (false branch)      | Text Extractor          |                             |
| Split Out             | Split Out                       | Splits data for detailed processing | If (true branch)       | Generate Unique Key     |                             |
| Generate Unique Key   | Set                             | Creates unique keys for invoices    | Split Out              | Send Invoice Data       |                             |
| Send Invoice Data     | Google Sheets                   | Writes invoice data to Sheets       | Generate Unique Key    | Fetch Master Data       |                             |
| Fetch Master Data     | Google Sheets                   | Retrieves master validation data    | Send Invoice Data      | Validation              |                             |
| Validation            | Code                            | Validates invoice data              | Fetch Master Data      | Update Results          |                             |
| Update Results        | Google Sheets                   | Updates validation results          | Validation             | Get last Index          |                             |
| Get last Index        | Code                            | Gets last row index                 | Update Results         | Update Totals           |                             |
| Update Totals         | Google Sheets                   | Updates totals summary              | Get last Index         | Wait                    |                             |
| Wait                  | Wait                           | Delays workflow for throttling     | Update Totals          | Loop Over Items         |                             |
| Notify                | Gmail                          | Sends notification emails          | Loop Over Items        | -                       |                             |
| Search Month Folder   | Google Drive                   | Searches for month folder           | Read Message           | Month Folder Found?     |                             |
| Month Folder Found?   | If                             | Checks month folder existence       | Search Month Folder    | Create Month Folder, Search Day Folder |                             |
| Create Month Folder   | Google Drive                   | Creates month folder if missing     | Month Folder Found? false | Search Day Folder      |                             |
| Search Day Folder     | Google Drive                   | Searches for day folder             | Month Folder Found? true, Create Month Folder | Day Folder Found? |                             |
| Day Folder Found?     | If                             | Checks day folder existence         | Search Day Folder      | Get Parent Folder ID, Get Day Folder ID, Create Day Folder |                             |
| Create Day Folder     | Google Drive                   | Creates day folder if missing       | Get Parent Folder ID   | Get Day Folder ID       |                             |
| Get Parent Folder ID  | Google Drive                   | Gets parent folder ID               | Day Folder Found? true | Create Day Folder       |                             |
| Get Day Folder ID     | Google Drive                   | Gets day folder ID                  | Create Day Folder, Day Folder Found? true | Merge                  |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Purpose: Trigger when new emails arrive in the inbox.  
   - Configure OAuth2 credentials for Gmail.  
   - Set to execute once initially to prevent duplicate triggers.

2. **Add Code node "Date From Email Subject"**  
   - Parses date from incoming email subject line to use in folder naming.  
   - Connect Gmail Trigger output to this node.

3. **Add Gmail node "Read Message"**  
   - Reads full message content and attachments from email.  
   - Connect from "Date From Email Subject".  
   - Use same OAuth2 credentials.

4. **Add Merge node**  
   - Merge data streams from Gmail Trigger (secondary input) and Read Message (primary input).  
   - Connect Gmail Trigger (secondary) and Read Message (primary) outputs here.

5. **Add Code node "Get Attachments"**  
   - Extracts attachment files from merged email data.  
   - Connect Merge output to this node.

6. **Add Google Drive node "Upload Invoices"**  
   - Upload extracted attachments to Google Drive.  
   - Connect from "Get Attachments".  
   - Set Drive OAuth2 credentials.  
   - Destination folder ID will be set dynamically after folder management.

7. **Add Extract From File node "Extract from File"**  
   - Extract text or metadata from uploaded files.  
   - Connect from "Get Attachments" as well.

8. **Add Split In Batches node "Loop Over Items"**  
   - Process each uploaded invoice file separately.  
   - Connect outputs of "Upload Invoices" and "Extract from File" to this node.

9. **Add LangChain Agent node "Text Extractor"**  
   - Use AI OCR/NLP to extract raw text from invoice files.  
   - Connect "Loop Over Items" output to this node.  
   - Configure AI credentials (OpenAI, OpenRouter, or similar).

10. **Add LangChain OpenRouter Chat Model node**  
    - AI language model for advanced text understanding.  
    - Connect "Text Extractor" as AI language model input.

11. **Add Code node "Post-Processing"**  
    - Clean and format extracted text for validation.  
    - Connect "Text Extractor" output to this node.

12. **Add If node**  
    - Decision node based on post-processing result (valid or not).  
    - Connect "Post-Processing" output here.

13. **Add Set node "Send Raw Text Again"**  
    - Prepares raw text for retry if invalid.  
    - Connect from If node false branch.  
    - Connect back to "Text Extractor" for retry.

14. **Add Split Out node**  
    - Splits valid data into individual items for processing.  
    - Connect If node true branch here.

15. **Add Set node "Generate Unique Key"**  
    - Generate unique identifiers for each invoice for tracking.  
    - Connect "Split Out" output here.

16. **Add Google Sheets node "Send Invoice Data"**  
    - Write invoice data to Google Sheets.  
    - Connect "Generate Unique Key" output here.  
    - Configure credentials and target sheet.

17. **Add Google Sheets node "Fetch Master Data"**  
    - Retrieve master records for validation.  
    - Connect from "Send Invoice Data".  
    - Set to execute once.

18. **Add Code node "Validation"**  
    - Validate invoice data against master data.  
    - Connect from "Fetch Master Data".

19. **Add Google Sheets node "Update Results"**  
    - Update validation results in Sheets.  
    - Connect from "Validation".

20. **Add Code node "Get last Index"**  
    - Obtain last row index in Sheets for summaries.  
    - Connect from "Update Results".

21. **Add Google Sheets node "Update Totals"**  
    - Update totals or summaries in the spreadsheet.  
    - Connect from "Get last Index".

22. **Add Wait node**  
    - Introduce delay before next batch processing.  
    - Connect from "Update Totals".

23. **Connect Wait output back to "Loop Over Items"**  
    - Enables batch processing loop.

24. **Add Gmail node "Notify"**  
    - Send notification email upon processing completion or errors.  
    - Connect from "Loop Over Items" (after processing).

25. **Add Google Drive nodes to manage folders:**  
    - "Search Month Folder" → "Month Folder Found?" → ("Create Month Folder" if false)  
    - "Search Day Folder" → "Day Folder Found?" → ("Create Day Folder" if false)  
    - "Get Parent Folder ID" and "Get Day Folder ID" for folder hierarchy and upload destination.  
    - Connect these nodes starting from "Read Message" output and ensure folder IDs are passed to "Upload Invoices".

26. **Ensure all nodes have correct credentials configured:**  
    - Gmail OAuth2 for Gmail nodes  
    - Google Drive OAuth2 for Drive nodes  
    - Google Sheets OAuth2 for Sheets nodes  
    - OpenRouter/OpenAI API keys for AI nodes

27. **Configure node parameters for batch sizes, retry logic, and error handling as needed.**

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow uses LangChain nodes and OpenRouter Chat Model for AI-powered OCR and NLP processing.  | Requires API keys and credential setup for AI providers.                                                 |
| Folder hierarchy in Google Drive is based on extracted date from email subject (Month/Day).     | Ensures organized storage of invoices by date.                                                           |
| Notifications are sent via Gmail node after processing each invoice batch.                       | Helps stakeholders track workflow status.                                                                 |
| The workflow includes retry logic to resend raw text for AI extraction if initial parsing fails.| Prevents data loss due to OCR or AI misinterpretation.                                                    |
| Master data used for validation is fetched once per batch to optimize performance.              | Reduces API calls and speeds up validation.                                                               |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated n8n workflow implementation. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.