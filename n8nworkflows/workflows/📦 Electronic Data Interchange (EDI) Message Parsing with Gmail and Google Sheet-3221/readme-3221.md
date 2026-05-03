📦 Electronic Data Interchange (EDI) Message Parsing with Gmail and Google Sheet

https://n8nworkflows.xyz/workflows/---electronic-data-interchange--edi--message-parsing-with-gmail-and-google-sheet-3221


# 📦 Electronic Data Interchange (EDI) Message Parsing with Gmail and Google Sheet

### 1. Workflow Overview

This workflow automates the processing of Electronic Data Interchange (EDI) messages received via Gmail, specifically targeting small and medium businesses in supply chain, logistics, and transportation sectors. It detects incoming emails with "EDI" in the subject, extracts and parses the EDI message content, transforms the parsed data into a structured tabular format, and logs the results into Google Sheets for further analysis and record-keeping.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger on new Gmail emails and filter those containing "EDI" in the subject.
- **1.2 Email Content Extraction:** Retrieve the full email content and extract the raw EDI message body.
- **1.3 EDI Parsing:** Use a JavaScript code node to parse the raw EDI message into structured JSON data.
- **1.4 Data Transformation:** Flatten the parsed EDI data into a tabular format suitable for Google Sheets.
- **1.5 Conditional Routing and Storage:** Determine the order type (Return or Outbound) and append the data to the corresponding Google Sheet tab.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new email arrives in Gmail and filters emails to only process those with "EDI" in the subject line.

- **Nodes Involved:**  
  - Email Trigger  
  - Subject includes EDI (If node)

- **Node Details:**

  - **Email Trigger**  
    - Type: Gmail Trigger (Trigger node)  
    - Role: Watches Gmail inbox for new emails every minute.  
    - Configuration: Polling mode set to every minute, no additional filters.  
    - Input/Output: No input; outputs new email metadata.  
    - Edge Cases: Gmail API authentication errors, rate limits, or connectivity issues.  
    - Notes: Requires Gmail API credentials configured in n8n.

  - **Subject includes EDI**  
    - Type: If node  
    - Role: Filters incoming emails to continue only if the subject contains the substring "EDI" (case-sensitive).  
    - Configuration: Condition checks if `$json.Subject` contains "EDI".  
    - Input: Output from Email Trigger.  
    - Output: True branch proceeds to get full email; false branch ends workflow.  
    - Edge Cases: Emails without subject or malformed data may cause expression failures.

---

#### 2.2 Email Content Extraction

- **Overview:**  
  Retrieves the full email content using the message ID and extracts the raw EDI message body for parsing.

- **Nodes Involved:**  
  - Get Email  
  - Extract Body

- **Node Details:**

  - **Get Email**  
    - Type: Gmail (App node)  
    - Role: Fetches the full email content by message ID from the trigger.  
    - Configuration: Uses the message ID from the trigger (`{{$json.id}}`), operation set to "get", with `simple` mode disabled to get full content.  
    - Input: True branch output from "Subject includes EDI".  
    - Output: Full email JSON including body text.  
    - Edge Cases: Gmail API errors, missing message ID, or permission issues.

  - **Extract Body**  
    - Type: Set node  
    - Role: Extracts and cleans the email body text for parsing.  
    - Configuration: Assigns a new field `body` by cleaning `$json.text` (replaces escaped newlines with actual newlines, trims leading/trailing quotes).  
    - Input: Output from Get Email.  
    - Output: JSON with cleaned `body` field containing raw EDI message.  
    - Edge Cases: Emails without text content or unexpected formatting may cause parsing issues.

---

#### 2.3 EDI Parsing

- **Overview:**  
  Parses the raw EDI message string into a structured JSON object containing headers, order details, dates, parties, and line items.

- **Nodes Involved:**  
  - Parse EDI Message (Code node)  
  - Order Information (Set node)

- **Node Details:**

  - **Parse EDI Message**  
    - Type: Code (JavaScript)  
    - Role: Implements a custom parser for EDI messages, splitting by segments and extracting key data fields.  
    - Configuration:  
      - Parses segments like UNA, UNB (interchange header), UNH (message header), BGM (order details), DTM (dates), NAD (parties), LIN (line items), IMD (descriptions), QTY (quantities), PRI (prices).  
      - Builds a comprehensive JSON object with arrays for dates, parties, and line items.  
      - Summarizes key order info: document type (hardcoded as "Return Order"), document number, order date, line item count, total quantity.  
    - Key Expressions: Uses `$input.first().json.body` to get the EDI message.  
    - Input: Output from Extract Body.  
    - Output: Parsed EDI JSON object.  
    - Edge Cases: Missing or malformed EDI message throws error; unexpected segment formats may cause partial parsing.  
    - Version: Compatible with n8n 1.82.1 and above.

  - **Order Information**  
    - Type: Set node  
    - Role: Extracts summary fields from parsed EDI JSON for easy access downstream.  
    - Configuration: Assigns fields like `documentType`, `documentNumber`, `orderDate`, `lineItemCount`, `totalQuantity` from `summary` object in parsed JSON.  
    - Input: Output from Parse EDI Message.  
    - Output: JSON with summarized order info fields.  
    - Edge Cases: Missing summary fields if parsing failed.

---

#### 2.4 Data Transformation

- **Overview:**  
  Transforms the parsed hierarchical EDI JSON into a flattened tabular format, combining header, date, party, and line item details into rows suitable for Google Sheets.

- **Nodes Involved:**  
  - Flatten Data to Orderlines (Code node)  
  - Split Out by Line (SplitOut node)  
  - Order Info + Orderlines (Merge node)

- **Node Details:**

  - **Flatten Data to Orderlines**  
    - Type: Code (JavaScript)  
    - Role: Converts parsed EDI JSON into an array of flat objects, each representing a line item enriched with header, date, and party info.  
    - Configuration:  
      - Creates header fields prefixed with `header_`.  
      - Processes up to three dates (`date1_`, `date2_`, `date3_` prefixes).  
      - Processes up to four parties (`party1_` to `party4_` prefixes).  
      - For each line item, creates a flat row combining all info.  
      - If no line items, creates one row with header info only.  
    - Input: Output from Parse EDI Message.  
    - Output: JSON with a `data` array of flattened rows.  
    - Edge Cases: Invalid input format throws error; missing arrays handled gracefully.

  - **Split Out by Line**  
    - Type: SplitOut  
    - Role: Splits the `data` array into individual items for merging.  
    - Configuration: Splits on field `data`.  
    - Input: Output from Flatten Data to Orderlines.  
    - Output: Individual JSON objects per line item.  
    - Edge Cases: Empty data array results in no output items.

  - **Order Info + Orderlines**  
    - Type: Merge  
    - Role: Combines order summary info and individual line items into a single stream for conditional routing.  
    - Configuration: Mode set to "combineBySql" (default merge mode).  
    - Input: Two inputs — from Order Information (summary) and Split Out by Line (line items).  
    - Output: Combined JSON objects with full order and line item data.  
    - Edge Cases: Mismatched input counts may cause incomplete merges.

---

#### 2.5 Conditional Routing and Storage

- **Overview:**  
  Routes the combined data based on order type and appends it to the appropriate Google Sheet tab (Return Orders or Outbound Orders).

- **Nodes Involved:**  
  - Order Type (If node)  
  - Return Orders (Google Sheets node)  
  - Outbound Orders (Google Sheets node)

- **Node Details:**

  - **Order Type**  
    - Type: If node  
    - Role: Checks if the `documentType` field equals "Return Order".  
    - Configuration: Strict equality check on `$json.documentType`.  
    - Input: Output from Order Info + Orderlines.  
    - Output: True branch to Return Orders, False branch to Outbound Orders.  
    - Edge Cases: Missing or unexpected `documentType` values may misroute data.

  - **Return Orders**  
    - Type: Google Sheets (Append operation)  
    - Role: Appends data rows to the "Return Orders" sheet tab in the specified Google Sheet.  
    - Configuration:  
      - Google Sheets API credentials required.  
      - Document ID set to a specific Google Sheet file.  
      - Sheet name selected by ID (numeric).  
      - Columns mapped automatically from input data fields.  
      - Operation: Append rows.  
    - Input: True branch from Order Type.  
    - Output: Google Sheets append result.  
    - Edge Cases: API authentication errors, sheet access issues, data type mismatches.

  - **Outbound Orders**  
    - Type: Google Sheets (Append operation)  
    - Role: Appends data rows to the "Outbound Orders" sheet tab in the same Google Sheet file.  
    - Configuration: Same as Return Orders but targeting a different sheet tab.  
    - Input: False branch from Order Type.  
    - Output: Google Sheets append result.  
    - Edge Cases: Same as Return Orders.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                          |
|---------------------|---------------------|----------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Email Trigger       | Gmail Trigger       | Triggers workflow on new Gmail email   | —                       | Subject includes EDI      | ### 1. Workflow Trigger with Gmail Trigger: Trigger on new emails with "EDI" in subject. Setup Gmail API credentials. [Learn more](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger) |
| Subject includes EDI | If                  | Filters emails with "EDI" in subject   | Email Trigger           | Get Email                |                                                                                                    |
| Get Email           | Gmail                | Retrieves full email content            | Subject includes EDI    | Extract Body             | ### 2. Get Email Body & Parse EDI Message: Setup Gmail API credentials. [Learn more](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail) |
| Extract Body        | Set                  | Extracts and cleans email body text     | Get Email               | Parse EDI Message        |                                                                                                    |
| Parse EDI Message   | Code (JavaScript)     | Parses raw EDI message into structured JSON | Extract Body            | Order Information, Flatten Data to Orderlines |                                                                                                    |
| Order Information   | Set                  | Extracts summary order info             | Parse EDI Message       | Order Info + Orderlines  |                                                                                                    |
| Flatten Data to Orderlines | Code (JavaScript) | Flattens parsed EDI data for tabular format | Parse EDI Message       | Split Out by Line        | ### 3. Process Parsed Data: Formats orderlines for Google Sheets. No setup needed.                 |
| Split Out by Line   | SplitOut              | Splits flattened data array into items | Flatten Data to Orderlines | Order Info + Orderlines  |                                                                                                    |
| Order Info + Orderlines | Merge               | Combines order summary and line items  | Order Information, Split Out by Line | Order Type              |                                                                                                    |
| Order Type          | If                    | Routes data based on order type         | Order Info + Orderlines | Return Orders, Outbound Orders |                                                                                                    |
| Return Orders       | Google Sheets         | Appends return orders to Google Sheet   | Order Type (true branch) | —                        | ### 4. Store the Transactions in a Google Sheet: Setup Google Sheets API credentials, select file and sheet. [Learn more](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Outbound Orders     | Google Sheets         | Appends outbound orders to Google Sheet | Order Type (false branch) | —                        |                                                                                                    |
| Sticky Note         | Sticky Note           | Provides example EDI message             | —                       | —                        | ### Example of EDI Message: Send yourself this email to test the workflow.                         |
| Sticky Note1        | Sticky Note           | Explains Gmail Trigger setup             | —                       | —                        | ### 1. Workflow Trigger with Gmail Trigger: Setup Gmail API credentials.                           |
| Sticky Note2        | Sticky Note           | Explains email body extraction and parsing | —                       | —                        | ### 2. Get Email Body & Parse EDI Message: Setup Gmail API credentials.                            |
| Sticky Note3        | Sticky Note           | Provides tutorial link and guide image   | —                       | —                        | ### 5. Do you need more details? Step-by-step guide and video tutorial available.                  |
| Sticky Note4        | Sticky Note           | Explains Google Sheets storage setup     | —                       | —                        | ### 4. Store the Transactions in a Google Sheet: Setup Google Sheets API credentials.              |
| Sticky Note5        | Sticky Note           | Explains parsed data processing           | —                       | —                        | ### 3. Process Parsed Data: Formats orderlines for Google Sheets. No setup needed.                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Credentials: Configure Gmail API credentials.  
   - Settings: Poll every minute.  
   - Purpose: Trigger workflow on new emails.

2. **Add If Node "Subject includes EDI"**  
   - Type: If  
   - Condition: Check if `$json.Subject` contains the string "EDI" (case-sensitive).  
   - Connect: Gmail Trigger → Subject includes EDI (true branch).

3. **Add Gmail Node "Get Email"**  
   - Type: Gmail  
   - Credentials: Use same Gmail API credentials.  
   - Operation: Get full email by message ID.  
   - Message ID: Set to `{{$json.id}}` from trigger.  
   - Connect: Subject includes EDI (true branch) → Get Email.

4. **Add Set Node "Extract Body"**  
   - Type: Set  
   - Add field `body` with expression:  
     `{{$json.text.replace(/\\n/g, '\n').replace(/^'|'$/g, '')}}`  
   - Connect: Get Email → Extract Body.

5. **Add Code Node "Parse EDI Message"**  
   - Type: Code (JavaScript)  
   - Paste the provided EDI parsing JavaScript code (see node details in section 2.3).  
   - Input: Use `body` field from Extract Body.  
   - Connect: Extract Body → Parse EDI Message.

6. **Add Set Node "Order Information"**  
   - Type: Set  
   - Assign fields from parsed JSON summary:  
     - `documentType` = `{{$json.summary.documentType}}`  
     - `documentNumber` = `{{$json.summary.documentNumber}}`  
     - `orderDate` = `{{$json.summary.orderDate}}`  
     - `lineItemCount` = `{{$json.summary.lineItemCount}}`  
     - `totalQuantity` = `{{$json.summary.totalQuantity}}`  
   - Connect: Parse EDI Message → Order Information.

7. **Add Code Node "Flatten Data to Orderlines"**  
   - Type: Code (JavaScript)  
   - Paste the provided flattening JavaScript code (see node details in section 2.4).  
   - Input: Parsed EDI JSON from Parse EDI Message.  
   - Connect: Parse EDI Message → Flatten Data to Orderlines.

8. **Add SplitOut Node "Split Out by Line"**  
   - Type: SplitOut  
   - Field to split: `data` (from Flatten Data to Orderlines output).  
   - Connect: Flatten Data to Orderlines → Split Out by Line.

9. **Add Merge Node "Order Info + Orderlines"**  
   - Type: Merge  
   - Mode: Combine by SQL (default).  
   - Connect two inputs:  
     - Input 1: Order Information  
     - Input 2: Split Out by Line  
   - Connect: Order Information → Merge (input 1), Split Out by Line → Merge (input 2).

10. **Add If Node "Order Type"**  
    - Type: If  
    - Condition: Check if `$json.documentType` equals "Return Order" (case-sensitive).  
    - Connect: Order Info + Orderlines → Order Type.

11. **Add Google Sheets Node "Return Orders"**  
    - Type: Google Sheets  
    - Credentials: Configure Google Sheets API credentials.  
    - Operation: Append rows.  
    - Document ID: Select or paste your Google Sheet file ID.  
    - Sheet Name: Select the sheet tab for Return Orders.  
    - Columns: Use auto-mapping from input data.  
    - Connect: Order Type (true branch) → Return Orders.

12. **Add Google Sheets Node "Outbound Orders"**  
    - Type: Google Sheets  
    - Credentials: Same as above.  
    - Operation: Append rows.  
    - Document ID: Same Google Sheet file.  
    - Sheet Name: Select the sheet tab for Outbound Orders.  
    - Columns: Auto-mapping.  
    - Connect: Order Type (false branch) → Outbound Orders.

13. **Add Sticky Notes** (optional but recommended)  
    - Add notes explaining each block and setup instructions as per the original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Sustainable and Efficient supply chains with N8N!                                                             | Workflow purpose and branding.                                                                   |
| Example EDI message included for testing the workflow.                                                        | See Sticky Note with example EDI message.                                                       |
| Step-by-step video tutorial available for detailed setup and explanation.                                     | [Watch Tutorial](https://youtu.be/-phwXeYk7Es)                                                  |
| Blog article explaining Electronic Data Interchange (EDI) basics and applications.                            | [Blog Article about Electronic Data Interchange (EDI)](https://www.samirsaci.com/what-is-edi-electronic-data-interchange/) |
| Workflow created with n8n version 1.82.1, submitted March 19th, 2025.                                         | Version compatibility note.                                                                     |
| Google API credentials required: Gmail API, Google Sheets API, Google Drive API.                              | Prerequisite for workflow operation.                                                            |
| No paid subscription required; uses free Google and Gmail APIs.                                              | Cost and accessibility note.                                                                     |
| Connect with Samir Saci on LinkedIn for logistics and supply chain automation insights.                        | [LinkedIn Profile](www.linkedin.com/in/samir-saci)                                              |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling users and automation agents to reproduce, modify, and troubleshoot the EDI message parsing and logging process effectively.