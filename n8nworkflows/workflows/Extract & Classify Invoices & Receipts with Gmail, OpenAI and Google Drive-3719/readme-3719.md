Extract & Classify Invoices & Receipts with Gmail, OpenAI and Google Drive

https://n8nworkflows.xyz/workflows/extract---classify-invoices---receipts-with-gmail--openai-and-google-drive-3719


# Extract & Classify Invoices & Receipts with Gmail, OpenAI and Google Drive

### 1. Workflow Overview

This workflow automates the extraction, classification, and organization of invoices and receipts from Gmail emails, leveraging OpenAI for AI-based document classification and Google Drive for storage. It is designed primarily for small business owners and freelancers who want to streamline their financial document management.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Folder Setup:** Receives a webhook trigger with parameters specifying the date range and email sending preference, then creates a dated Google Drive folder for storing matched documents.
- **1.2 Email Retrieval and Filtering:** Fetches emails with attachments from Gmail within the specified date range, optionally filters emails, and isolates PDF attachments.
- **1.3 PDF Content Extraction and Token Limit Check:** Reads the text content of PDF attachments and ensures the content size is within OpenAI token limits.
- **1.4 AI Classification:** Uses OpenAI to classify each PDF as a receipt or invoice (or other user-defined category).
- **1.5 Upload and Optional Email Dispatch:** Uploads matched PDFs to the created Google Drive folder and optionally sends an email with all matched documents attached to a specified recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Folder Setup

- **Overview:**  
  This block triggers the workflow via a webhook, receives input parameters (date range and email sending flag), creates a Google Drive folder named by the start date, and responds to the webhook with the folder URL.

- **Nodes Involved:**  
  - Webhook  
  - Create folder (Google Drive)  
  - Configure (Set node)  
  - Respond to Webhook  

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook Trigger  
    - Role: Entry point; listens for POST requests with JSON body containing `startDate`, `endDate`, and `sendEmail`.  
    - Configuration: Path set to a unique webhook ID; expects header authentication.  
    - Inputs: External HTTP POST request  
    - Outputs: JSON body forwarded downstream  
    - Edge Cases: Authentication failure, malformed JSON, missing required fields.

  - **Create folder**  
    - Type: Google Drive node (folder creation)  
    - Role: Creates a folder named `invoices_<startDate>` in Google Drive root folder.  
    - Configuration: Folder name dynamically set using `startDate` from webhook body; uses Google Drive OAuth2 credentials.  
    - Inputs: JSON from Webhook node  
    - Outputs: Folder metadata including folder ID  
    - Edge Cases: API quota limits, permission errors, invalid folder name.

  - **Configure**  
    - Type: Set node  
    - Role: Defines workflow parameters such as token limits, classification keywords, Google Drive folder URL, recipient email, and email sending flag.  
    - Configuration:  
      - `maxTokenSize`: 8000 (limits PDF text tokens sent to OpenAI)  
      - `replyTokenSize`: 50 (reserved tokens for OpenAI reply)  
      - `Match on`: Classification keyword phrase (default: "receipt or invoice that can be considered a software engineering business cost")  
      - `Google Drive folder to upload matched PDFs`: URL constructed dynamically from created folder ID  
      - `sendInvoicesTo`: Recipient email address (empty by default)  
      - `sendEmail`: Boolean, derived from webhook body `sendEmail` field  
    - Inputs: Folder metadata from Create folder node  
    - Outputs: Configured parameters for downstream nodes  
    - Edge Cases: Missing or invalid parameter values.

  - **Respond to Webhook**  
    - Type: Respond to Webhook node  
    - Role: Sends HTTP 202 Accepted response with JSON containing status and Google Drive folder URL.  
    - Configuration: Response code 202; response body includes folder URL constructed from folder ID.  
    - Inputs: Folder metadata from Create folder node  
    - Outputs: HTTP response to webhook caller  
    - Edge Cases: Network issues, delayed response.

---

#### 2.2 Email Retrieval and Filtering

- **Overview:**  
  Retrieves all emails with attachments from Gmail within the specified date range, optionally filters out emails sent to specific addresses, and prepares attachments for processing.

- **Nodes Involved:**  
  - Get emails with attachments (Gmail)  
  - Optional filter for emails (Filter)  
  - Iterate over email attachments (Code)  

- **Node Details:**

  - **Get emails with attachments**  
    - Type: Gmail node (getAll operation)  
    - Role: Fetches all emails with attachments received between `startDate` and `endDate`.  
    - Configuration:  
      - Query: `has:attachment`  
      - Filters: `receivedAfter` and `receivedBefore` set dynamically from webhook body  
      - Downloads attachments with prefix `attachment_`  
      - Returns all matching emails  
      - Uses Gmail OAuth2 credentials  
    - Inputs: Parameters from Configure node  
    - Outputs: Emails with attachments  
    - Edge Cases: Gmail API rate limits, authentication errors, empty mailbox.

  - **Optional filter for emails**  
    - Type: Filter node  
    - Role: Optionally excludes emails sent to a specific address (empty string by default, effectively no filtering).  
    - Configuration: Filters emails where `to.value[0].address` is not equal to an empty string (can be customized).  
    - Inputs: Emails from Gmail node  
    - Outputs: Filtered emails  
    - Edge Cases: Missing or malformed email address fields.

  - **Iterate over email attachments**  
    - Type: Code node (JavaScript)  
    - Role: Iterates through all binary attachments in each email, flattening them into individual items for processing.  
    - Configuration: Custom JS code loops over all input items and their binary keys, returning separate items each containing one attachment binary data.  
    - Inputs: Filtered emails  
    - Outputs: Individual attachment items  
    - Edge Cases: Emails without attachments, binary data missing or corrupted.

---

#### 2.3 PDF Content Extraction and Token Limit Check

- **Overview:**  
  Filters attachments to PDFs, extracts their text content, and checks if the extracted text length is within the token limit configured for OpenAI processing.

- **Nodes Involved:**  
  - Is attachment a PDF? (If)  
  - Read PDF email attachments (Read PDF)  
  - Is text within token limit? (If)  
  - Not a PDF (NoOp)  

- **Node Details:**

  - **Is attachment a PDF?**  
    - Type: If node  
    - Role: Checks if the attachment file extension is "pdf" (case-sensitive).  
    - Configuration: Compares `$binary.data.fileExtension` to `"pdf"`.  
    - Inputs: Individual attachments from Iterate node  
    - Outputs: True branch to Read PDF node; False branch to NoOp node  
    - Edge Cases: Missing file extension, uppercase extensions, non-PDF files.

  - **Read PDF email attachments**  
    - Type: Read PDF node  
    - Role: Extracts text content from PDF binary data.  
    - Configuration: Default settings; continues on error (does not fail workflow).  
    - Inputs: PDF attachments  
    - Outputs: JSON with extracted text  
    - Edge Cases: Corrupted PDFs, encrypted PDFs, extraction failures.

  - **Is text within token limit?**  
    - Type: If node  
    - Role: Checks if the extracted text length divided by 4 (approximate token count) is less than or equal to `maxTokenSize - replyTokenSize` from Configure node.  
    - Configuration: Expression compares `text.length()/4 <= maxTokenSize - replyTokenSize`.  
    - Inputs: PDF text from Read PDF node  
    - Outputs: True branch to OpenAI classification; False branch discards or skips.  
    - Edge Cases: Very large PDFs exceeding token limits, missing text field.

  - **Not a PDF**  
    - Type: NoOp node  
    - Role: Placeholder for non-PDF attachments; effectively ignores them.  
    - Inputs: Non-PDF attachments  
    - Outputs: None (ends branch)  
    - Edge Cases: None.

---

#### 2.4 AI Classification

- **Overview:**  
  Uses OpenAI to classify each PDF based on its filename and extracted text, determining if it matches the user-defined category (e.g., receipt or invoice).

- **Nodes Involved:**  
  - OpenAI (Langchain OpenAI node)  
  - Merge (Merge node)  
  - Is matched (If node)  

- **Node Details:**

  - **OpenAI**  
    - Type: Langchain OpenAI node  
    - Role: Sends prompt to OpenAI GPT-4.1-MINI model to classify PDF as matching or not matching the category.  
    - Configuration:  
      - Model: GPT-4.1-MINI  
      - Prompt: Asks if the PDF file looks like the category defined in Configure node's "Match on" field.  
      - Input variables: PDF filename (`$binary.data.fileName`), extracted text (`$json.text`), classification keyword.  
      - Credentials: OpenAI API key  
    - Inputs: PDF text and binary data from previous nodes  
    - Outputs: JSON with OpenAI response content ("true" or "false")  
    - Edge Cases: API rate limits, prompt failures, network errors, unexpected responses.

  - **Merge**  
    - Type: Merge node (combine mode)  
    - Role: Combines OpenAI classification result with original PDF binary data for downstream processing.  
    - Configuration: Merges by position, preferring input 1 on clash.  
    - Inputs: OpenAI classification (input 1), original PDF binary data (input 2)  
    - Outputs: Combined JSON and binary data  
    - Edge Cases: Mismatched input lengths, data loss.

  - **Is matched**  
    - Type: If node  
    - Role: Checks if OpenAI response content equals "true" (case-sensitive).  
    - Configuration: Compares `$json.message.content` to `"true"`.  
    - Inputs: Merged data from Merge node  
    - Outputs: True branch to upload and email nodes; False branch ends processing for that file.  
    - Edge Cases: Unexpected OpenAI responses, missing content field.

---

#### 2.5 Upload and Optional Email Dispatch

- **Overview:**  
  Uploads matched PDFs to the Google Drive folder created earlier and optionally sends an email with all matched PDFs attached to a configured recipient.

- **Nodes Involved:**  
  - Upload file to folder (Google Drive)  
  - Send email with invoices? (If)  
  - Aggregate attachments (Code)  
  - Send to my accountant (Gmail)  

- **Node Details:**

  - **Upload file to folder**  
    - Type: Google Drive node (upload file)  
    - Role: Uploads each matched PDF file to the folder created in step 1.  
    - Configuration:  
      - File name: Original PDF filename from binary data  
      - Parent folder: ID from Create folder node  
      - Uploads binary data  
      - Uses Google Drive OAuth2 credentials  
    - Inputs: Matched PDF binary and JSON data  
    - Outputs: Metadata of uploaded file  
    - Edge Cases: API quota limits, permission errors, file name conflicts.

  - **Send email with invoices?**  
    - Type: If node  
    - Role: Checks if the `sendEmail` flag from Configure node is true.  
    - Configuration: Boolean check on `sendEmail` parameter.  
    - Inputs: After upload node  
    - Outputs: True branch to aggregate attachments and send email; False branch ends workflow.  
    - Edge Cases: Missing or invalid flag.

  - **Aggregate attachments**  
    - Type: Code node  
    - Role: Aggregates all matched PDF binary files into a single item with multiple attachments for email sending.  
    - Configuration: Custom JS code merges all binary files keyed by filename into one item.  
    - Inputs: Multiple matched PDF items  
    - Outputs: Single item with all binary attachments  
    - Edge Cases: Large number of attachments causing memory issues.

  - **Send to my accountant**  
    - Type: Gmail node (send email)  
    - Role: Sends an email with all aggregated matched PDFs attached to the recipient email specified in Configure node.  
    - Configuration:  
      - Recipient: Configured email (default example: `test@example.com`)  
      - Subject: Includes start and end dates from webhook body, formatted in Polish ("Dokumenty kosztowe za okres od ... do ...")  
      - Message body: Simple text message  
      - Attachments: All aggregated PDFs from previous node  
      - Uses Gmail OAuth2 credentials  
    - Inputs: Aggregated attachments  
    - Outputs: Email sent confirmation  
    - Edge Cases: Gmail API limits, attachment size limits, invalid recipient email.

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                                      | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                                                   |
|-----------------------------|--------------------------------|-----------------------------------------------------|-----------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Webhook                     | HTTP Webhook Trigger            | Entry point; receives date range and sendEmail flag | None                        | Create folder                    |                                                                                                                              |
| Create folder               | Google Drive (folder creation) | Creates dated folder in Google Drive                 | Webhook                     | Configure, Respond to Webhook    |                                                                                                                              |
| Configure                  | Set node                       | Defines parameters and settings                       | Create folder               | Get emails with attachments      | See Sticky Note1 for detailed parameter explanations                                                                         |
| Respond to Webhook          | Respond to Webhook             | Sends HTTP 202 response with folder URL              | Create folder               | None                           |                                                                                                                              |
| Get emails with attachments | Gmail (getAll)                 | Retrieves emails with attachments in date range     | Configure                   | Optional filter for emails       |                                                                                                                              |
| Optional filter for emails  | Filter                        | Optionally excludes emails sent to specific addresses| Get emails with attachments | Iterate over email attachments   |                                                                                                                              |
| Iterate over email attachments | Code                        | Flattens attachments into individual items           | Optional filter for emails  | Is attachment a PDF?             |                                                                                                                              |
| Is attachment a PDF?        | If                            | Filters only PDF attachments                          | Iterate over email attachments | Read PDF email attachments, Not a PDF |                                                                                                                              |
| Not a PDF                   | NoOp                          | Ignores non-PDF attachments                           | Is attachment a PDF? (false) | None                           |                                                                                                                              |
| Read PDF email attachments  | Read PDF                      | Extracts text from PDF attachments                    | Is attachment a PDF? (true) | Is text within token limit?      |                                                                                                                              |
| Is text within token limit? | If                            | Checks if PDF text length is within OpenAI token limit| Read PDF email attachments  | OpenAI, Merge                   |                                                                                                                              |
| OpenAI                     | Langchain OpenAI               | Classifies PDF as matching or not                     | Is text within token limit? | Merge                          |                                                                                                                              |
| Merge                      | Merge                         | Combines OpenAI result with original PDF binary      | OpenAI, Is text within token limit? | Is matched                    |                                                                                                                              |
| Is matched                 | If                            | Checks if OpenAI response is "true"                   | Merge                       | Upload file to folder, Send email with invoices? |                                                                                                                              |
| Upload file to folder       | Google Drive (upload file)     | Uploads matched PDFs to Google Drive folder           | Is matched                  | None                           |                                                                                                                              |
| Send email with invoices?   | If                            | Checks if email sending is enabled                     | Is matched                  | Aggregate attachments           |                                                                                                                              |
| Aggregate attachments       | Code                          | Aggregates all matched PDFs into one email attachment | Send email with invoices?   | Send to my accountant           |                                                                                                                              |
| Send to my accountant       | Gmail (send email)             | Sends email with all matched PDFs attached            | Aggregate attachments       | None                           |                                                                                                                              |
| Sticky Note                 | Sticky Note                   | Workflow overview and disclaimer                       | None                        | None                           | See section 5 for full content                                                                                               |
| Sticky Note1                | Sticky Note                   | Parameter explanations                                 | None                        | None                           | See section 5 for full content                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - Path: Unique identifier (e.g., `cded3af3-31df-47c2-a826-ff84eb4a41df`)  
   - HTTP Method: POST  
   - Authentication: Header Auth (configure credentials)  
   - Purpose: Receive JSON body with `startDate`, `endDate`, and `sendEmail` boolean.

2. **Create Google Drive Folder Node**  
   - Type: Google Drive (folder creation)  
   - Name: `invoices_{{ $json.body.startDate.split('T')[0] }}`  
   - Parent Folder: Root (`root`)  
   - Credentials: Google Drive OAuth2  
   - Connect Webhook output to this node.

3. **Create Configure Node (Set)**  
   - Type: Set node  
   - Parameters:  
     - Number:  
       - `maxTokenSize` = 8000  
       - `replyTokenSize` = 50  
     - String:  
       - `Match on` = `"receipt or invoice that can be considered a software engineering business cost"`  
       - `Google Drive folder to upload matched PDFs` = `"https://drive.google.com/drive/folders/" + $json.id` (from Create folder)  
       - `sendInvoicesTo` = (recipient email, e.g., `"accounting@example.com"`)  
     - Boolean:  
       - `sendEmail` = `={{ $('Webhook').item.json.body.sendEmail === "true" }}`  
   - Connect Create folder output to Configure node.

4. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Response Code: 202  
   - Response Body (JSON):  
     ```json
     {
       "status": "Accepted",
       "driveFolderUrl": "https://drive.google.com/drive/folders/{{ $json.id }}"
     }
     ```  
   - Connect Create folder output to Respond to Webhook.

5. **Create Gmail Node to Get Emails with Attachments**  
   - Type: Gmail (getAll)  
   - Query: `has:attachment`  
   - Filters:  
     - `receivedAfter` = `={{ $('Webhook').item.json.body.startDate }}`  
     - `receivedBefore` = `={{ $('Webhook').item.json.body.endDate }}`  
   - Download Attachments: Enabled  
   - Credentials: Gmail OAuth2  
   - Connect Configure output to this node.

6. **Create Optional Filter Node**  
   - Type: Filter  
   - Condition: Exclude emails sent to specific address (default empty, can be customized)  
   - Connect Gmail node output to this node.

7. **Create Code Node to Iterate Over Email Attachments**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     let results = [];
     for (const item of $input.all()) {
       for (const key of Object.keys(item.binary)) {
         results.push({
           json: {},
           binary: {
             data: item.binary[key],
           }
         });
       }
     }
     return results;
     ```  
   - Connect Filter node output to this node.

8. **Create If Node to Check for PDF Attachments**  
   - Type: If  
   - Condition: `$binary.data.fileExtension == "pdf"`  
   - Connect Code node output to this node.

9. **Create Read PDF Node**  
   - Type: Read PDF  
   - On Error: Continue (do not fail workflow)  
   - Connect If node True output to this node.

10. **Create If Node to Check Text Token Limit**  
    - Type: If  
    - Condition:  
      ``` 
      $json.text.length()/4 <= maxTokenSize - replyTokenSize 
      ```  
      where `maxTokenSize` and `replyTokenSize` come from Configure node.  
    - Connect Read PDF output to this node.

11. **Create OpenAI Node**  
    - Type: Langchain OpenAI  
    - Model: GPT-4.1-MINI  
    - Prompt:  
      ```
      Does this PDF file look like a {{ $(“Configure”).first().json[“Match on”] }}? Return "true" if it is a {{ $(“Configure”).first().json[“Match on”] }} and "false" if not. Only reply with lowercase letters "true" or "false".

      This is the PDF filename:
      ```
      {{ $binary.data.fileName }}
      ```

      This is the PDF text content:
      ```
      {{ $json.text }}
      ```
      ```  
    - Credentials: OpenAI API key  
    - Connect If node True output (token limit check) to OpenAI node.

12. **Create Merge Node**  
    - Type: Merge (combine mode)  
    - Combine OpenAI output (input 1) with original PDF binary data (input 2)  
    - Connect OpenAI output to input 1, and the token limit If node False output (or original PDF binary) to input 2.

13. **Create If Node to Check OpenAI Classification**  
    - Type: If  
    - Condition: `$json.message.content == "true"`  
    - Connect Merge output to this node.

14. **Create Google Drive Upload Node**  
    - Type: Google Drive (upload file)  
    - File Name: `={{ $binary.data.fileName }}`  
    - Parent Folder: ID from Create folder node  
    - Upload binary data  
    - Credentials: Google Drive OAuth2  
    - Connect If node True output to this node.

15. **Create If Node to Check if Email Sending is Enabled**  
    - Type: If  
    - Condition: `sendEmail == true` (from Configure node)  
    - Connect If node True output (classification) to this node.

16. **Create Code Node to Aggregate Attachments**  
    - Type: Code (JavaScript)  
    - Code:  
      ```javascript
      const merged = { json: {}, binary: {} };
      for (const item of $input.all()) {
        const data = { [item.binary.data.fileName]: item.binary.data };
        Object.assign(merged.binary, data);
      }
      return [merged];
      ```  
    - Connect If node True output (send email) to this node.

17. **Create Gmail Node to Send Email**  
    - Type: Gmail (send email)  
    - Recipient: Configured email (e.g., from Configure node `sendInvoicesTo`)  
    - Subject:  
      ```
      Dokumenty kosztowe za okres od {{ $node['Webhook'].json.body.startDate.split('T')[0] }} do {{ $node['Webhook'].json.body.endDate.split('T')[0] }}
      ```  
    - Message: "Hello, here are my invoices and receipts."  
    - Attachments: All aggregated PDFs (binary data)  
    - Credentials: Gmail OAuth2  
    - Connect Code node output (aggregate attachments) to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| **Workflow Purpose:** Automatically aggregates invoices and receipts from Gmail, classifies them using OpenAI, uploads matched PDFs to Google Drive, and optionally emails them to an accountant. Designed for small business owners and freelancers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Workflow description                                                                                         |
| **Disclaimer:** AI classification is not perfect. Always verify that the correct documents were identified and uploaded.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note content                                                                                          |
| **Parameters:**  
- `maxTokenSize` (default 8000): Limits PDF text length sent to OpenAI to avoid errors and high costs.  
- `replyTokenSize` (default 50): Reserves tokens for OpenAI's reply.  
- `Match on`: Keyword or phrase for classification (default "receipt or invoice that can be considered a software engineering business cost").  
- `sendInvoicesTo`: Recipient email for final email step.  
- `sendEmail`: Boolean flag to enable/disable sending email with attachments.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note1 content                                                                                         |
| **Setup Requirements:**  
- Gmail and Google Drive accounts with OAuth2 credentials configured in n8n.  
- OpenAI API key configured in n8n.  
- Google Cloud OAuth 2.0 Client ID or service account with Gmail and Drive API enabled.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Workflow preconditions                                                                                       |
| **Webhook Input Schema:**  
```json
{
  "name": "getInvoicesAndReceiptsFromEmails",
  "description": "Finds and uploads to Google Drive all receipts and invoices from emails within a specified date range.",
  "parameters": {
    "type": "object",
    "properties": {
      "startDate": {
        "type": "string",
        "format": "date-time",
        "description": "The start date of the range to search for emails. Must be in ISO 8601 format."
      },
      "endDate": {
        "type": "string",
        "format": "date-time",
        "description": "The end date of the range to search for emails. Must be in ISO 8601 format."
      },
      "sendEmail": {
        "type": "boolean",
        "description": "Indicates whether to send an email with all receipts and invoices after processing. Must be true or false."
      }
    },
    "required": [
      "startDate",
      "endDate"
    ]
  }
}
```                                                                                                                                                                                                                                                                                                                                                                  | Webhook input schema                                                                                         |
| **Example Webhook Body:**  
```json
{
  "startDate": "2025-03-01T00:00:00Z",
  "endDate": "2025-04-01T00:00:00Z",
  "sendEmail": true
}
```                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Example webhook payload                                                                                      |
| **Integration with AI Chat:** Can be triggered by AI chat tools supporting tool use, e.g., BrowseWiz. Setup instructions available at [BrowseWiz blog](https://browsewiz.com/blog/invoice-and-receipt-aggregation-n8n-workflow-and-how-to-add-it-to-browsewiz).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | External integration link                                                                                     |
| **Author:** Workflow written by [Tom](https://browsewiz.com)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Author credit                                                                                                |

---

This document provides a detailed, stepwise understanding and reproduction guide for the "Extract & Classify Invoices & Receipts with Gmail, OpenAI and Google Drive" workflow, enabling advanced users and AI agents to maintain, modify, or extend the workflow confidently.