Parse Incoming Invoices From Outlook using AI Document Understanding

https://n8nworkflows.xyz/workflows/parse-incoming-invoices-from-outlook-using-ai-document-understanding-3396


# Parse Incoming Invoices From Outlook using AI Document Understanding

### 1. Workflow Overview

This workflow automates the extraction and processing of invoice data from emails received in an Outlook mailbox, leveraging AI document understanding and Excel for data storage. It is designed primarily for accounts receivable teams to reduce manual labor by automatically identifying invoice emails, validating invoice attachments, extracting invoice details using AI, and appending the results to an Excel workbook.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Email Retrieval:** Periodically fetch recent emails with attachments from a specified Outlook mailbox folder.
- **1.2 Email Classification:** Use AI to classify whether each email is an invoice-related message.
- **1.3 Attachment Processing:** Download attachments, split them, and classify each attachment to confirm if it is an invoice.
- **1.4 Invoice Data Extraction:** Extract detailed invoice information from confirmed invoice attachments using AI.
- **1.5 Data Output:** Append the extracted invoice data to an Excel worksheet for further use or review.
- **1.6 Control and Flow Management:** Includes batching, conditional checks, and wait nodes to manage processing flow and error handling.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Email Retrieval

**Overview:**  
This block triggers the workflow on a schedule and retrieves recent emails with attachments from a specific Outlook folder.

**Nodes Involved:**  
- Schedule Trigger  
- Get Recent Messages  
- Markdown  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every hour.  
  - Config: Interval set to every 1 hour.  
  - Inputs: None (trigger node).  
  - Outputs: Connects to Get Recent Messages.  
  - Edge Cases: If schedule misfires or is disabled, no emails will be processed.

- **Get Recent Messages**  
  - Type: Microsoft Outlook (Get All Messages)  
  - Role: Fetches emails received in the last hour with attachments from a specified folder (Accounts receivable mailbox).  
  - Config: Filters include receivedAfter = current time minus 1 hour, hasAttachments=true, folder ID specified.  
  - Credentials: Microsoft Outlook OAuth2 account configured to target mailbox.  
  - Inputs: Trigger from Schedule Trigger.  
  - Outputs: Connects to Markdown node.  
  - Edge Cases: Authentication errors, folder ID incorrect, no emails found.

- **Markdown**  
  - Type: Markdown  
  - Role: Converts email body HTML content to markdown format for easier text processing.  
  - Config: Converts `body.content` field to markdown stored in `markdown` key.  
  - Inputs: Emails from Get Recent Messages.  
  - Outputs: Connects to Message Classifier.  
  - Edge Cases: Malformed HTML bodies could cause conversion issues.

---

#### 1.2 Email Classification

**Overview:**  
Classifies each email to determine if it is an invoice message using AI text classification.

**Nodes Involved:**  
- Message Classifier  
- For Each Message  
- Message Ref  

**Node Details:**  

- **Message Classifier**  
  - Type: Langchain Text Classifier  
  - Role: Uses AI to classify emails as "invoice" or "other" based on sender, subject, and message content.  
  - Config: Input text composed of sender email, subject, and markdown message content. Categories include "invoice" with fallback "other".  
  - Inputs: Markdown node output.  
  - Outputs: Two outputs: main (classified messages) and empty (non-invoice).  
  - Edge Cases: Misclassification due to ambiguous email content; fallback ensures non-invoice messages are ignored.

- **For Each Message**  
  - Type: Split In Batches  
  - Role: Processes each classified invoice message individually to manage resource usage.  
  - Config: Default batch size (1).  
  - Inputs: From Message Classifier (invoice messages).  
  - Outputs: Two outputs: main to Excel node, secondary to Message Ref.  
  - Edge Cases: Large batch sizes could cause timeouts or memory issues.

- **Message Ref**  
  - Type: No Operation (NoOp)  
  - Role: Holds reference to the current message for later use, especially to merge with extracted data.  
  - Inputs: From For Each Message.  
  - Outputs: Connects to Download Attachments.  
  - Edge Cases: None.

---

#### 1.3 Attachment Processing

**Overview:**  
Downloads attachments from each invoice email, splits them into individual files, and classifies each attachment to confirm if it is an invoice document.

**Nodes Involved:**  
- Download Attachments  
- Split Attachments  
- Extract from File  
- Invoice Classifier With Gemini 2.0  
- Filter Invoices  
- Has Invoice?  

**Node Details:**  

- **Download Attachments**  
  - Type: Microsoft Outlook  
  - Role: Downloads all attachments from the current email message.  
  - Config: Operation "get" with downloadAttachments=true, messageId from Message Ref.  
  - Credentials: Microsoft Outlook OAuth2.  
  - Inputs: From Message Ref.  
  - Outputs: Connects to Split Attachments.  
  - Edge Cases: Attachment download failures, large attachments causing delays.

- **Split Attachments**  
  - Type: Code  
  - Role: Splits multiple attachments into separate items for individual processing.  
  - Config: JavaScript loops over all binary attachments, outputs each as a separate item with filename and binary data.  
  - Inputs: From Download Attachments.  
  - Outputs: Connects to Extract from File.  
  - Edge Cases: Empty attachments, binary data corruption.

- **Extract from File**  
  - Type: Extract From File  
  - Role: Converts binary attachment data to property data (e.g., base64 or text) for AI processing.  
  - Config: Operation "binaryToProperty".  
  - Inputs: From Split Attachments.  
  - Outputs: Connects to Invoice Classifier With Gemini 2.0.  
  - Edge Cases: Unsupported file types, conversion errors.

- **Invoice Classifier With Gemini 2.0**  
  - Type: HTTP Request  
  - Role: Sends attachment data to Google Gemini AI to classify if the document is an invoice issued to the company by a supplier.  
  - Config: POST request to Gemini API with inline data (mime type and base64 data) and prompt text. Uses generationConfig to request JSON response with boolean fields `is_invoice` and `is_issued_to_company`.  
  - Credentials: Google Gemini (PaLM) API.  
  - Inputs: From Extract from File.  
  - Outputs: Connects to Filter Invoices.  
  - Edge Cases: API rate limits, malformed responses, network errors.

- **Filter Invoices**  
  - Type: Filter  
  - Role: Filters attachments to only those classified as invoices issued to the company.  
  - Config: Condition checks that both `is_invoice` and `is_issued_to_company` are true in the Gemini response JSON.  
  - Inputs: From Invoice Classifier With Gemini 2.0.  
  - Outputs: Connects to Has Invoice? (true path).  
  - Edge Cases: False negatives/positives from AI classification.

- **Has Invoice?**  
  - Type: If  
  - Role: Checks if filtered invoice data exists to proceed with extraction or send empty response.  
  - Config: Condition checks if input JSON is not empty.  
  - Inputs: From Filter Invoices.  
  - Outputs: True path to File-Based OCR with Gemini 2.0, false path to Empty Response.  
  - Edge Cases: Empty or malformed data.

---

#### 1.4 Invoice Data Extraction

**Overview:**  
Extracts detailed invoice information from confirmed invoice attachments using Google Gemini AI and merges the output with the original email reference.

**Nodes Involved:**  
- File-Based OCR with Gemini 2.0  
- Parse Output  
- Empty Response  
- Wait  

**Node Details:**  

- **File-Based OCR with Gemini 2.0**  
  - Type: HTTP Request  
  - Role: Sends attachment data to Google Gemini AI to extract detailed invoice fields such as invoice number, dates, amounts, supplier info, services, etc.  
  - Config: POST request to Gemini API with inline data and prompt text specifying extraction role. Uses generationConfig with detailed JSON schema for invoice fields.  
  - Credentials: Google Gemini (PaLM) API.  
  - Inputs: From Has Invoice? (true path).  
  - Outputs: Connects to Parse Output (success) and Empty Response (on error, continue error output enabled).  
  - Edge Cases: API errors, incomplete extraction, network issues.

- **Parse Output**  
  - Type: Set  
  - Role: Parses Gemini JSON response and merges it with the original email metadata (excluding email body) for structured output.  
  - Config: Uses expression to parse JSON from Gemini response and combines with first item from Message Ref node.  
  - Inputs: From File-Based OCR with Gemini 2.0.  
  - Outputs: Connects to Wait.  
  - Edge Cases: JSON parse errors, missing fields.

- **Empty Response**  
  - Type: Set  
  - Role: Outputs a null invoice with email metadata when no invoice data is extracted or on error.  
  - Config: Sets invoice to null and merges with email metadata from Message Ref.  
  - Inputs: From Has Invoice? (false path) and File-Based OCR with Gemini 2.0 (on error).  
  - Outputs: Connects to Wait.  
  - Edge Cases: None.

- **Wait**  
  - Type: Wait  
  - Role: Introduces a 1-second delay before processing the next message batch to avoid API rate limits or overload.  
  - Config: Wait time set to 1 second.  
  - Inputs: From Parse Output and Empty Response.  
  - Outputs: Connects back to For Each Message to continue processing.  
  - Edge Cases: None.

---

#### 1.5 Data Output

**Overview:**  
Appends the extracted invoice data rows to a specified Excel workbook worksheet.

**Nodes Involved:**  
- For Each Message (second output)  
- Microsoft Excel 365  

**Node Details:**  

- **For Each Message** (second output)  
  - Type: Split In Batches (same node as above)  
  - Role: Provides the original message reference to the Excel node for appending data.  
  - Inputs: From Message Classifier.  
  - Outputs: Connects to Microsoft Excel 365.  
  - Edge Cases: None.

- **Microsoft Excel 365**  
  - Type: Microsoft Excel  
  - Role: Appends extracted invoice data as a new row into a specified Excel workbook and worksheet.  
  - Config: Operation "append" on a workbook and worksheet identified by IDs.  
  - Credentials: Microsoft Excel OAuth2 account.  
  - Inputs: From For Each Message.  
  - Outputs: None (end node).  
  - Edge Cases: Authentication errors, workbook or worksheet ID invalid, data mapping errors.

---

#### 1.6 Control and Flow Management

**Overview:**  
Manages workflow execution flow, batching, and error handling.

**Nodes Involved:**  
- No additional nodes beyond those described above; control is embedded in Split In Batches, If, and Wait nodes.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                              | Input Node(s)              | Output Node(s)                | Sticky Note                                                                                                          |
|-------------------------------|----------------------------------|----------------------------------------------|----------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger                 | Initiates workflow every hour                 | None                       | Get Recent Messages           | ## Try it out... (overview and usage instructions)                                                                    |
| Get Recent Messages           | Microsoft Outlook                | Fetch recent emails with attachments          | Schedule Trigger            | Markdown                     |                                                                                                                      |
| Markdown                     | Markdown                        | Converts email body HTML to markdown          | Get Recent Messages         | Message Classifier           |                                                                                                                      |
| Message Classifier           | Langchain Text Classifier       | Classifies emails as invoice or other         | Markdown                   | For Each Message             | ## 1. Check for Invoice Emails (explains AI classification benefits)                                                  |
| For Each Message             | Split In Batches                | Processes each invoice email individually     | Message Classifier          | Microsoft Excel 365, Message Ref |                                                                                                                      |
| Message Ref                 | NoOp                           | Holds message reference for downstream nodes | For Each Message            | Download Attachments         |                                                                                                                      |
| Download Attachments         | Microsoft Outlook               | Downloads attachments from email               | Message Ref                 | Split Attachments            | ## 2. Classify If Attachment is Invoice (explains attachment classification with Gemini)                             |
| Split Attachments            | Code                           | Splits multiple attachments into separate items | Download Attachments        | Extract from File            |                                                                                                                      |
| Extract from File            | Extract From File              | Converts binary attachment to property data   | Split Attachments           | Invoice Classifier With Gemini 2.0 |                                                                                                                      |
| Invoice Classifier With Gemini 2.0 | HTTP Request                   | Classifies if attachment is invoice document  | Extract from File           | Filter Invoices              |                                                                                                                      |
| Filter Invoices             | Filter                         | Filters only attachments classified as invoices | Invoice Classifier With Gemini 2.0 | Has Invoice?                 |                                                                                                                      |
| Has Invoice?                | If                             | Checks if invoice data exists                  | Filter Invoices             | File-Based OCR with Gemini 2.0, Empty Response |                                                                                                                      |
| File-Based OCR with Gemini 2.0 | HTTP Request                   | Extracts detailed invoice data from attachment | Has Invoice? (true)         | Parse Output, Empty Response | ## 3. Extract Invoice Details (explains detailed extraction with Gemini API)                                         |
| Parse Output                | Set                            | Parses and merges extracted invoice data       | File-Based OCR with Gemini 2.0 | Wait                        |                                                                                                                      |
| Empty Response              | Set                            | Outputs null invoice when no data extracted    | Has Invoice? (false), File-Based OCR with Gemini 2.0 (error) | Wait                        |                                                                                                                      |
| Wait                       | Wait                           | Delays next batch processing by 1 second       | Parse Output, Empty Response | For Each Message             |                                                                                                                      |
| Microsoft Excel 365          | Microsoft Excel                | Appends invoice data to Excel workbook          | For Each Message            | None                        | ## 4. Upload to Excel Workbook (explains Excel node usage)                                                           |
| Sticky Note1                | Sticky Note                   | Notes on attachment classification block       | None                       | None                        | ## 2. Classify If Attachment is Invoice (with helpful links)                                                         |
| Sticky Note2                | Sticky Note                   | Notes on invoice extraction block               | None                       | None                        | ## 3. Extract Invoice Details (with helpful links)                                                                    |
| Sticky Note3                | Sticky Note                   | Notes on Excel upload block                      | None                       | None                        | ## 4. Upload to Excel Workbook (with helpful links)                                                                   |
| Sticky Note4                | Sticky Note                   | General workflow overview and usage instructions | None                       | None                        | ## Try it out (full workflow overview, usage, and help links)                                                        |
| Sticky Note5                | Sticky Note                   | Notes on possible workflow extensions            | None                       | None                        | ### Where Next? It's Up to You! (suggestions for extending workflow)                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 1 hour.

2. **Create Microsoft Outlook Node (Get Recent Messages)**  
   - Operation: Get All  
   - Filters: Received after current time minus 1 hour, hasAttachments=true, specify folder ID for Accounts receivable mailbox.  
   - Credentials: Configure Microsoft Outlook OAuth2 with access to mailbox.

3. **Create Markdown Node**  
   - Input: `body.content` from Outlook messages  
   - Convert HTML to markdown, output to key `markdown`.

4. **Create Langchain Text Classifier Node (Message Classifier)**  
   - Input Text: Compose from sender email, subject, and markdown content.  
   - Categories: Define "invoice" category with fallback "other".  
   - Use AI classification to detect invoice emails.

5. **Create Split In Batches Node (For Each Message)**  
   - Batch size: default (1)  
   - Input: Messages classified as invoice.

6. **Create NoOp Node (Message Ref)**  
   - Purpose: Hold message reference for downstream nodes.

7. **Create Microsoft Outlook Node (Download Attachments)**  
   - Operation: Get message by ID from Message Ref node.  
   - Enable downloadAttachments option.  
   - Credentials: Same Microsoft Outlook OAuth2.

8. **Create Code Node (Split Attachments)**  
   - JavaScript to iterate over all binary attachments and output each as separate item with filename and binary data.

9. **Create Extract From File Node**  
   - Operation: binaryToProperty  
   - Input: Each attachment from Split Attachments.

10. **Create HTTP Request Node (Invoice Classifier With Gemini 2.0)**  
    - Method: POST  
    - URL: Google Gemini API endpoint for content generation.  
    - Body: JSON with inline data (mime type and base64 data) and prompt to classify if document is invoice issued to company.  
    - Use generationConfig to request JSON response with boolean fields `is_invoice` and `is_issued_to_company`.  
    - Credentials: Google Gemini (PaLM) API.

11. **Create Filter Node (Filter Invoices)**  
    - Condition: Both `is_invoice` and `is_issued_to_company` must be true.

12. **Create If Node (Has Invoice?)**  
    - Condition: Input JSON is not empty.

13. **Create HTTP Request Node (File-Based OCR with Gemini 2.0)**  
    - Method: POST  
    - URL: Google Gemini API endpoint for content generation.  
    - Body: JSON with inline data and prompt to extract detailed invoice fields (invoice number, dates, amounts, supplier info, services, etc.).  
    - Use generationConfig with detailed JSON schema for invoice fields.  
    - Credentials: Google Gemini (PaLM) API.  
    - On error: continue workflow with error output.

14. **Create Set Node (Parse Output)**  
    - Parse Gemini JSON response from OCR node.  
    - Merge with original email metadata from Message Ref (excluding email body).

15. **Create Set Node (Empty Response)**  
    - Output JSON with `invoice: null` and email metadata from Message Ref.

16. **Create Wait Node**  
    - Delay: 1 second.  
    - Inputs: From Parse Output and Empty Response.  
    - Output: Connect back to For Each Message to continue processing.

17. **Connect For Each Message (second output) to Microsoft Excel Node**  
    - Create Microsoft Excel node.  
    - Operation: Append row to specified workbook and worksheet by ID.  
    - Credentials: Microsoft Excel OAuth2 account.  
    - Map extracted invoice data fields to Excel columns as needed.

18. **Add Sticky Notes**  
    - Add descriptive sticky notes near logical blocks for documentation and user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini AI for both classification and detailed invoice data extraction, leveraging its unique ability to process PDF documents directly.                                                                        | https://n8n.io/workflows/2421-transcribing-bank-statements-to-markdown-using-gemini-vision-ai/         |
| The Outlook node fetches emails with a date filter because the Outlook Trigger node does not support date/time filters, making this approach more efficient for invoice processing.                                                        | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook                      |
| The Excel node appends invoice data to a worksheet, allowing human review before integration into accounting systems.                                                                                                                   | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftexcel/                       |
| For large volumes or diverse invoice formats, consider adding document conversion nodes or human feedback mechanisms such as tagging emails for processed invoices.                                                                       |                                                                                                       |
| Join the n8n community for support and discussion: Discord and Forum links provided.                                                                                                                                                      | Discord: https://discord.com/invite/XPKeKXeB7d, Forum: https://community.n8n.io/                        |

---

This document provides a comprehensive reference to understand, reproduce, and extend the "Parse Incoming Invoices From Outlook using AI Document Understanding" workflow. It covers all nodes, their configurations, interconnections, and potential failure points, enabling advanced users and AI agents to confidently work with this automation.