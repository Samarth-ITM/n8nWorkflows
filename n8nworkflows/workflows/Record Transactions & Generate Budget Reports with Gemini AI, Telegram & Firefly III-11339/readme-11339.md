Record Transactions & Generate Budget Reports with Gemini AI, Telegram & Firefly III

https://n8nworkflows.xyz/workflows/record-transactions---generate-budget-reports-with-gemini-ai--telegram---firefly-iii-11339


# Record Transactions & Generate Budget Reports with Gemini AI, Telegram & Firefly III

### 1. Workflow Overview

This workflow automates personal finance management by integrating Telegram, Google Gemini AI, and Firefly III budgeting software. It enables users to send transaction data via images, PDFs, or text messages to a Telegram channel. The workflow extracts, parses, categorizes, and posts transaction records to Firefly III. Additionally, sending the command “Report” generates a budget report for the current month and delivers it back via Telegram.

The workflow’s logic is divided into the following functional blocks:

- **1.1 Input Reception:** Receiving transaction data or commands through Telegram.
- **1.2 Input Type Detection and Retrieval:** Differentiating between images, PDFs, and text commands, then fetching files from Telegram servers.
- **1.3 AI-based Transaction Data Extraction:** Using Google Gemini AI to analyze images or PDFs and parse structured transaction details.
- **1.4 Transaction Data Formatting:** Transforming and categorizing parsed data for Firefly III.
- **1.5 Transaction Posting:** Sending individual transactions to Firefly III API and confirming via Telegram.
- **1.6 Budget Report Generation:** On command, retrieving monthly budget data from Firefly III, formatting it as CSV, and sending it back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Listens to incoming messages on Telegram, which can be images, PDFs, or text commands.

**Nodes Involved:**  
- Transaction Image or PDF Received (Telegram Trigger)

**Node Details:**

- **Transaction Image or PDF Received**  
  - Type: Telegram Trigger  
  - Role: Listens for new messages in Telegram, triggers workflow on any message update.  
  - Configuration: Watches for all message updates, including text, photo, and document messages.  
  - Input: Telegram updates  
  - Output: Message JSON with content type info  
  - Edge Cases: Could receive unsupported message types; requires proper filtering downstream.  
  - Credentials: Telegram API credentials required.

---

#### 2.2 Input Type Detection and Retrieval

**Overview:**  
Determines the type of incoming data (image, PDF, or "Report" command) and retrieves the file from Telegram servers if applicable.

**Nodes Involved:**  
- Switch  
- Get image from server (Telegram node)  
- Get file from server (Telegram node)

**Node Details:**

- **Switch**  
  - Type: Switch node  
  - Role: Branches workflow based on message content type.  
  - Configuration: Checks if message includes photo, document, or text equals "Report"/"report".  
  - Outputs: Four branches — Image, PDF, Report (case-sensitive), report (case-insensitive)  
  - Edge Cases: Case sensitivity handled; message without expected fields leads to no match.  

- **Get image from server**  
  - Type: Telegram node (file download)  
  - Role: Downloads photo file from Telegram using file ID from message.  
  - Configuration: Uses `$json.message.photo[1].file_id` to fetch the image.  
  - Input: Output from Switch node’s Image branch  
  - Output: Binary file data for AI processing  
  - Edge Cases: Image could be missing or corrupt; Telegram API errors; credential errors.

- **Get file from server**  
  - Type: Telegram node (file download)  
  - Role: Downloads document file (PDF) from Telegram using file ID.  
  - Configuration: Uses `$json.message.document.file_id` for the file.  
  - Input: Output from Switch node’s PDF branch  
  - Output: Binary file data for AI processing  
  - Edge Cases: Same as image node but for documents.

---

#### 2.3 AI-based Transaction Data Extraction

**Overview:**  
Uses Google Gemini AI models to extract structured transaction data from image or PDF binary inputs.

**Nodes Involved:**  
- Analyze the image or pdf (Google Gemini)  
- Parse Output (Structured output parser)  
- Parse details & add category (Langchain agent node)  
- Analyzer Scraper (Google Gemini LM Chat)  

**Node Details:**

- **Analyze the image or pdf**  
  - Type: Google Gemini AI (Langchain node)  
  - Role: Performs OCR and semantic extraction of transaction details from files.  
  - Configuration: Input is binary file; prompts specify extraction rules and expected fields (Billing Amount, Date, Payee, Bank ID).  
  - Input: Binary file from Telegram nodes  
  - Output: Raw parsed transaction data JSON  
  - Edge Cases: OCR errors, model timeouts, malformed files, inaccurate extraction.  
  - Credentials: Google Palm API credentials needed.

- **Parse Output**  
  - Type: Langchain output parser (structured)  
  - Role: Converts AI raw output into structured JSON objects with transaction fields.  
  - Configuration: JSON schema example defines transaction fields like Type, Amount, Date, Category.  
  - Input: AI output from “Analyze the image or pdf”  
  - Output: Parsed structured transaction array  
  - Edge Cases: Parsing fails if AI output deviates from schema.

- **Parse details & add category**  
  - Type: Langchain agent node  
  - Role: Further parses and categorizes transactions, adding fields like Category and Transaction Type based on rules.  
  - Configuration: Custom system message with parsing rules and category definitions.  
  - Input: Parsed output from “Parse Output” or directly from “Analyze the image or pdf” (depending on branch)  
  - Output: Fully structured and categorized transaction data  
  - Edge Cases: Misclassification, incomplete data, model errors.  
  - Credentials: Google Palm API required.

- **Analyzer Scraper**  
  - Type: Google Gemini LM Chat node  
  - Role: Invoked as part of AI processing chain, likely to assist in detail scraping or validation.  
  - Input: Not directly connected to initial steps; likely invoked conditionally or as fallback.  
  - Output: Feeds into “Parse details & add category.”

---

#### 2.4 Transaction Data Formatting

**Overview:**  
Transforms parsed transaction data into a consistent structure suitable for Firefly III API and prepares for batch processing.

**Nodes Involved:**  
- Make it pretty (Code node)  
- Loop for Multiple Items (Split in Batches)

**Node Details:**

- **Make it pretty**  
  - Type: Code node (JavaScript)  
  - Role: Normalizes transaction data fields, ensures presence of keys like Type, Date, Payee, Amount, Account, Description, Category.  
  - Configuration: Iterates over transactions; creates output array with standardized keys.  
  - Input: Output from “Parse details & add category”  
  - Output: Array of clean transaction objects  
  - Edge Cases: Missing fields handled with defaults; empty arrays cause no output.

- **Loop for Multiple Items**  
  - Type: SplitInBatches node  
  - Role: Processes transactions one by one for posting and confirmation.  
  - Configuration: Batch size defaults to 1, allowing sequential API calls per transaction.  
  - Input: Array from “Make it pretty”  
  - Output: Single transaction per iteration  
  - Edge Cases: Large batches slow down processing; state reset option disabled.

---

#### 2.5 Transaction Posting

**Overview:**  
Posts each transaction to Firefly III through its API and sends a confirmation message back to the Telegram chat.

**Nodes Involved:**  
- Post to Firefly (HTTP Request)  
- Confirmation Message (Telegram node)

**Node Details:**

- **Post to Firefly**  
  - Type: HTTP Request node  
  - Role: Posts transaction data to Firefly III API endpoint `/api/v1/transactions`.  
  - Configuration: Uses OAuth2 credentials for authentication; JSON body constructed with transaction fields mapped to Firefly schema.  
  - Input: Single transaction item from “Loop for Multiple Items”  
  - Output: Firefly API response confirming transaction creation  
  - Edge Cases: OAuth token expiry, API errors, malformed data, network timeouts.

- **Confirmation Message**  
  - Type: Telegram node (send message)  
  - Role: Sends a confirmation text “Transaction(s) recorded” back to the user in Telegram.  
  - Configuration: Uses chat ID from original Telegram message; forces reply UI in Telegram client.  
  - Input: After posting transaction to Firefly  
  - Output: Telegram message sent  
  - Edge Cases: Telegram API errors, invalid chat ID, message quota exceeded.

---

#### 2.6 Budget Report Generation

**Overview:**  
Generates a report of budget usage from Firefly III for the current month upon receiving the “Report” command via Telegram and delivers it as a CSV file.

**Nodes Involved:**  
- Get Remaining Budget (HTTP Request)  
- Parse JSON (Code node)  
- Pull required fields (Set node)  
- Make CSV (ConvertToFile node)  
- Send CSV (Telegram node)

**Node Details:**

- **Get Remaining Budget**  
  - Type: HTTP Request node  
  - Role: Fetches budget limits and usage from Firefly III API `/api/v1/budget-limits`.  
  - Configuration: OAuth2 authentication; query parameters specify date range from month start to current day.  
  - Input: Triggered by Switch node on “Report” or “report” text  
  - Output: JSON response with budgets and spending data  
  - Edge Cases: API errors, invalid date parameters, OAuth issues.

- **Parse JSON**  
  - Type: Code node  
  - Role: Parses Firefly API response, calculates totals, percent used, and organizes data by budget category.  
  - Configuration: Handles multiple possible input structures; outputs sorted array with totals row.  
  - Input: Response from “Get Remaining Budget”  
  - Output: Structured budget report data array  
  - Edge Cases: Unexpected API response formats, missing keys.

- **Pull required fields**  
  - Type: Set node  
  - Role: Selects and renames key fields (Bucket, Budget, Spent, Remaining) for CSV export.  
  - Input: Output from “Parse JSON”  
  - Output: Simplified data for CSV conversion

- **Make CSV**  
  - Type: ConvertToFile node  
  - Role: Converts data array into a CSV file, named dynamically with current month and day.  
  - Configuration: Filename uses expressions for date formatting  
  - Output: CSV binary file

- **Send CSV**  
  - Type: Telegram node (send document)  
  - Role: Sends the generated CSV file back to Telegram chat as a document.  
  - Configuration: Uses chat ID from original message; sends binary file with document operation.  
  - Edge Cases: Telegram API file size limits, invalid chat ID.

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                          | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                               |
|-------------------------------|---------------------------------------|----------------------------------------|--------------------------------------|--------------------------------------|---------------------------------------------------------------------------|
| Transaction Image or PDF Received | Telegram Trigger                     | Entry point: receives Telegram messages | None                                 | Switch                               | ## 1. Manual Trigger<br>Image, PDF or Text sent by user to Telegram channel |
| Switch                        | Switch                                | Routes based on message content type   | Transaction Image or PDF Received    | Get image from server, Get file from server, Get Remaining Budget (x2) |                                                                           |
| Get image from server         | Telegram node (file download)          | Downloads photo file from Telegram     | Switch (Image branch)                 | Analyze the image or pdf             | ## 2. Image & PDF Processing<br>Pulls the image or document from Telegram servers. Gemini analyzes and parses the relevant info |
| Get file from server          | Telegram node (file download)          | Downloads document file (PDF)           | Switch (PDF branch)                   | Analyze the image or pdf             | Same as Get image from server                                               |
| Analyze the image or pdf      | Google Gemini AI (Langchain node)      | Extracts raw transaction data from files | Get image from server, Get file from server | Parse Output, Parse details & add category |                                                                           |
| Parse Output                  | Langchain Output Parser (structured)  | Parses AI raw output into structured transactions | Analyze the image or pdf             | Parse details & add category         |                                                                           |
| Parse details & add category  | Langchain Agent node                   | Adds categorization and detailed parsing | Analyze the image or pdf, Parse Output | Make it pretty                      |                                                                           |
| Make it pretty                | Code node                             | Normalizes and formats transaction data | Parse details & add category         | Loop for Multiple Items              | ## 3. Formatting and posting<br>The info is formatted to fit the Firefly III requirements. Each transaction is posted 1 by 1, with a confirmation of each record sent back to the Telegram channel |
| Loop for Multiple Items       | SplitInBatches                        | Processes transactions individually    | Make it pretty                      | Post to Firefly, Confirmation Message |                                                                           |
| Post to Firefly               | HTTP Request                         | Posts transactions to Firefly III API  | Loop for Multiple Items              | Loop for Multiple Items              |                                                                           |
| Confirmation Message          | Telegram node (send message)           | Sends confirmation to Telegram chat    | Loop for Multiple Items              | None                               |                                                                           |
| Get Remaining Budget          | HTTP Request                         | Retrieves budget limits & spending     | Switch (Report/report branches)      | Parse JSON                         | ## 4. Bonus: Report Generation<br>Produces a budget report from the beginning of the month to now if 'report' is sent to the telegram chat |
| Parse JSON                   | Code node                            | Parses and transforms budget data      | Get Remaining Budget                 | Pull required fields                |                                                                           |
| Pull required fields          | Set node                            | Selects key budget fields for export   | Parse JSON                         | Make CSV                          |                                                                           |
| Make CSV                     | ConvertToFile                       | Converts budget data to CSV file        | Pull required fields                | Send CSV                         |                                                                           |
| Send CSV                     | Telegram node (send document)          | Sends CSV budget report to Telegram    | Make CSV                          | None                               |                                                                           |
| Analyzer Scraper             | Google Gemini LM Chat                  | Auxiliary AI chat analysis for scraping | Not directly connected in main flow | Parse details & add category         |                                                                           |
| Sticky Notes (various)       | Sticky Note                          | Provide explanations, instructions, and setup info | None                               | None                               | See detailed notes in Section 5                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Set to listen for message updates (photos, documents, text).  
   - Configure with your Telegram API credentials.

2. **Add Switch Node**  
   - Type: Switch  
   - Create four outputs with conditions:  
     - Photo exists: `$json.message.photo[0]` exists → "Image" branch  
     - Document exists: `$json.message.document` exists → "PDF" branch  
     - Text equals "Report" (case-sensitive) → "Report" branch  
     - Text equals "report" (case-insensitive) → "report" branch

3. **Add Telegram File Download Nodes**  
   - For Image branch:  
     - Telegram node to get file from server, using `{{$json.message.photo[1].file_id}}`  
     - Use Telegram API credentials  
   - For PDF branch:  
     - Telegram node to get file from server, using `{{$json.message.document.file_id}}`  
     - Use Telegram API credentials

4. **Add Google Gemini AI Node for Document Analysis**  
   - Type: Google Gemini (Langchain) node  
   - Input type: Binary (file)  
   - Insert prompt to extract transaction data fields (Amount, Date, Payee, Bank ID) with rules as per your Firefly III bank IDs.  
   - Connect outputs of Telegram file download nodes to this node.  
   - Configure with Google Palm API credentials and chosen Gemini model.

5. **Add Langchain Output Parser Node**  
   - Type: OutputParserStructured  
   - Provide JSON schema matching transaction structure.  
   - Connect it to the “Analyze the image or pdf” node.

6. **Add Langchain Agent Node for Parsing & Categorizing**  
   - Use Langchain Agent node with system message containing detailed parsing rules, categories, and transaction types.  
   - Connect output from the previous parser node.

7. **Add Code Node "Make it pretty"**  
   - JavaScript code to normalize and standardize transaction data fields.  
   - Connect from parsing node output.

8. **Add SplitInBatches Node**  
   - Batch size 1 to process transactions one at a time.  
   - Connect from code node.

9. **Add HTTP Request Node “Post to Firefly”**  
   - Method: POST  
   - URL: `http[s]://[your firefly url]/api/v1/transactions`  
   - Authentication: OAuth2 API with Firefly credentials  
   - JSON body: Map fields from transaction object to Firefly required fields (type, date, amount, description, category_name, source_id, destination_name)  
   - Connect from SplitInBatches node.

10. **Add Telegram Send Message Node “Confirmation Message”**  
    - Sends “Transaction(s) recorded” to the chat ID from the original Telegram message  
    - Connect from “Post to Firefly” node.

11. **For Report Branch (Report/report commands):**  
    - Add HTTP Request node “Get Remaining Budget”  
      - Method: GET  
      - URL: `http[s]://[your firefly url]/api/v1/budget-limits`  
      - OAuth2 credentials  
      - Query parameters: `start` = first day of current month, `end` = current day  
    - Add Code node “Parse JSON”  
      - Parses Firefly API response, calculates budget usage and totals  
    - Add Set node “Pull required fields”  
      - Extract Bucket, Budget, Spent, Remaining for CSV  
    - Add ConvertToFile node “Make CSV”  
      - Format: CSV  
      - Filename: `Budget report for {{ $now.monthLong }} 01 to {{ $now.day }}.csv`  
    - Add Telegram node “Send CSV”  
      - Sends CSV file as document to Telegram chat  
    - Connect all these nodes sequentially from the Switch node’s Report/report branch.

12. **Add any auxiliary AI nodes (Analyzer Scraper) as needed**  
    - Configure with Google Palm API credentials.

13. **Credential Setup:**  
    - Create and assign Telegram API credentials in all Telegram nodes.  
    - Create and assign Google Palm API credentials in AI nodes.  
    - Create and assign OAuth2 credentials for Firefly III API in HTTP nodes.

14. **Testing and Validation:**  
    - Test sending images and PDFs via Telegram.  
    - Test “Report” command text.  
    - Verify data appears correctly in Firefly III and that confirmation messages and reports are returned.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| **Automated Transaction Posting in Firefly III:** Workflow enables sending transaction images, PDFs, or commands to Telegram, which are parsed by Gemini AI and posted to Firefly III automatically. Sending "Report" generates a budget report for the month-to-date. Setup requires credentials for Telegram, Firefly OAuth2, and Gemini API. Customizable parsing rules and categories are provided. | Sticky Note in workflow.                                                                                                                  |
| How to create Firefly III OAuth2 client: https://docs.firefly-iii.org/how-to/firefly-iii/features/api/#create-an-oauth2-client                                                                                                                                                                                                                                                                                                                                                                                                                            | Important for OAuth2 API credential configuration.                                                                                        |
| Gemini AI Model selection and prompt customization tips included in nodes “Analyze the image or pdf” and “Parse details & add category”. Ensure Bank ID and categories match your Firefly III setup.                                                                                                                                                                                                                                                                                                                                                        | Important for accurate data extraction and categorization.                                                                               |
| Telegram API file limits and OAuth token expiry should be monitored to avoid workflow failures. Use retry policies and error handling as needed.                                                                                                                                                                                                                                                                                                                                                                                                        | Operational advice.                                                                                                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.