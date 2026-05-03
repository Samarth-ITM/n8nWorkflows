Process emailed PDF invoices into QuickBooks bills with OpenAI

https://n8nworkflows.xyz/workflows/process-emailed-pdf-invoices-into-quickbooks-bills-with-openai-14085


# Process emailed PDF invoices into QuickBooks bills with OpenAI

# 1. Workflow Overview

This workflow automatically turns emailed PDF invoices into QuickBooks Online bills, using OpenAI to classify and extract invoice data from the PDF content.

Primary use cases:
- Small and medium businesses receiving vendor invoices by email
- Bookkeepers processing recurring PDF invoices
- Accounting teams wanting semi-automated AP intake into QuickBooks Online

The workflow is organized into five logical blocks.

## 1.1 Email Trigger & Configuration
The workflow starts when Gmail detects a new email with a PDF attachment. A configuration node enriches the item with QuickBooks company settings, notification settings, and fallback accounting values.

## 1.2 AI Invoice Classification & Data Extraction
The PDF attachment is converted to text, then passed to an OpenAI-powered Information Extractor node. The AI determines whether the PDF is actually an invoice and extracts structured billing fields.

## 1.3 Validation, Vendor Lookup, and Bill Creation
The extracted invoice data is validated in code. If valid, the workflow searches QuickBooks for a matching vendor and creates a bill using the native QuickBooks node.

## 1.4 PDF Attachment Upload Pipeline
Because the bill is created first and the original PDF must then be attached to it, a separate binary-handling pipeline reconstructs multipart upload data. It uploads both metadata and the original PDF file to the QuickBooks upload endpoint.

## 1.5 Confirmation and Error Handling
Depending on the result, the workflow either:
- silently skips non-invoices,
- emails the AP team for extraction/vendor issues,
- or optionally sends a confirmation email back to the original sender after successful processing.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Email Trigger & Configuration

### Overview
This block receives incoming Gmail messages with PDF attachments and adds central configuration values used throughout the workflow. It also preserves binary attachment access for downstream nodes.

### Nodes Involved
- Gmail Trigger
- Config

### Node Details

#### Gmail Trigger
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Poll-based Gmail trigger that starts the workflow when matching emails arrive.
- **Configuration choices:**
  - Polls every 15 minutes
  - Gmail search query is `filename:pdf`
  - `Simplify` is disabled, so the node returns fuller Gmail payload data
  - Attachment download is enabled
- **Key expressions or variables used:**
  - No complex expression in configuration
  - Downstream nodes use fields such as:
    - `{{$('Gmail Trigger').item.json.from}}`
    - `{{$('Gmail Trigger').item.json.subject}}`
    - `{{$('Gmail Trigger').item.json.messageId}}`
- **Input and output connections:**
  - Entry point node
  - Outputs to `Config`
- **Version-specific requirements:**
  - Uses type version `1.3`
  - Behavior depends on Gmail OAuth2 credential support in the n8n instance
- **Edge cases or potential failure types:**
  - Gmail OAuth2 authorization failure
  - Polling delays due to 15-minute interval
  - Emails with no actual PDF binary despite matching filename query
  - Multiple attachments are not fully handled; the workflow is hardcoded around `attachment_0`
  - Messages matching the query but containing non-invoice PDFs
- **Sub-workflow reference:** None

#### Config
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds reusable workflow settings while preserving original Gmail fields and binaries.
- **Configuration choices:**
  - `includeOtherFields: true`, so original Gmail data remains available
  - Defines:
    - `realmId`: QuickBooks company ID
    - `apTeamEmail`: destination for manual-processing alerts
    - `sendConfirmation`: boolean toggle for sender confirmation emails
    - `defaultExpenseAccountId`: default QuickBooks expense account for bill line items
- **Key expressions or variables used:**
  - Static values configured directly in the node
  - Later referenced via:
    - `{{$('Config').item.json.realmId}}`
    - `{{$('Config').item.json.apTeamEmail}}`
    - `{{$('Config').item.json.sendConfirmation}}`
    - `{{$('Config').item.json.defaultExpenseAccountId}}`
  - Binary reference:
    - `$('Config').item.binary.attachment_0`
- **Input and output connections:**
  - Input from `Gmail Trigger`
  - Output to `Extract Text from PDF`
- **Version-specific requirements:**
  - Uses Set node version `3.4`
- **Edge cases or potential failure types:**
  - If `realmId` is not set, QuickBooks upload URL will fail
  - If `defaultExpenseAccountId` is invalid, bill creation will fail
  - If `apTeamEmail` is invalid, error notifications cannot be sent
  - If no `attachment_0` exists, later PDF extraction/upload steps fail
- **Sub-workflow reference:** None

---

## 2.2 Block: AI Invoice Classification & Data Extraction

### Overview
This block converts the PDF to plain text and uses OpenAI to extract invoice fields in a structured schema. It also filters out non-invoice documents before any accounting action is taken.

### Nodes Involved
- Extract Text from PDF
- OpenAI Chat Model
- AI Extract Invoice Data
- Is Invoice?
- Validate Extracted Data
- Is Valid?

### Node Details

#### Extract Text from PDF
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Reads the PDF binary and extracts textual content.
- **Configuration choices:**
  - Operation: `pdf`
  - Binary property: `attachment_0`
  - Joins all pages into one text output
- **Key expressions or variables used:**
  - Binary field name is fixed: `attachment_0`
- **Input and output connections:**
  - Input from `Config`
  - Output to `AI Extract Invoice Data`
- **Version-specific requirements:**
  - Uses type version `1`
- **Edge cases or potential failure types:**
  - Missing binary field `attachment_0`
  - Encrypted/scanned/image-only PDFs may extract poor or no text
  - Very large PDFs may cause memory or processing overhead
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM backend used by the Information Extractor node.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0` for deterministic extraction behavior
- **Key expressions or variables used:**
  - No runtime expression; attached as AI model input to the extractor node
- **Input and output connections:**
  - No main input/output path
  - Connected through `ai_languageModel` to `AI Extract Invoice Data`
- **Version-specific requirements:**
  - Uses LangChain node package
  - Requires compatible n8n version with AI nodes enabled
- **Edge cases or potential failure types:**
  - OpenAI credential/authentication failure
  - API quota/rate limiting
  - Model availability changes
  - Cost exposure if many PDFs are processed
- **Sub-workflow reference:** None

#### AI Extract Invoice Data
- **Type and technical role:** `@n8n/n8n-nodes-langchain.informationExtractor`  
  Uses an LLM and a provided schema example to produce structured invoice data.
- **Configuration choices:**
  - Input text: `{{$json.text}}`
  - Schema mode: JSON schema inferred from provided example
  - Expected fields include:
    - `is_invoice`
    - `vendor_name`
    - `invoice_number`
    - `amount`
    - `currency`
    - `due_date`
    - `txn_date`
    - `line_items[]`
- **Key expressions or variables used:**
  - `={{ $json.text }}`
  - Output is later consumed as `{{$json.output...}}`
- **Input and output connections:**
  - Main input from `Extract Text from PDF`
  - AI model input from `OpenAI Chat Model`
  - Main output to `Is Invoice?`
- **Version-specific requirements:**
  - Uses type version `1.2`
  - Requires LangChain integration support
- **Edge cases or potential failure types:**
  - Hallucinated or malformed extraction
  - Missing expected `output` structure
  - Inaccurate date/currency extraction
  - Poor PDF text quality can degrade results
- **Sub-workflow reference:** None

#### Is Invoice?
- **Type and technical role:** `n8n-nodes-base.if`  
  Filters processing to only items where the AI marked `is_invoice` as true.
- **Configuration choices:**
  - Boolean condition:
    - `{{$json.output.is_invoice}} is true`
- **Key expressions or variables used:**
  - `={{ $json.output.is_invoice }}`
- **Input and output connections:**
  - Input from `AI Extract Invoice Data`
  - True output to `Validate Extracted Data`
  - False output is unused, effectively silently skipping non-invoices
- **Version-specific requirements:**
  - Uses IF node version `2.2`
- **Edge cases or potential failure types:**
  - If `output.is_invoice` is missing or not boolean, item may be skipped unexpectedly
  - Non-invoice documents are intentionally dropped with no notification
- **Sub-workflow reference:** None

#### Validate Extracted Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Validates required invoice fields and normalizes dates/currency.
- **Configuration choices:**
  - Runs once per item
  - Checks:
    - `vendor_name` exists
    - `amount` is present and > 0
    - `line_items` exists and is non-empty
  - Normalizes `due_date` and `txn_date` to `YYYY-MM-DD`
  - Defaults currency to `USD`
- **Key expressions or variables used:**
  - Reads `const data = $json.output`
  - Returns:
    - `valid`
    - `errors`
    - `data`
- **Input and output connections:**
  - Input from `Is Invoice?`
  - Output to `Is Valid?`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Unexpected extractor shape could cause runtime issues
  - Invalid date strings become `null`
  - Amount `0` or missing line items are treated as invalid
  - Currency defaults to USD even if invoice is in another currency but extraction failed
- **Sub-workflow reference:** None

#### Is Valid?
- **Type and technical role:** `n8n-nodes-base.if`  
  Splits valid invoice data from invalid extraction results.
- **Configuration choices:**
  - Boolean condition on `{{$json.valid}}`
- **Key expressions or variables used:**
  - `={{ $json.valid }}`
- **Input and output connections:**
  - Input from `Validate Extracted Data`
  - True output to `Search vendor`
  - False output to `Error: Extraction Failed`
- **Version-specific requirements:**
  - Uses IF node version `2.2`
- **Edge cases or potential failure types:**
  - If `valid` is absent or non-boolean, routing may be incorrect
- **Sub-workflow reference:** None

---

## 2.3 Block: QuickBooks Vendor Search & Bill Creation

### Overview
This block looks up the vendor in QuickBooks, prepares a single-line bill payload, and creates the bill. If no vendor is found, it notifies the AP team instead of creating invalid accounting records.

### Nodes Involved
- Search vendor
- Vendor Found?
- Prepare Bill Data
- Create Bill

### Node Details

#### Search vendor
- **Type and technical role:** `n8n-nodes-base.quickbooks`  
  Searches QuickBooks Online vendor records using the extracted vendor name.
- **Configuration choices:**
  - Resource: `vendor`
  - Operation: `getAll`
  - Limit: `1`
  - Query filter:
    - `WHERE DisplayName LIKE '%{{ $json.data.vendor_name }}%'`
- **Key expressions or variables used:**
  - `={{ $json.data.vendor_name }}`
- **Input and output connections:**
  - Input from `Is Valid?`
  - Output to `Vendor Found?`
- **Version-specific requirements:**
  - Uses QuickBooks node version `1`
  - Requires valid QuickBooks OAuth2 credentials
- **Edge cases or potential failure types:**
  - QuickBooks auth/token refresh failure
  - Query matching may be too loose or too strict
  - Vendor names containing quotes or special characters may break query syntax
  - Only first result is used; ambiguous matches are not resolved
  - No normalization for punctuation/casing differences
- **Sub-workflow reference:** None

#### Vendor Found?
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether the vendor lookup returned a QuickBooks vendor ID.
- **Configuration choices:**
  - Condition checks if `{{$json.Id}}` exists
- **Key expressions or variables used:**
  - `={{ $json.Id }}`
- **Input and output connections:**
  - Input from `Search vendor`
  - True output to `Prepare Bill Data`
  - False output to `Error: Vendor Not Found`
- **Version-specific requirements:**
  - Uses IF node version `2.2`
- **Edge cases or potential failure types:**
  - If QuickBooks returns an empty array or unexpected structure, no vendor path is taken
- **Sub-workflow reference:** None

#### Prepare Bill Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Assembles the bill payload expected by the QuickBooks Bill creation node.
- **Configuration choices:**
  - Runs once per item
  - Reads validated extraction data from `Validate Extracted Data`
  - Reads vendor ID from `Search vendor`
  - Reads default expense account from `Config`
  - Converts multiple extracted line items into one combined description string
  - Truncates description to 4000 characters
- **Key expressions or variables used:**
  - `$('Validate Extracted Data').item.json.data`
  - `$('Search vendor').item.json.Id`
  - `$('Config').item.json`
- **Input and output connections:**
  - Input from `Vendor Found?`
  - Output to `Create Bill`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Named-node references fail if upstream execution context changes unexpectedly
  - Long line item descriptions are truncated
  - Quantity is only included textually, not as actual accounting line structure
  - If `defaultExpenseAccountId` is missing, bill creation will fail
- **Sub-workflow reference:** None

#### Create Bill
- **Type and technical role:** `n8n-nodes-base.quickbooks`  
  Creates a QuickBooks Online bill using a single account-based expense line.
- **Configuration choices:**
  - Resource: `bill`
  - Operation: `create`
  - `VendorRef` from prepared vendor ID
  - One `Line` entry:
    - `Amount`: extracted total
    - `accountId`: configured default expense account
    - `DetailType`: `AccountBasedExpenseLineDetail`
    - `Description`: combined invoice summary
  - Additional fields:
    - `DueDate`
    - `TxnDate`
    - `TotalAmt`
- **Key expressions or variables used:**
  - `={{ $json.vendorId }}`
  - `={{ $json.amount }}`
  - `={{ $json.accountId }}`
  - `={{ $json.description }}`
  - `={{ $json.dueDate }}`
  - `={{ $json.txnDate }}`
  - `={{ $json.totalAmt }}`
- **Input and output connections:**
  - Input from `Prepare Bill Data`
  - Output to both:
    - `Build Attachment Metadata`
    - `Fetch PDF Binary`
- **Version-specific requirements:**
  - Uses QuickBooks node version `1`
- **Edge cases or potential failure types:**
  - Invalid vendor/account references
  - QuickBooks OAuth2 auth issues
  - Currency mismatches are not explicitly handled at bill-creation level
  - The node only creates one line item in this implementation
  - Empty dates may be accepted or rejected depending on API behavior
- **Sub-workflow reference:** None

---

## 2.4 Block: PDF Attachment Pipeline

### Overview
This block uploads the original PDF to the newly created QuickBooks bill. It reconstructs multipart form data by building attachment metadata JSON, re-fetching the original PDF binary, merging both binaries, and converting stream-backed binaries into inline base64 before upload.

### Nodes Involved
- Build Attachment Metadata
- Metadata to Binary
- Fetch PDF Binary
- Merge PDF + Metadata
- Force Inline Binary
- Upload PDF to Bill

### Node Details

#### Build Attachment Metadata
- **Type and technical role:** `n8n-nodes-base.code`  
  Creates the JSON metadata required by QuickBooks for attaching a file to a bill.
- **Configuration choices:**
  - Reads original binary metadata from `Config.binary.attachment_0`
  - Builds JSON with:
    - `AttachableRef` referencing the created bill ID
    - `FileName`
    - `ContentType`
- **Key expressions or variables used:**
  - `$('Config').item.binary.attachment_0`
  - `$('Create Bill').item.json.Id`
- **Input and output connections:**
  - Input from `Create Bill`
  - Output to `Metadata to Binary`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Missing `attachment_0` binary in `Config`
  - Missing `Create Bill` ID
  - File metadata may be incomplete if Gmail did not provide it
- **Sub-workflow reference:** None

#### Metadata to Binary
- **Type and technical role:** `n8n-nodes-base.code`  
  Serializes the metadata JSON into a binary file (`data.json`) for multipart upload.
- **Configuration choices:**
  - Converts current JSON into UTF-8 buffer
  - Stores it under binary property `myBinaryFile`
- **Key expressions or variables used:**
  - Uses current `$json`
- **Input and output connections:**
  - Input from `Build Attachment Metadata`
  - Output to `Merge PDF + Metadata`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Rare buffer serialization issues
  - Downstream node expects property name `myBinaryFile`; renaming would break upload
- **Sub-workflow reference:** None

#### Fetch PDF Binary
- **Type and technical role:** `n8n-nodes-base.code`  
  Rehydrates the original PDF binary from the earlier `Config` node.
- **Configuration choices:**
  - Returns all binary data from `$('Config').item.binary`
- **Key expressions or variables used:**
  - `$('Config').item.binary`
- **Input and output connections:**
  - Input from `Create Bill`
  - Output to `Merge PDF + Metadata` at input index 1
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - If the trigger email had no downloaded binary, upload cannot proceed
  - If multiple attachments existed, this copies all binary properties, though downstream expects `attachment_0`
- **Sub-workflow reference:** None

#### Merge PDF + Metadata
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the metadata item and the PDF binary item into a single item by position.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `position`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Input 0 from `Metadata to Binary`
  - Input 1 from `Fetch PDF Binary`
  - Output to `Force Inline Binary`
- **Version-specific requirements:**
  - Uses Merge node version `3.2`
- **Edge cases or potential failure types:**
  - Item count mismatch between branches can produce no merged result
- **Sub-workflow reference:** None

#### Force Inline Binary
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts stream/database-backed binary files into inline base64 binary objects to satisfy the QuickBooks upload API requirements.
- **Configuration choices:**
  - Iterates through all input items
  - For `attachment_0`:
    - reads binary buffer with `this.helpers.getBinaryDataBuffer`
    - rewrites binary object with inline `data`
  - For `myBinaryFile`:
    - performs the same conversion
- **Key expressions or variables used:**
  - `await this.helpers.getBinaryDataBuffer(i, 'attachment_0')`
  - `await this.helpers.getBinaryDataBuffer(i, 'myBinaryFile')`
- **Input and output connections:**
  - Input from `Merge PDF + Metadata`
  - Output to `Upload PDF to Bill`
- **Version-specific requirements:**
  - Uses Code node version `2`
  - This node is especially relevant in n8n v2-style binary storage behavior
- **Edge cases or potential failure types:**
  - Missing binary field names
  - Binary buffer retrieval failures
  - Memory pressure for very large PDFs due to inline base64 conversion
- **Sub-workflow reference:** None

#### Upload PDF to Bill
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the QuickBooks upload endpoint directly using multipart form data.
- **Configuration choices:**
  - Method: `POST`
  - URL:
    - `https://quickbooks.api.intuit.com/v3/company/{{ $('Config').item.json.realmId }}/upload`
  - Authentication: predefined QuickBooks OAuth2 credential
  - Sends multipart form-data with:
    - `file_metadata_01` → binary field `myBinaryFile`
    - `file_content_01` → binary field `attachment_0`
  - Full response enabled
- **Key expressions or variables used:**
  - `=https://quickbooks.api.intuit.com/v3/company/{{ $('Config').item.json.realmId }}/upload`
  - `=myBinaryFile`
- **Input and output connections:**
  - Input from `Force Inline Binary`
  - Output to `Send Confirmation?`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Invalid `realmId`
  - QuickBooks OAuth2 token failure
  - Multipart format rejection if binary fields are malformed
  - Upload size/API limit issues
  - If the bill was created but upload fails, the workflow leaves a bill without attachment
- **Sub-workflow reference:** None

---

## 2.5 Block: Confirmation and Error Handling

### Overview
This block handles the final communication outcomes. It optionally confirms successful processing to the email sender and notifies the AP team when AI extraction fails or no QuickBooks vendor match is found.

### Nodes Involved
- Send Confirmation?
- Send Confirmation Email
- Error: Extraction Failed
- Error: Vendor Not Found

### Node Details

#### Send Confirmation?
- **Type and technical role:** `n8n-nodes-base.if`  
  Gates sender confirmation based on a configuration toggle.
- **Configuration choices:**
  - Boolean condition on `$('Config').item.json.sendConfirmation`
- **Key expressions or variables used:**
  - `={{ $('Config').item.json.sendConfirmation }}`
- **Input and output connections:**
  - Input from `Upload PDF to Bill`
  - True output to `Send Confirmation Email`
  - False output ends workflow silently
- **Version-specific requirements:**
  - Uses IF node version `2.2`
- **Edge cases or potential failure types:**
  - If config value is missing or non-boolean, behavior may be unexpected
- **Sub-workflow reference:** None

#### Send Confirmation Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a plain-text success notification back to the original sender.
- **Configuration choices:**
  - Recipient: original sender from Gmail trigger
  - Subject includes extracted invoice number and created QuickBooks bill ID
  - Plain text body confirms processing and PDF attachment
- **Key expressions or variables used:**
  - `{{$('Gmail Trigger').item.json.from}}`
  - `{{$('Validate Extracted Data').item.json.data.invoice_number || 'N/A'}}`
  - `{{$('Create Bill').item.json.Id}}`
- **Input and output connections:**
  - Input from `Send Confirmation?`
  - No downstream output
- **Version-specific requirements:**
  - Uses Gmail node version `2.1`
- **Edge cases or potential failure types:**
  - Gmail send authorization failure
  - Sender address may be malformed or not directly reply-safe
  - Confirmation can expose internal automation details to external senders
- **Sub-workflow reference:** None

#### Error: Extraction Failed
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Emails the AP team when AI extraction produced unusable invoice data.
- **Configuration choices:**
  - Recipient: `Config.apTeamEmail`
  - Includes source sender, subject, extraction errors, and original Gmail message ID
- **Key expressions or variables used:**
  - `{{$('Config').item.json.apTeamEmail}}`
  - `{{$('Gmail Trigger').item.json.from}}`
  - `{{$('Gmail Trigger').item.json.subject}}`
  - `{{$json.errors.join(', ')}}`
  - `{{$('Gmail Trigger').item.json.messageId}}`
- **Input and output connections:**
  - Input from false output of `Is Valid?`
  - No downstream output
- **Version-specific requirements:**
  - Uses Gmail node version `2.1`
- **Edge cases or potential failure types:**
  - Notification email failure if Gmail credentials fail
  - If `errors` is not an array, expression may fail
- **Sub-workflow reference:** None

#### Error: Vendor Not Found
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Alerts the AP team that the extracted vendor does not exist in QuickBooks.
- **Configuration choices:**
  - Recipient: `Config.apTeamEmail`
  - Includes vendor name, invoice number, amount, original sender, and manual next steps
- **Key expressions or variables used:**
  - `{{$('Config').item.json.apTeamEmail}}`
  - `{{$('Validate Extracted Data').item.json.data.vendor_name}}`
  - `{{$('Validate Extracted Data').item.json.data.invoice_number || 'N/A'}}`
  - `{{$('Validate Extracted Data').item.json.data.currency}}`
  - `{{$('Validate Extracted Data').item.json.data.amount}}`
  - `{{$('Gmail Trigger').item.json.from}}`
  - `{{$('Gmail Trigger').item.json.messageId}}`
- **Input and output connections:**
  - Input from false output of `Vendor Found?`
  - No downstream output
- **Version-specific requirements:**
  - Uses Gmail node version `2.1`
- **Edge cases or potential failure types:**
  - Email send failure
  - Potential formatting issues because the body contains markdown-like `**Action needed:**` inside plain text
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | ## QuickBooks: AI Invoice Processor Automatically processes vendor invoices received by email. |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | ### Who is this for? |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - Small/medium businesses using QuickBooks Online |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - Bookkeepers processing 20+ invoices/month |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - Accounting firms managing multiple clients |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | ### How it works |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 1. **Gmail** monitors for new emails with PDF attachments |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 2. **Extract from PDF** converts the PDF to text |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 3. **AI (Information Extractor)** extracts structured invoice data and classifies if it's actually an invoice |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 4. **QuickBooks** vendor search by name → gets Vendor ID |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 5. **QuickBooks** creates a Bill with extracted data |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 6. **PDF attachment** pipeline uploads the original invoice to the bill |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 7. **Confirmation email** sent back to the sender |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | ### Setup checklist |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - [ ] Connect **Gmail** credentials |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - [ ] Connect **OpenAI** API credentials |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - [ ] Connect **QuickBooks OAuth2** credentials |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - [ ] Edit the **Config** node with your values: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - `realmId` — your QuickBooks Company ID |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - `apTeamEmail` — AP team email for error notifications |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - `defaultExpenseAccountId` — see steps below |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - [ ] (Optional) Add Gmail label filter |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | ### How to find `defaultExpenseAccountId` |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 1. Log in to **QuickBooks Online** |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 2. Go to **Settings** (gear icon) → **Chart of Accounts** |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 3. Find your default expense account (e.g. "Accounts Payable", "Office Supplies", "Professional Services") |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 4. Click on the account → look at the URL: `...?accountId=`**`123`** |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | 5. Copy the number and paste it into the **Config** node |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | ## 1. Email Trigger & Config Monitors Gmail for new emails with PDF invoice attachments. |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | **Config node** centralizes all settings: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - `realmId` — Your QuickBooks Company ID (find in QB → Settings → Account) |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - `apTeamEmail` — Where error notifications go |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - `sendConfirmation` — Toggle confirmation emails on/off |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - `defaultExpenseAccountId` — QB expense account for line items |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | **Gmail Setup:** |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - Connect your Gmail account |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - Optionally filter by label or sender |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with PDF attachments and start workflow |  | Config | - `Simplify` is set to false to get headers + binary attachments |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | ## QuickBooks: AI Invoice Processor Automatically processes vendor invoices received by email. |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | ### Who is this for? |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - Small/medium businesses using QuickBooks Online |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - Bookkeepers processing 20+ invoices/month |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - Accounting firms managing multiple clients |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | ### How it works |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 1. **Gmail** monitors for new emails with PDF attachments |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 2. **Extract from PDF** converts the PDF to text |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 3. **AI (Information Extractor)** extracts structured invoice data and classifies if it's actually an invoice |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 4. **QuickBooks** vendor search by name → gets Vendor ID |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 5. **QuickBooks** creates a Bill with extracted data |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 6. **PDF attachment** pipeline uploads the original invoice to the bill |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 7. **Confirmation email** sent back to the sender |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | ### Setup checklist |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - [ ] Connect **Gmail** credentials |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - [ ] Connect **OpenAI** API credentials |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - [ ] Connect **QuickBooks OAuth2** credentials |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - [ ] Edit the **Config** node with your values: |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - `realmId` — your QuickBooks Company ID |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - `apTeamEmail` — AP team email for error notifications |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - `defaultExpenseAccountId` — see steps below |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - [ ] (Optional) Add Gmail label filter |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | ### How to find `defaultExpenseAccountId` |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 1. Log in to **QuickBooks Online** |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 2. Go to **Settings** (gear icon) → **Chart of Accounts** |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 3. Find your default expense account (e.g. "Accounts Payable", "Office Supplies", "Professional Services") |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 4. Click on the account → look at the URL: `...?accountId=`**`123`** |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | 5. Copy the number and paste it into the **Config** node |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | ## 1. Email Trigger & Config Monitors Gmail for new emails with PDF invoice attachments. |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | **Config node** centralizes all settings: |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - `realmId` — Your QuickBooks Company ID (find in QB → Settings → Account) |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - `apTeamEmail` — Where error notifications go |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - `sendConfirmation` — Toggle confirmation emails on/off |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - `defaultExpenseAccountId` — QB expense account for line items |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | **Gmail Setup:** |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - Connect your Gmail account |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - Optionally filter by label or sender |
| Config | n8n-nodes-base.set | Centralize workflow settings while preserving email/binary data | Gmail Trigger | Extract Text from PDF | - `Simplify` is set to false to get headers + binary attachments |
| Extract Text from PDF | n8n-nodes-base.extractFromFile | Convert PDF binary into text | Config | AI Extract Invoice Data |  |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | ## 2. AI Invoice Classification & Extraction Uses the **Information Extractor** node with OpenAI to: |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | 1. **Classify** if the PDF is actually an invoice (`is_invoice`) |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | 2. **Extract** structured data if it is: |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | - `vendor_name` — Company issuing the invoice |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | - `invoice_number` — Reference number |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | - `amount` — Total due |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | - `currency` — 3-letter code (USD, EUR, etc.) |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | - `due_date` / `txn_date` — Dates in YYYY-MM-DD |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | - `line_items` — Array of {description, amount, quantity} |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | Non-invoices (receipts, contracts, etc.) are silently skipped. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supply the LLM used for structured extraction |  | AI Extract Invoice Data | Invalid extractions (missing vendor/amount) route to AP team. |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | ## 2. AI Invoice Classification & Extraction Uses the **Information Extractor** node with OpenAI to: |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | 1. **Classify** if the PDF is actually an invoice (`is_invoice`) |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | 2. **Extract** structured data if it is: |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | - `vendor_name` — Company issuing the invoice |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | - `invoice_number` — Reference number |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | - `amount` — Total due |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | - `currency` — 3-letter code (USD, EUR, etc.) |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | - `due_date` / `txn_date` — Dates in YYYY-MM-DD |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | - `line_items` — Array of {description, amount, quantity} |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | Non-invoices (receipts, contracts, etc.) are silently skipped. |
| AI Extract Invoice Data | @n8n/n8n-nodes-langchain.informationExtractor | Classify document and extract invoice fields into structured JSON | Extract Text from PDF; OpenAI Chat Model | Is Invoice? | Invalid extractions (missing vendor/amount) route to AP team. |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | ## 2. AI Invoice Classification & Extraction Uses the **Information Extractor** node with OpenAI to: |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | 1. **Classify** if the PDF is actually an invoice (`is_invoice`) |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | 2. **Extract** structured data if it is: |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | - `vendor_name` — Company issuing the invoice |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | - `invoice_number` — Reference number |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | - `amount` — Total due |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | - `currency` — 3-letter code (USD, EUR, etc.) |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | - `due_date` / `txn_date` — Dates in YYYY-MM-DD |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | - `line_items` — Array of {description, amount, quantity} |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | Non-invoices (receipts, contracts, etc.) are silently skipped. |
| Is Invoice? | n8n-nodes-base.if | Route only documents classified as invoices | AI Extract Invoice Data | Validate Extracted Data | Invalid extractions (missing vendor/amount) route to AP team. |
| Validate Extracted Data | n8n-nodes-base.code | Validate required fields and normalize extracted values | Is Invoice? | Is Valid? |  |
| Is Valid? | n8n-nodes-base.if | Route valid extractions to QuickBooks and invalid ones to AP notification | Validate Extracted Data | Search vendor; Error: Extraction Failed |  |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | ## 3. QuickBooks: Vendor Search & Bill Creation Searches QuickBooks for the vendor by name extracted by AI. |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | - If no vendor found → error notification to AP team |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | **Bill Creation (Native QB node):** |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | - Uses native QuickBooks node for reliable OAuth2 auth |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | - Single line item with total amount (native node limitation) |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | - Invoice number included in line description |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | - DueDate, TxnDate, TotalAmt from AI extraction |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | - Returns Bill ID for PDF attachment step |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | **Note:** Native QB node supports only one line item per bill. |
| Search vendor | n8n-nodes-base.quickbooks | Look up vendor in QuickBooks by display name | Is Valid? | Vendor Found? | Individual line items are combined into the description field. |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | ## 3. QuickBooks: Vendor Search & Bill Creation Searches QuickBooks for the vendor by name extracted by AI. |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | - If no vendor found → error notification to AP team |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | **Bill Creation (Native QB node):** |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | - Uses native QuickBooks node for reliable OAuth2 auth |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | - Single line item with total amount (native node limitation) |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | - Invoice number included in line description |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | - DueDate, TxnDate, TotalAmt from AI extraction |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | - Returns Bill ID for PDF attachment step |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | **Note:** Native QB node supports only one line item per bill. |
| Vendor Found? | n8n-nodes-base.if | Check whether vendor lookup returned an ID | Search vendor | Prepare Bill Data; Error: Vendor Not Found | Individual line items are combined into the description field. |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | ## 3. QuickBooks: Vendor Search & Bill Creation Searches QuickBooks for the vendor by name extracted by AI. |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | - If no vendor found → error notification to AP team |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | **Bill Creation (Native QB node):** |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | - Uses native QuickBooks node for reliable OAuth2 auth |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | - Single line item with total amount (native node limitation) |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | - Invoice number included in line description |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | - DueDate, TxnDate, TotalAmt from AI extraction |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | - Returns Bill ID for PDF attachment step |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | **Note:** Native QB node supports only one line item per bill. |
| Prepare Bill Data | n8n-nodes-base.code | Build bill payload and combine line items into one description | Vendor Found? | Create Bill | Individual line items are combined into the description field. |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | ## 3. QuickBooks: Vendor Search & Bill Creation Searches QuickBooks for the vendor by name extracted by AI. |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | - If no vendor found → error notification to AP team |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | **Bill Creation (Native QB node):** |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | - Uses native QuickBooks node for reliable OAuth2 auth |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | - Single line item with total amount (native node limitation) |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | - Invoice number included in line description |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | - DueDate, TxnDate, TotalAmt from AI extraction |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | - Returns Bill ID for PDF attachment step |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | **Note:** Native QB node supports only one line item per bill. |
| Create Bill | n8n-nodes-base.quickbooks | Create QuickBooks bill from extracted invoice data | Prepare Bill Data | Build Attachment Metadata; Fetch PDF Binary | Individual line items are combined into the description field. |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | ## 4. PDF Attachment Pipeline Attaches the original PDF invoice to the created QuickBooks bill. |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | **Binary data flow:** |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | Binary (PDF) is lost after the AI extraction step, so the attachment nodes reference it by name from **Config** using `$('Config').item.binary.attachment_0`. |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | Both nodes are connected to Create Bill for **execution order** (they must run after the bill exists), but get binary data via named reference. |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | **The Force Inline Binary fix:** |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | n8n v2 stores binary as DB streams without Content-Length. QuickBooks upload API requires it. The Code node converts streams to inline base64. |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | **Pipeline:** |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | 1. Build metadata JSON (AttachableRef → Bill) — gets binary from Config by name |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | 2. Convert metadata to binary field (`myBinaryFile`) |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | 3. Fetch PDF Binary from Config by name |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | 4. Merge PDF binary + metadata binary into one item |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | 5. ⚠️ Force Inline Binary (stream → base64) |
| Build Attachment Metadata | n8n-nodes-base.code | Build QuickBooks attachment metadata JSON for the created bill | Create Bill | Metadata to Binary | 6. Upload multipart to QB `/upload` endpoint |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | ## 4. PDF Attachment Pipeline Attaches the original PDF invoice to the created QuickBooks bill. |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | **Binary data flow:** |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | Binary (PDF) is lost after the AI extraction step, so the attachment nodes reference it by name from **Config** using `$('Config').item.binary.attachment_0`. |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | Both nodes are connected to Create Bill for **execution order** (they must run after the bill exists), but get binary data via named reference. |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | **The Force Inline Binary fix:** |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | n8n v2 stores binary as DB streams without Content-Length. QuickBooks upload API requires it. The Code node converts streams to inline base64. |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | **Pipeline:** |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | 1. Build metadata JSON (AttachableRef → Bill) — gets binary from Config by name |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | 2. Convert metadata to binary field (`myBinaryFile`) |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | 3. Fetch PDF Binary from Config by name |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | 4. Merge PDF binary + metadata binary into one item |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | 5. ⚠️ Force Inline Binary (stream → base64) |
| Metadata to Binary | n8n-nodes-base.code | Convert attachment metadata JSON into binary form-data part | Build Attachment Metadata | Merge PDF + Metadata | 6. Upload multipart to QB `/upload` endpoint |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | ## 4. PDF Attachment Pipeline Attaches the original PDF invoice to the created QuickBooks bill. |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | **Binary data flow:** |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | Binary (PDF) is lost after the AI extraction step, so the attachment nodes reference it by name from **Config** using `$('Config').item.binary.attachment_0`. |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | Both nodes are connected to Create Bill for **execution order** (they must run after the bill exists), but get binary data via named reference. |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | **The Force Inline Binary fix:** |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | n8n v2 stores binary as DB streams without Content-Length. QuickBooks upload API requires it. The Code node converts streams to inline base64. |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | **Pipeline:** |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | 1. Build metadata JSON (AttachableRef → Bill) — gets binary from Config by name |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | 2. Convert metadata to binary field (`myBinaryFile`) |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | 3. Fetch PDF Binary from Config by name |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | 4. Merge PDF binary + metadata binary into one item |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | 5. ⚠️ Force Inline Binary (stream → base64) |
| Fetch PDF Binary | n8n-nodes-base.code | Recover original PDF binary for attachment upload | Create Bill | Merge PDF + Metadata | 6. Upload multipart to QB `/upload` endpoint |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | ## 4. PDF Attachment Pipeline Attaches the original PDF invoice to the created QuickBooks bill. |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | **Binary data flow:** |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | Binary (PDF) is lost after the AI extraction step, so the attachment nodes reference it by name from **Config** using `$('Config').item.binary.attachment_0`. |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | Both nodes are connected to Create Bill for **execution order** (they must run after the bill exists), but get binary data via named reference. |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | **The Force Inline Binary fix:** |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | n8n v2 stores binary as DB streams without Content-Length. QuickBooks upload API requires it. The Code node converts streams to inline base64. |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | **Pipeline:** |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | 1. Build metadata JSON (AttachableRef → Bill) — gets binary from Config by name |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | 2. Convert metadata to binary field (`myBinaryFile`) |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | 3. Fetch PDF Binary from Config by name |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | 4. Merge PDF binary + metadata binary into one item |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | 5. ⚠️ Force Inline Binary (stream → base64) |
| Merge PDF + Metadata | n8n-nodes-base.merge | Combine metadata binary and PDF binary into one item | Metadata to Binary; Fetch PDF Binary | Force Inline Binary | 6. Upload multipart to QB `/upload` endpoint |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | ## 4. PDF Attachment Pipeline Attaches the original PDF invoice to the created QuickBooks bill. |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | **Binary data flow:** |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | Binary (PDF) is lost after the AI extraction step, so the attachment nodes reference it by name from **Config** using `$('Config').item.binary.attachment_0`. |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | Both nodes are connected to Create Bill for **execution order** (they must run after the bill exists), but get binary data via named reference. |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | **The Force Inline Binary fix:** |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | n8n v2 stores binary as DB streams without Content-Length. QuickBooks upload API requires it. The Code node converts streams to inline base64. |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | **Pipeline:** |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | 1. Build metadata JSON (AttachableRef → Bill) — gets binary from Config by name |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | 2. Convert metadata to binary field (`myBinaryFile`) |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | 3. Fetch PDF Binary from Config by name |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | 4. Merge PDF binary + metadata binary into one item |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | 5. ⚠️ Force Inline Binary (stream → base64) |
| Force Inline Binary | n8n-nodes-base.code | Convert stream-backed binaries to inline base64 for QuickBooks upload | Merge PDF + Metadata | Upload PDF to Bill | 6. Upload multipart to QB `/upload` endpoint |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | ## 4. PDF Attachment Pipeline Attaches the original PDF invoice to the created QuickBooks bill. |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | **Binary data flow:** |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | Binary (PDF) is lost after the AI extraction step, so the attachment nodes reference it by name from **Config** using `$('Config').item.binary.attachment_0`. |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | Both nodes are connected to Create Bill for **execution order** (they must run after the bill exists), but get binary data via named reference. |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | **The Force Inline Binary fix:** |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | n8n v2 stores binary as DB streams without Content-Length. QuickBooks upload API requires it. The Code node converts streams to inline base64. |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | **Pipeline:** |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | 1. Build metadata JSON (AttachableRef → Bill) — gets binary from Config by name |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | 2. Convert metadata to binary field (`myBinaryFile`) |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | 3. Fetch PDF Binary from Config by name |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | 4. Merge PDF binary + metadata binary into one item |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | 5. ⚠️ Force Inline Binary (stream → base64) |
| Upload PDF to Bill | n8n-nodes-base.httpRequest | Upload original PDF and metadata to QuickBooks bill via multipart API | Force Inline Binary | Send Confirmation? | 6. Upload multipart to QB `/upload` endpoint |
| Send Confirmation? | n8n-nodes-base.if | Decide whether to email the original sender after success | Upload PDF to Bill | Send Confirmation Email | ## 5. Confirmation & Error Handling |
| Send Confirmation? | n8n-nodes-base.if | Decide whether to email the original sender after success | Upload PDF to Bill | Send Confirmation Email | **Success path:** |
| Send Confirmation? | n8n-nodes-base.if | Decide whether to email the original sender after success | Upload PDF to Bill | Send Confirmation Email | Sends confirmation email back to the invoice sender with bill details (toggleable via Config). |
| Send Confirmation? | n8n-nodes-base.if | Decide whether to email the original sender after success | Upload PDF to Bill | Send Confirmation Email | **Skip paths:** |
| Send Confirmation? | n8n-nodes-base.if | Decide whether to email the original sender after success | Upload PDF to Bill | Send Confirmation Email | - ⏭️ **Not an invoice** — PDF is not an invoice, silently skipped |
| Send Confirmation? | n8n-nodes-base.if | Decide whether to email the original sender after success | Upload PDF to Bill | Send Confirmation Email | **Error paths:** |
| Send Confirmation? | n8n-nodes-base.if | Decide whether to email the original sender after success | Upload PDF to Bill | Send Confirmation Email | - ❌ **AI extraction failed** — Missing required fields, forwards to AP team |
| Send Confirmation? | n8n-nodes-base.if | Decide whether to email the original sender after success | Upload PDF to Bill | Send Confirmation Email | - ❌ **Vendor not found** — Notifies AP team with the vendor name so they can create it in QuickBooks |
| Send Confirmation Email | n8n-nodes-base.gmail | Send success confirmation to original email sender | Send Confirmation? |  | ## 5. Confirmation & Error Handling |
| Send Confirmation Email | n8n-nodes-base.gmail | Send success confirmation to original email sender | Send Confirmation? |  | **Success path:** |
| Send Confirmation Email | n8n-nodes-base.gmail | Send success confirmation to original email sender | Send Confirmation? |  | Sends confirmation email back to the invoice sender with bill details (toggleable via Config). |
| Send Confirmation Email | n8n-nodes-base.gmail | Send success confirmation to original email sender | Send Confirmation? |  | **Skip paths:** |
| Send Confirmation Email | n8n-nodes-base.gmail | Send success confirmation to original email sender | Send Confirmation? |  | - ⏭️ **Not an invoice** — PDF is not an invoice, silently skipped |
| Send Confirmation Email | n8n-nodes-base.gmail | Send success confirmation to original email sender | Send Confirmation? |  | **Error paths:** |
| Send Confirmation Email | n8n-nodes-base.gmail | Send success confirmation to original email sender | Send Confirmation? |  | - ❌ **AI extraction failed** — Missing required fields, forwards to AP team |
| Send Confirmation Email | n8n-nodes-base.gmail | Send success confirmation to original email sender | Send Confirmation? |  | - ❌ **Vendor not found** — Notifies AP team with the vendor name so they can create it in QuickBooks |
| Error: Extraction Failed | n8n-nodes-base.gmail | Notify AP team that invoice extraction was invalid | Is Valid? |  | ## 5. Confirmation & Error Handling |
| Error: Extraction Failed | n8n-nodes-base.gmail | Notify AP team that invoice extraction was invalid | Is Valid? |  | **Success path:** |
| Error: Extraction Failed | n8n-nodes-base.gmail | Notify AP team that invoice extraction was invalid | Is Valid? |  | Sends confirmation email back to the invoice sender with bill details (toggleable via Config). |
| Error: Extraction Failed | n8n-nodes-base.gmail | Notify AP team that invoice extraction was invalid | Is Valid? |  | **Skip paths:** |
| Error: Extraction Failed | n8n-nodes-base.gmail | Notify AP team that invoice extraction was invalid | Is Valid? |  | - ⏭️ **Not an invoice** — PDF is not an invoice, silently skipped |
| Error: Extraction Failed | n8n-nodes-base.gmail | Notify AP team that invoice extraction was invalid | Is Valid? |  | **Error paths:** |
| Error: Extraction Failed | n8n-nodes-base.gmail | Notify AP team that invoice extraction was invalid | Is Valid? |  | - ❌ **AI extraction failed** — Missing required fields, forwards to AP team |
| Error: Extraction Failed | n8n-nodes-base.gmail | Notify AP team that invoice extraction was invalid | Is Valid? |  | - ❌ **Vendor not found** — Notifies AP team with the vendor name so they can create it in QuickBooks |
| Error: Vendor Not Found | n8n-nodes-base.gmail | Notify AP team that extracted vendor does not exist in QuickBooks | Vendor Found? |  | ## 5. Confirmation & Error Handling |
| Error: Vendor Not Found | n8n-nodes-base.gmail | Notify AP team that extracted vendor does not exist in QuickBooks | Vendor Found? |  | **Success path:** |
| Error: Vendor Not Found | n8n-nodes-base.gmail | Notify AP team that extracted vendor does not exist in QuickBooks | Vendor Found? |  | Sends confirmation email back to the invoice sender with bill details (toggleable via Config). |
| Error: Vendor Not Found | n8n-nodes-base.gmail | Notify AP team that extracted vendor does not exist in QuickBooks | Vendor Found? |  | **Skip paths:** |
| Error: Vendor Not Found | n8n-nodes-base.gmail | Notify AP team that extracted vendor does not exist in QuickBooks | Vendor Found? |  | - ⏭️ **Not an invoice** — PDF is not an invoice, silently skipped |
| Error: Vendor Not Found | n8n-nodes-base.gmail | Notify AP team that extracted vendor does not exist in QuickBooks | Vendor Found? |  | **Error paths:** |
| Error: Vendor Not Found | n8n-nodes-base.gmail | Notify AP team that extracted vendor does not exist in QuickBooks | Vendor Found? |  | - ❌ **AI extraction failed** — Missing required fields, forwards to AP team |
| Error: Vendor Not Found | n8n-nodes-base.gmail | Notify AP team that extracted vendor does not exist in QuickBooks | Vendor Found? |  | - ❌ **Vendor not found** — Notifies AP team with the vendor name so they can create it in QuickBooks |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Gmail Trigger node** named `Gmail Trigger`.
   - Type: Gmail Trigger
   - Connect Gmail OAuth2 credentials
   - Set polling to every 15 minutes
   - Disable `Simplify`
   - Enable attachment download
   - In Gmail filters, set query to `filename:pdf`
   - Optional: add label/sender filtering if needed

3. **Add a Set node** named `Config`.
   - Enable “include other fields”
   - Add the following fields:
     - `realmId` as string, set to your QuickBooks company ID
     - `apTeamEmail` as string, set to your AP/manual-review mailbox
     - `sendConfirmation` as boolean, default `false`
     - `defaultExpenseAccountId` as string, set to your fallback expense account ID
   - Connect `Gmail Trigger -> Config`

4. **Add an Extract From File node** named `Extract Text from PDF`.
   - Operation: PDF
   - Binary property: `attachment_0`
   - Enable join pages
   - Connect `Config -> Extract Text from PDF`

5. **Add an OpenAI Chat Model node** named `OpenAI Chat Model`.
   - Type: OpenAI Chat Model from LangChain
   - Connect OpenAI API credentials
   - Model: `gpt-4o`
   - Temperature: `0`

6. **Add an Information Extractor node** named `AI Extract Invoice Data`.
   - Set text input to `{{ $json.text }}`
   - Schema type: from JSON example
   - Use this schema example conceptually:
     - boolean `is_invoice`
     - string `vendor_name`
     - string `invoice_number`
     - number `amount`
     - string `currency`
     - string `due_date`
     - string `txn_date`
     - array `line_items` with objects containing `description`, `amount`, `quantity`
   - Connect:
     - `Extract Text from PDF -> AI Extract Invoice Data`
     - `OpenAI Chat Model -> AI Extract Invoice Data` using the AI language model connector

7. **Add an IF node** named `Is Invoice?`.
   - Condition type: boolean
   - Expression: `{{ $json.output.is_invoice }}`
   - Check that it is true
   - Connect `AI Extract Invoice Data -> Is Invoice?`
   - Leave the false branch unconnected to silently skip non-invoices

8. **Add a Code node** named `Validate Extracted Data`.
   - Mode: Run once for each item
   - Paste logic that:
     - reads `$json.output`
     - verifies `vendor_name`
     - verifies `amount > 0`
     - verifies `line_items` exists and is non-empty
     - normalizes `due_date` and `txn_date` to `YYYY-MM-DD`
     - defaults currency to `USD`
     - returns:
       - `valid`
       - `errors`
       - `data`
   - Connect `Is Invoice? (true) -> Validate Extracted Data`

9. **Add an IF node** named `Is Valid?`.
   - Condition: `{{ $json.valid }}` is true
   - Connect `Validate Extracted Data -> Is Valid?`

10. **Add a QuickBooks node** named `Search vendor`.
    - Connect QuickBooks OAuth2 credentials
    - Resource: Vendor
    - Operation: Get Many / Get All
    - Limit: `1`
    - Use query filter:
      - `WHERE DisplayName LIKE '%{{ $json.data.vendor_name }}%'`
    - Connect `Is Valid? (true) -> Search vendor`

11. **Add a Gmail node** named `Error: Extraction Failed`.
    - Connect Gmail OAuth2 credentials
    - Send email to: `{{ $('Config').item.json.apTeamEmail }}`
    - Subject:
      - `[Manual Processing] Could not extract invoice data from {{ $('Gmail Trigger').item.json.from }}`
    - Plain text body should include:
      - sender
      - email subject
      - joined extraction errors
      - original message ID
      - manual processing instruction
    - Connect `Is Valid? (false) -> Error: Extraction Failed`

12. **Add an IF node** named `Vendor Found?`.
    - Condition: string exists
    - Expression: `{{ $json.Id }}`
    - Connect `Search vendor -> Vendor Found?`

13. **Add a Gmail node** named `Error: Vendor Not Found`.
    - Connect Gmail OAuth2 credentials
    - Send email to: `{{ $('Config').item.json.apTeamEmail }}`
    - Subject:
      - `[Action Required] Unknown vendor: {{ $('Validate Extracted Data').item.json.data.vendor_name }}`
    - Body should include:
      - extracted vendor name
      - invoice number
      - amount and currency
      - original sender
      - manual next steps
      - original message ID
    - Connect `Vendor Found? (false) -> Error: Vendor Not Found`

14. **Add a Code node** named `Prepare Bill Data`.
    - Mode: Run once for each item
    - Build code that:
      - gets invoice data from `$('Validate Extracted Data').item.json.data`
      - gets vendor ID from `$('Search vendor').item.json.Id`
      - gets config from `$('Config').item.json`
      - combines all extracted line items into one description string
      - prefixes invoice number if available
      - truncates description to 4000 characters
      - returns:
        - `vendorId`
        - `amount`
        - `description`
        - `accountId`
        - `dueDate`
        - `txnDate`
        - `totalAmt`
    - Connect `Vendor Found? (true) -> Prepare Bill Data`

15. **Add a QuickBooks node** named `Create Bill`.
    - Connect QuickBooks OAuth2 credentials
    - Resource: Bill
    - Operation: Create
    - Vendor reference: `{{ $json.vendorId }}`
    - Add one line item:
      - Amount: `{{ $json.amount }}`
      - Account ID: `{{ $json.accountId }}`
      - Detail type: `AccountBasedExpenseLineDetail`
      - Description: `{{ $json.description }}`
    - Additional fields:
      - DueDate: `{{ $json.dueDate }}`
      - TxnDate: `{{ $json.txnDate }}`
      - TotalAmt: `{{ $json.totalAmt }}`
    - Connect `Prepare Bill Data -> Create Bill`

16. **Add a Code node** named `Build Attachment Metadata`.
    - Mode: Run once for each item
    - Build JSON containing:
      - `AttachableRef` with entity type `Bill` and created bill ID from `$('Create Bill').item.json.Id`
      - `FileName` from `$('Config').item.binary.attachment_0.fileName`
      - `ContentType` from `$('Config').item.binary.attachment_0.mimeType`
    - Connect `Create Bill -> Build Attachment Metadata`

17. **Add a Code node** named `Metadata to Binary`.
    - Mode: Run once for each item
    - Convert the metadata JSON to a UTF-8 buffer
    - Store it as binary property `myBinaryFile`
    - Use filename `data.json`
    - MIME type `application/json`
    - Connect `Build Attachment Metadata -> Metadata to Binary`

18. **Add a Code node** named `Fetch PDF Binary`.
    - Mode: Run once for each item
    - Return binary from `$('Config').item.binary`
    - Connect `Create Bill -> Fetch PDF Binary`

19. **Add a Merge node** named `Merge PDF + Metadata`.
    - Mode: Combine
    - Combine by position
    - Connect:
      - `Metadata to Binary -> Merge PDF + Metadata` input 1
      - `Fetch PDF Binary -> Merge PDF + Metadata` input 2

20. **Add a Code node** named `Force Inline Binary`.
    - This node is important for QuickBooks uploads where n8n stores binary as streams
    - In code:
      - iterate through all items
      - for `attachment_0`, get buffer with `this.helpers.getBinaryDataBuffer`
      - rewrite it to inline base64 binary
      - repeat for `myBinaryFile`
    - Connect `Merge PDF + Metadata -> Force Inline Binary`

21. **Add an HTTP Request node** named `Upload PDF to Bill`.
    - Method: POST
    - URL:
      - `https://quickbooks.api.intuit.com/v3/company/{{ $('Config').item.json.realmId }}/upload`
    - Authentication: predefined credential type
    - Credential type: QuickBooks OAuth2
    - Content type: multipart/form-data
    - Send body: enabled
    - Body parameters:
      - `file_metadata_01` as binary form field from `myBinaryFile`
      - `file_content_01` as binary form field from `attachment_0`
    - Optionally enable full response
    - Connect `Force Inline Binary -> Upload PDF to Bill`

22. **Add an IF node** named `Send Confirmation?`.
    - Condition: `{{ $('Config').item.json.sendConfirmation }}` is true
    - Connect `Upload PDF to Bill -> Send Confirmation?`

23. **Add a Gmail node** named `Send Confirmation Email`.
    - Connect Gmail OAuth2 credentials
    - Recipient: `{{ $('Gmail Trigger').item.json.from }}`
    - Subject:
      - `Invoice Processed: {{ $('Validate Extracted Data').item.json.data.invoice_number || 'N/A' }} → QuickBooks Bill #{{ $('Create Bill').item.json.Id }}`
    - Plain text body should confirm:
      - invoice processed automatically
      - PDF attached to bill
      - automated message notice
    - Connect `Send Confirmation? (true) -> Send Confirmation Email`

24. **Leave these intentional terminal branches as-is:**
    - `Is Invoice?` false branch: no connection
    - `Send Confirmation?` false branch: no connection
    - Error email nodes: terminal

25. **Configure credentials carefully.**
    - **Gmail OAuth2**
      - Required for both trigger and send-email nodes
      - Ensure inbox access and send-email scope are granted
    - **OpenAI API**
      - Required by `OpenAI Chat Model`
      - Verify model access for `gpt-4o`
    - **QuickBooks OAuth2**
      - Required by:
        - `Search vendor`
        - `Create Bill`
        - `Upload PDF to Bill`
      - Ensure correct QuickBooks company is authorized

26. **Set required business values in `Config`.**
    - `realmId`: QuickBooks company ID
    - `apTeamEmail`: monitored AP mailbox
    - `sendConfirmation`: true or false
    - `defaultExpenseAccountId`: valid expense account ID in QuickBooks

27. **Test with at least four scenarios.**
    - A valid PDF invoice from an existing vendor
    - A valid invoice whose vendor does not exist in QuickBooks
    - A non-invoice PDF attachment
    - A poor-quality or malformed PDF that causes extraction failure

28. **Pay special attention to attachment assumptions.**
    - The workflow expects the invoice PDF to be in `attachment_0`
    - If emails may contain multiple PDFs, you will need an additional loop/split design
    - If the file is scanned image-only, consider adding OCR before extraction

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow title: QuickBooks: AI Invoice Processor | Overall workflow purpose |
| Automatically processes vendor invoices received by email. | Overall workflow purpose |
| Intended for small/medium businesses, bookkeepers processing 20+ invoices/month, and accounting firms managing multiple clients. | Audience |
| Setup checklist includes Gmail, OpenAI, and QuickBooks OAuth2 credentials. | Deployment/setup |
| Required Config values: `realmId`, `apTeamEmail`, `defaultExpenseAccountId`, optional `sendConfirmation`. | Deployment/setup |
| To find `defaultExpenseAccountId`: QuickBooks Online → Settings → Chart of Accounts → open account → copy numeric `accountId` from URL. | QuickBooks setup |
| Native QuickBooks bill creation is implemented with a single line item. Individual invoice line items are collapsed into description text. | Design limitation |
| Non-invoice PDFs are silently skipped by design. | Processing behavior |
| Invalid AI extraction and vendor-not-found cases are routed to AP email notifications for manual handling. | Error handling |
| The PDF upload stage uses the QuickBooks `/upload` endpoint and an inline-binary workaround for n8n binary stream handling. | Technical implementation |