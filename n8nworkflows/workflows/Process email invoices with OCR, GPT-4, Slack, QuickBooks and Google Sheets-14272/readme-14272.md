Process email invoices with OCR, GPT-4, Slack, QuickBooks and Google Sheets

https://n8nworkflows.xyz/workflows/process-email-invoices-with-ocr--gpt-4--slack--quickbooks-and-google-sheets-14272


# Process email invoices with OCR, GPT-4, Slack, QuickBooks and Google Sheets

# 1. Workflow Overview

This workflow automates invoice intake from Gmail, extracts invoice text from attachments, uses GPT-4 to structure the invoice into JSON, validates financial totals, masks sensitive fields, optionally routes low-confidence or inconsistent invoices for human review in Slack, creates a bill in QuickBooks, logs processing activity to Google Sheets, and sends a final Slack success notification.

It is intended for accounts payable or finance automation scenarios where invoice emails arrive in a monitored mailbox and need structured extraction, review control, bookkeeping entry, and audit logging.

## 1.1 Email Intake and Runtime Configuration
The workflow starts from Gmail, polling a labeled inbox for new emails with attachments. It then enriches the execution with static configuration values and invoice metadata derived from the email and attachment.

## 1.2 Attachment Text Extraction
The workflow checks whether the first attachment appears to be a PDF. PDFs are parsed directly; non-PDF files are sent to an OCR API over HTTP. Both routes converge into a single merged payload used for downstream AI extraction.

## 1.3 AI-Based Invoice Structuring
The extracted text is passed to a LangChain Agent backed by GPT-4o and a structured output parser. The AI is instructed to return only structured invoice JSON, including line items and field-level confidence scores.

## 1.4 Validation and PII Masking
A code node recalculates line-item totals, subtotal, tax, and grand total, then compares them against extracted values within a configurable tolerance. A second code node masks PAN, GST, and bank account data while preserving originals in a temporary field.

## 1.5 Review Decision and Human Approval
If validation fails or any confidence score is below a configured threshold, a Slack review message is sent and the workflow pauses until resumed via webhook. Otherwise, it proceeds directly to bookkeeping.

## 1.6 QuickBooks Sync, Audit Logging, and Notification
Approved or auto-approved invoices are submitted to QuickBooks as bills, then logged to Google Sheets for audit tracking, and finally announced in Slack as successfully processed.

---

# 2. Block-by-Block Analysis

## 2.1 Email Intake and Runtime Configuration

### Overview
This block receives incoming invoice emails from Gmail and prepares shared workflow settings and initial metadata. It establishes all global values later used for OCR, confidence thresholds, Slack routing, and audit storage.

### Nodes Involved
- Invoice Email Trigger
- Workflow Configuration
- Capture Invoice Metadata

### Node Details

#### Invoice Email Trigger
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Watches a Gmail mailbox for new messages matching a label filter.
- **Configuration choices:**
  - Advanced mode (`simple: false`)
  - Filters by Gmail label: `AP Inbox`
  - Downloads attachments automatically
  - Polls every 5 minutes
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Entry point node
  - Outputs to **Workflow Configuration**
- **Version-specific requirements:**
  - Uses type version `1.3`
  - Requires Gmail credentials with permission to read mailbox content and attachments
- **Edge cases or potential failure types:**
  - Gmail authentication failure or expired OAuth token
  - Label not existing in the connected mailbox
  - Email without attachments
  - Multiple attachments present but downstream logic only references `attachment_0`
  - Delayed trigger behavior due to polling model
- **Sub-workflow reference:** None

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Stores reusable configuration values as fields in the execution payload.
- **Configuration choices:**
  - Adds:
    - `ocrApiUrl`
    - `validationTolerance` = `1`
    - `confidenceThreshold` = `0.85`
    - `slackChannel`
    - `auditSheetId`
  - Preserves incoming fields with `includeOtherFields: true`
- **Key expressions or variables used:** Static placeholders for environment-specific values
- **Input and output connections:**
  - Input from **Invoice Email Trigger**
  - Output to **Capture Invoice Metadata**
- **Version-specific requirements:**
  - Uses Set node version `3.4`
- **Edge cases or potential failure types:**
  - Placeholder values left unreplaced
  - Invalid OCR endpoint URL
  - Invalid Slack channel ID
  - Invalid Google Sheets document ID
- **Sub-workflow reference:** None

#### Capture Invoice Metadata
- **Type and technical role:** `n8n-nodes-base.set`  
  Derives invoice-related metadata from the incoming Gmail message and attachment.
- **Configuration choices:**
  - `invoiceSource` = `email`
  - `receivedTimestamp` = current ISO timestamp
  - `fileHash` = SHA-256 hash of binary attachment data if `attachment_0` exists
  - `vendorName` = sender name, else sender address, else `Unknown`
  - Keeps all original fields
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
  - `{{ $binary.attachment_0 ? $hash($binary.attachment_0.data, 'sha256') : '' }}`
  - `{{ $json.from?.name || $json.from?.address || 'Unknown' }}`
- **Input and output connections:**
  - Input from **Workflow Configuration**
  - Output to **Check File Type**
- **Version-specific requirements:**
  - Uses Set node version `3.4`
  - Depends on binary attachment data being downloaded by Gmail Trigger
- **Edge cases or potential failure types:**
  - No attachment means empty `fileHash`
  - Sender object shape may differ depending on Gmail node output
  - Vendor name inferred from sender may not match actual invoice vendor
- **Sub-workflow reference:** None

---

## 2.2 Attachment Text Extraction

### Overview
This block determines whether the attachment should be parsed as a PDF or submitted to an OCR API as an image/document file. It then merges both possible extraction routes into a single payload for AI processing.

### Nodes Involved
- Check File Type
- Extract Text from PDF
- OCR for Images
- Merge OCR Results

### Node Details

#### Check File Type
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches processing based on whether the MIME type of `attachment_0` contains `pdf`.
- **Configuration choices:**
  - Condition checks string containment against:
    - `{{ $('Capture Invoice Metadata').item.json.attachment_0.mimeType }}`
  - Right value: `pdf`
- **Key expressions or variables used:**
  - References `attachment_0.mimeType` from the prior node’s item
- **Input and output connections:**
  - Input from **Capture Invoice Metadata**
  - True output to **Extract Text from PDF**
  - False output to **OCR for Images**
- **Version-specific requirements:**
  - Uses IF node version `2.3`
- **Edge cases or potential failure types:**
  - `attachment_0.mimeType` may be missing
  - Some PDFs may have unusual MIME values
  - Non-image, non-PDF attachments still go to OCR route
- **Sub-workflow reference:** None

#### Extract Text from PDF
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts text directly from a PDF attachment.
- **Configuration choices:**
  - Operation: `pdf`
  - Binary property: `attachment_0`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from **Check File Type** (PDF branch)
  - Output to **Merge OCR Results**
- **Version-specific requirements:**
  - Uses version `1.1`
  - Needs valid PDF binary in `attachment_0`
- **Edge cases or potential failure types:**
  - Scanned PDFs with no embedded text may produce weak or empty extraction
  - Corrupt PDF files
  - Password-protected PDFs
- **Sub-workflow reference:** None

#### OCR for Images
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the attachment to an external OCR API as multipart form-data.
- **Configuration choices:**
  - Method: `POST`
  - URL from `Workflow Configuration.ocrApiUrl`
  - Response expected as JSON
  - Body type: multipart form-data
  - Form field `file` populated from binary `attachment_0`
  - Authentication mode: generic credential type
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.ocrApiUrl }}`
- **Input and output connections:**
  - Input from **Check File Type** (non-PDF branch)
  - Output to **Merge OCR Results**
- **Version-specific requirements:**
  - Uses HTTP Request version `4.3`
  - Requires matching credential type depending on OCR provider
- **Edge cases or potential failure types:**
  - OCR endpoint not configured
  - API auth failure
  - Unsupported file format
  - OCR API timeout or rate limit
  - Response shape may not contain `ParsedResults[0].ParsedText`
- **Sub-workflow reference:** None

#### Merge OCR Results
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the output of the PDF extraction route and OCR route into a unified item.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input 0 from **Extract Text from PDF**
  - Input 1 from **OCR for Images**
  - Output to **AI Invoice Extractor**
- **Version-specific requirements:**
  - Uses Merge version `3.2`
- **Edge cases or potential failure types:**
  - Because only one branch executes, combined output may depend on n8n merge behavior with missing counterpart input
  - If either extraction result is malformed, downstream text expression may be empty
- **Sub-workflow reference:** None

---

## 2.3 AI-Based Invoice Structuring

### Overview
This block transforms raw extracted text into a structured invoice object using GPT-4o and a manual JSON schema. It is the core intelligence layer of the workflow.

### Nodes Involved
- AI Invoice Extractor
- OpenAI GPT-4
- Invoice Schema Parser
- OpenAI GPT-4 for Parser

### Node Details

#### AI Invoice Extractor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain agent node that prompts an LLM to extract invoice fields into structured JSON.
- **Configuration choices:**
  - Input text:
    - `{{ $json.extractedText || $json.ParsedResults?.[0]?.ParsedText || '' }}`
  - Prompt type: defined directly in node
  - Includes system message instructing:
    - extract header fields
    - line items
    - tax and totals
    - low-confidence fields
    - no calculations
    - return only structured JSON
  - Output parser enabled
- **Key expressions or variables used:**
  - Pulls text from either PDF extraction output or OCR API response
- **Input and output connections:**
  - Main input from **Merge OCR Results**
  - AI language model input from **OpenAI GPT-4**
  - AI output parser input from **Invoice Schema Parser**
  - Main output to **Validate and Recalculate Totals**
- **Version-specific requirements:**
  - Uses LangChain Agent version `3`
  - Requires compatible n8n LangChain package support
- **Edge cases or potential failure types:**
  - Empty extraction text leads to null-heavy output or parser failure
  - LLM may hallucinate fields despite prompt
  - Schema mismatch can break downstream validation
  - Token/context issues for very long invoices
- **Sub-workflow reference:** None

#### OpenAI GPT-4
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Main language model powering the agent extraction.
- **Configuration choices:**
  - Model: `gpt-4o`
  - No built-in tools configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as `ai_languageModel` to **AI Invoice Extractor**
- **Version-specific requirements:**
  - Uses version `1.3`
  - Requires OpenAI credentials
  - Model availability depends on account/API access
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model not available
  - OpenAI rate limits or API errors
- **Sub-workflow reference:** None

#### Invoice Schema Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a manual JSON schema on the LLM output.
- **Configuration choices:**
  - `autoFix: true`
  - Manual schema defining:
    - invoice header fields
    - vendor identifiers
    - subtotal, tax, total
    - line items array
    - confidence object with field-level numeric scores
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI parser output to **AI Invoice Extractor**
  - AI language model input from **OpenAI GPT-4 for Parser**
- **Version-specific requirements:**
  - Uses version `1.3`
  - Requires parser-compatible AI model for autofix
- **Edge cases or potential failure types:**
  - Schema does not explicitly require fields, so output may omit some
  - Numeric fields may be returned as strings and require autofix
  - Deeply malformed output may still fail parsing
- **Sub-workflow reference:** None

#### OpenAI GPT-4 for Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Secondary language model used by the structured output parser to repair invalid JSON when necessary.
- **Configuration choices:**
  - Model: `gpt-4o`
  - No built-in tools
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as `ai_languageModel` to **Invoice Schema Parser**
- **Version-specific requirements:**
  - Uses version `1.3`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - Same OpenAI API failure risks as primary model
  - Extra cost and latency during parser autofix
- **Sub-workflow reference:** None

---

## 2.4 Validation and PII Masking

### Overview
This block verifies the arithmetic consistency of the extracted invoice and masks sensitive vendor information before routing the payload further. It reduces the risk of bad accounting entries and accidental exposure of tax or banking identifiers.

### Nodes Involved
- Validate and Recalculate Totals
- Mask PII Data

### Node Details

#### Validate and Recalculate Totals
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript validation logic that recomputes invoice amounts.
- **Configuration choices:**
  - Reads invoice JSON from first input item
  - Reads `validationTolerance` from **Workflow Configuration**
  - For each line item:
    - calculates `quantity * rate`
    - compares against extracted `amount`
    - adds `calculatedAmount` and `amountMatch`
  - Recalculates:
    - subtotal
    - tax using `taxRate`
    - total
  - Generates validation flags:
    - `subtotalMatch`
    - `taxMatch`
    - `totalMatch`
    - `validationPassed`
- **Key expressions or variables used:**
  - `$('Workflow Configuration').first().json.validationTolerance`
- **Input and output connections:**
  - Input from **AI Invoice Extractor**
  - Output to **Mask PII Data**
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Missing `lineItems` handled as empty array
  - Missing numeric values treated as `0`
  - If invoice tax is not based on a single taxRate, validation may incorrectly fail
  - Currency formatting from AI must already be normalized to numbers
- **Sub-workflow reference:** None

#### Mask PII Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Masks tax and banking identifiers while preserving the unmasked values in a side field.
- **Configuration choices:**
  - Defines helper functions:
    - `maskPAN`
    - `maskBankAccount`
    - `maskGST`
  - Creates `_originalPII` containing original values
  - Replaces:
    - `vendorPAN`
    - `vendorBankAccount`
    - `vendorGST`
  - Sets `_piiMasked: true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from **Validate and Recalculate Totals**
  - Output to **Check if Review Needed**
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Masking logic assumes simple string lengths, not country-specific formatting
  - `_originalPII` is not actually encrypted despite code comment
  - If downstream systems should never receive original PII, this field is a compliance risk
- **Sub-workflow reference:** None

---

## 2.5 Review Decision and Human Approval

### Overview
This block determines whether the invoice can be auto-processed or requires human intervention. It uses validation results and confidence scores, sends a Slack request when needed, and pauses the workflow until externally resumed.

### Nodes Involved
- Check if Review Needed
- Send Review Request
- Wait for Human Review

### Node Details

#### Check if Review Needed
- **Type and technical role:** `n8n-nodes-base.if`  
  Decides whether to route the invoice to manual review.
- **Configuration choices:**
  - Boolean expression evaluates true when:
    - validation failed, or
    - any confidence score is below configured threshold
- **Key expressions or variables used:**
  - `{{ !$json.validation?.validationPassed || Object.values($json.confidence || {}).some(c => c < $('Workflow Configuration').first().json.confidenceThreshold) }}`
- **Input and output connections:**
  - Input from **Mask PII Data**
  - True branch to **Send Review Request**
  - False branch to **Create QuickBooks Bill**
- **Version-specific requirements:**
  - Uses IF node version `2.3`
- **Edge cases or potential failure types:**
  - If `confidence` is missing or empty, `some(...)` returns false and may allow auto-processing
  - Confidence values must be numeric
- **Sub-workflow reference:** None

#### Send Review Request
- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a Slack message asking for invoice review.
- **Configuration choices:**
  - Sends formatted message including:
    - invoice number
    - vendor
    - total
    - issue reason
    - validation details
  - Sends to channel ID from configuration
- **Key expressions or variables used:**
  - `{{ $json.invoiceNumber }}`
  - `{{ $json.vendorName }}`
  - `{{ $json.total }}`
  - `{{ !$json.validation?.validationPassed ? "Validation failed - amounts don't match" : "Low confidence extraction" }}`
  - channel from `{{ $('Workflow Configuration').first().json.slackChannel }}`
- **Input and output connections:**
  - Input from **Check if Review Needed**
  - Output to **Wait for Human Review**
- **Version-specific requirements:**
  - Uses Slack node version `2.4`
  - Requires Slack credentials with permission to post to target channel
- **Edge cases or potential failure types:**
  - Invalid Slack channel ID
  - Bot not invited to channel
  - Message does not include an actual approval action mechanism
- **Sub-workflow reference:** None

#### Wait for Human Review
- **Type and technical role:** `n8n-nodes-base.wait`  
  Suspends execution until resumed via webhook.
- **Configuration choices:**
  - Resume mode: `webhook`
  - Response mode: `lastNode`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from **Send Review Request**
  - Output to **Create QuickBooks Bill**
- **Version-specific requirements:**
  - Uses version `1.1`
  - Requires external logic or human process to call the generated resume webhook
- **Edge cases or potential failure types:**
  - No approval/rejection payload is implemented in this workflow
  - Workflow resumes regardless of approval semantics unless external caller enforces them
  - Long wait periods may complicate operations
- **Sub-workflow reference:** None

---

## 2.6 QuickBooks Sync, Audit Logging, and Notification

### Overview
This block creates the accounting record in QuickBooks, builds an audit log entry, writes it to Google Sheets, and notifies Slack of success. It is the final operationalization stage.

### Nodes Involved
- Create QuickBooks Bill
- Prepare Audit Log Entry
- Log to Audit Trail
- Send Success Notification

### Node Details

#### Create QuickBooks Bill
- **Type and technical role:** `n8n-nodes-base.quickbooks`  
  Creates a bill in QuickBooks from extracted invoice data.
- **Configuration choices:**
  - Resource: `bill`
  - Operation: `create`
  - Vendor reference from `vendorName`
  - Uses only the first line item:
    - amount
    - itemId
    - description
  - Additional fields:
    - `DueDate` from `dueDate`
    - `TxnDate` from `invoiceDate`
    - `TotalAmt` from `totalAmount`
- **Key expressions or variables used:**
  - `{{ $json.lineItems[0].amount }}`
  - `{{ $json.lineItems[0].itemId }}`
  - `{{ $json.lineItems[0].description }}`
  - `{{ $json.vendorName }}`
  - `{{ $json.dueDate }}`
  - `{{ $json.invoiceDate }}`
  - `{{ $json.totalAmount }}`
- **Input and output connections:**
  - Inputs from:
    - **Check if Review Needed** false branch
    - **Wait for Human Review**
  - Output to **Prepare Audit Log Entry**
- **Version-specific requirements:**
  - Uses QuickBooks node version `1`
  - Requires QuickBooks OAuth credentials and valid company connection
- **Edge cases or potential failure types:**
  - `vendorName` may not be a valid QuickBooks `VendorRef`
  - `lineItems[0].itemId` is never generated elsewhere in workflow
  - Only first line item is posted; additional line items are ignored
  - Uses `totalAmount`, but the extracted schema defines `total`, not `totalAmount`
  - Human review does not alter payload in current design
- **Sub-workflow reference:** None

#### Prepare Audit Log Entry
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds audit metadata after QuickBooks creation.
- **Configuration choices:**
  - `auditTimestamp` = current ISO timestamp
  - `quickbooksId` = QuickBooks response `id`
  - `processingStatus` = `completed`
  - `humanReviewed` = true if **Wait for Human Review** has data
  - `validationResults` = JSON string of validation object
  - `confidenceScores` = JSON string of confidence object
  - `maskedData` = `true`
  - Keeps all other fields
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
  - `{{ $json.id }}`
  - `{{ $('Wait for Human Review').first() ? true : false }}`
  - `{{ JSON.stringify($json.validation) }}`
  - `{{ JSON.stringify($json.confidence) }}`
- **Input and output connections:**
  - Input from **Create QuickBooks Bill**
  - Output to **Log to Audit Trail**
- **Version-specific requirements:**
  - Uses Set node version `3.4`
- **Edge cases or potential failure types:**
  - Depending on execution path, `$('Wait for Human Review').first()` may throw or behave unexpectedly if node never ran
  - QuickBooks response field may differ from `id`
- **Sub-workflow reference:** None

#### Log to Audit Trail
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends or updates an audit row in Google Sheets.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document ID from workflow config
  - Sheet name: `Audit Log`
  - Matching column: `invoiceNumber`
  - Explicit mappings:
    - `invoiceNumber`
    - `vendorName`
    - `total`
    - `auditTimestamp`
    - `quickbooksId`
    - `processingStatus`
    - `humanReviewed`
    - `validationResults`
    - `confidenceScores`
    - `fileHash`
- **Key expressions or variables used:**
  - Uses `{{ $('Workflow Configuration').first().json.auditSheetId }}`
  - Pulls mapped fields from current item JSON
- **Input and output connections:**
  - Input from **Prepare Audit Log Entry**
  - Output to **Send Success Notification**
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.7`
  - Requires Google Sheets credentials
  - Target sheet and columns must exist or be compatible with schema
- **Edge cases or potential failure types:**
  - Invalid spreadsheet ID
  - Missing worksheet named `Audit Log`
  - Duplicate invoice numbers overwrite/update same row due to matching column choice
- **Sub-workflow reference:** None

#### Send Success Notification
- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a completion message to Slack.
- **Configuration choices:**
  - Message includes:
    - invoice number
    - vendor
    - total
    - QuickBooks ID
    - whether human reviewed
  - Sends to configured Slack channel
- **Key expressions or variables used:**
  - `{{ $json.invoiceNumber }}`
  - `{{ $json.vendorName }}`
  - `{{ $json.total }}`
  - `{{ $json.quickbooksId }}`
  - `{{ $json.humanReviewed ? "Yes" : "No" }}`
  - channel from `{{ $('Workflow Configuration').first().json.slackChannel }}`
- **Input and output connections:**
  - Input from **Log to Audit Trail**
  - Final node
- **Version-specific requirements:**
  - Uses Slack node version `2.4`
- **Edge cases or potential failure types:**
  - Slack auth or channel access issues
  - Notification may succeed even if previous audit data contains inconsistencies
- **Sub-workflow reference:** None

---

## 2.7 Documentation and Visual Annotation Nodes

### Overview
These nodes are non-executable sticky notes placed on the canvas to explain each stage. They are useful for maintainers and should be preserved if recreating the visual design.

### Nodes Involved
- Sticky Note1
- Sticky Note
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9
- Sticky Note10
- Sticky Note11

### Node Details

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for email trigger area
- **Content:** `## Email Trigger Watches inbox and downloads invoice attachments automatically.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for configuration and metadata area
- **Content:** `## Configuration Stores OCR API, thresholds, Slack channel, and audit sheet settings. and Extracts sender info, timestamp, file hash, and vendor details.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for file-processing area
- **Content:** `## File Processing Routes PDF to extractor or images to OCR API for text extraction.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for AI extraction area
- **Content:** `## AI Extraction AI extracts structured invoice data with confidence scores.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for validation/masking area
- **Content:** `## Validation and Masking Recalculates totals and checks for mismatches and Masks sensitive data like PAN, GST, and bank details.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for review decision area
- **Content:** `## Review Check Flags invoices needing human review based on validation or confidence.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for human review area
- **Content:** `## Human Review Sends Slack alert and waits for approval if issues are found.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for accounting sync area
- **Content:** `## Accounting Sync Creates bill in QuickBooks for approved invoices.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for audit logging area
- **Content:** `## Audit Logging Logs processing details and results to Google Sheets.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note10
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** General workflow note
- **Content:**  
  `## How it works Monitors email for invoice attachments, extracts data using OCR and AI, validates totals, masks sensitive fields, and decides whether to auto-process or send for review.`  
  `## Setup Connect Gmail, OpenAI, OCR API, Slack, QuickBooks, and Google Sheets. Configure thresholds, audit sheet, and test with sample invoices before enabling..`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note11
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Comment for notification area
- **Content:** `## Notifications Sends success message after invoice is processed.`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Invoice Email Trigger | gmailTrigger | Poll Gmail for invoice emails and download attachments |  | Workflow Configuration | ## Email Trigger<br>Watches inbox and downloads invoice attachments automatically. |
| Workflow Configuration | set | Store runtime config values such as OCR URL, thresholds, Slack channel, and Sheet ID | Invoice Email Trigger | Capture Invoice Metadata | ## Configuration<br>Stores OCR API, thresholds, Slack channel, and audit sheet settings. and Extracts sender info, timestamp, file hash, and vendor details. |
| Capture Invoice Metadata | set | Add source, received timestamp, file hash, and inferred vendor | Workflow Configuration | Check File Type | ## Configuration<br>Stores OCR API, thresholds, Slack channel, and audit sheet settings. and Extracts sender info, timestamp, file hash, and vendor details. |
| Check File Type | if | Route attachment to PDF extraction or OCR | Capture Invoice Metadata | Extract Text from PDF, OCR for Images | ## File Processing<br>Routes PDF to extractor or images to OCR API for text extraction. |
| Extract Text from PDF | extractFromFile | Extract text directly from PDF attachment | Check File Type | Merge OCR Results | ## File Processing<br>Routes PDF to extractor or images to OCR API for text extraction. |
| OCR for Images | httpRequest | Send non-PDF attachment to OCR API | Check File Type | Merge OCR Results | ## File Processing<br>Routes PDF to extractor or images to OCR API for text extraction. |
| Merge OCR Results | merge | Combine extraction branches into one item | Extract Text from PDF, OCR for Images | AI Invoice Extractor | ## File Processing<br>Routes PDF to extractor or images to OCR API for text extraction. |
| AI Invoice Extractor | @n8n/n8n-nodes-langchain.agent | Convert extracted text into structured invoice JSON | Merge OCR Results; OpenAI GPT-4; Invoice Schema Parser | Validate and Recalculate Totals | ## AI Extraction<br>AI extracts structured invoice data with confidence scores. |
| OpenAI GPT-4 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main LLM for invoice extraction |  | AI Invoice Extractor | ## AI Extraction<br>AI extracts structured invoice data with confidence scores. |
| Invoice Schema Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON schema on AI output | OpenAI GPT-4 for Parser | AI Invoice Extractor | ## AI Extraction<br>AI extracts structured invoice data with confidence scores. |
| OpenAI GPT-4 for Parser | @n8n/n8n-nodes-langchain.lmChatOpenAi | Parser repair model for structured output autofix |  | Invoice Schema Parser |  |
| Validate and Recalculate Totals | code | Recompute totals and generate validation flags | AI Invoice Extractor | Mask PII Data | ## Validation and Masking<br>Recalculates totals and checks for mismatches and Masks sensitive data like PAN, GST, and bank details. |
| Mask PII Data | code | Mask PAN, GST, and bank account details | Validate and Recalculate Totals | Check if Review Needed | ## Validation and Masking<br>Recalculates totals and checks for mismatches and Masks sensitive data like PAN, GST, and bank details. |
| Check if Review Needed | if | Decide whether invoice requires manual review | Mask PII Data | Send Review Request, Create QuickBooks Bill | ## Review Check<br>Flags invoices needing human review based on validation or confidence. |
| Send Review Request | slack | Notify Slack channel that manual review is needed | Check if Review Needed | Wait for Human Review | ## Human Review<br>Sends Slack alert and waits for approval if issues are found. |
| Wait for Human Review | wait | Pause workflow until resumed via webhook | Send Review Request | Create QuickBooks Bill | ## Human Review<br>Sends Slack alert and waits for approval if issues are found. |
| Create QuickBooks Bill | quickbooks | Create bill in QuickBooks | Check if Review Needed, Wait for Human Review | Prepare Audit Log Entry | ## Accounting Sync<br>Creates bill in QuickBooks for approved invoices. |
| Prepare Audit Log Entry | set | Build audit metadata after QuickBooks creation | Create QuickBooks Bill | Log to Audit Trail | ## Audit Logging<br>Logs processing details and results to Google Sheets. |
| Log to Audit Trail | googleSheets | Append or update invoice audit row in Google Sheets | Prepare Audit Log Entry | Send Success Notification | ## Audit Logging<br>Logs processing details and results to Google Sheets. |
| Send Success Notification | slack | Post Slack success message after processing | Log to Audit Trail |  | ## Notifications<br>Sends success message after invoice is processed. |
| Sticky Note1 | stickyNote | Visual documentation |  |  |  |
| Sticky Note | stickyNote | Visual documentation |  |  |  |
| Sticky Note3 | stickyNote | Visual documentation |  |  |  |
| Sticky Note4 | stickyNote | Visual documentation |  |  |  |
| Sticky Note5 | stickyNote | Visual documentation |  |  |  |
| Sticky Note6 | stickyNote | Visual documentation |  |  |  |
| Sticky Note7 | stickyNote | Visual documentation |  |  |  |
| Sticky Note8 | stickyNote | Visual documentation |  |  |  |
| Sticky Note9 | stickyNote | Visual documentation |  |  |  |
| Sticky Note10 | stickyNote | Visual documentation |  |  |  |
| Sticky Note11 | stickyNote | Visual documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **Process email invoices with OCR, GPT-4, Slack, QuickBooks and Google Sheets**

2. **Add a Gmail Trigger node** named **Invoice Email Trigger**.
   - Type: Gmail Trigger
   - Set it to advanced mode rather than simple mode
   - Add a label filter with `AP Inbox`
   - Enable attachment download
   - Set polling to every 5 minutes
   - Configure Gmail OAuth2 credentials with mailbox read access

3. **Add a Set node** named **Workflow Configuration** after the trigger.
   - Enable keeping other incoming fields
   - Add these fields:
     - `ocrApiUrl` as string
     - `validationTolerance` as number, default `1`
     - `confidenceThreshold` as number, default `0.85`
     - `slackChannel` as string
     - `auditSheetId` as string
   - Replace placeholder values with real environment values

4. **Add another Set node** named **Capture Invoice Metadata**.
   - Keep other fields
   - Add:
     - `invoiceSource` = `email`
     - `receivedTimestamp` = `{{ $now.toISO() }}`
     - `fileHash` = `{{ $binary.attachment_0 ? $hash($binary.attachment_0.data, 'sha256') : '' }}`
     - `vendorName` = `{{ $json.from?.name || $json.from?.address || 'Unknown' }}`
   - Connect **Workflow Configuration** to it

5. **Add an IF node** named **Check File Type**.
   - Condition type: string contains
   - Left value: `{{ $('Capture Invoice Metadata').item.json.attachment_0.mimeType }}`
   - Right value: `pdf`
   - Connect **Capture Invoice Metadata** to it

6. **Add an Extract From File node** named **Extract Text from PDF**.
   - Operation: PDF
   - Binary property: `attachment_0`
   - Connect the true/PDF branch of **Check File Type** to it

7. **Add an HTTP Request node** named **OCR for Images**.
   - Method: `POST`
   - URL: `{{ $('Workflow Configuration').first().json.ocrApiUrl }}`
   - Response format: JSON
   - Body type: multipart/form-data
   - Add form-data parameter:
     - name: `file`
     - type: binary
     - input binary property: `attachment_0`
   - Set authentication to match your OCR provider
   - Connect the false/non-PDF branch of **Check File Type** to it

8. **Add a Merge node** named **Merge OCR Results**.
   - Mode: Combine
   - Combine by: Combine All
   - Connect **Extract Text from PDF** to input 1
   - Connect **OCR for Images** to input 2

9. **Add a LangChain Agent node** named **AI Invoice Extractor**.
   - Set prompt type to define directly
   - Text input:
     - `{{ $json.extractedText || $json.ParsedResults?.[0]?.ParsedText || '' }}`
   - Add a system message instructing the model to:
     - extract invoice fields
     - extract line items
     - extract tax and totals
     - add confidence scores
     - avoid calculations
     - return only JSON
     - use `null` for unclear fields
   - Enable output parser
   - Connect **Merge OCR Results** to it

10. **Add an OpenAI Chat Model node** named **OpenAI GPT-4**.
    - Model: `gpt-4o`
    - Configure OpenAI credentials
    - Connect it to **AI Invoice Extractor** as the AI language model

11. **Add a Structured Output Parser node** named **Invoice Schema Parser**.
    - Enable `autoFix`
    - Choose manual schema
    - Define a JSON schema with:
      - `invoiceNumber`
      - `invoiceDate`
      - `dueDate`
      - `vendorName`
      - `vendorGST`
      - `vendorPAN`
      - `vendorBankAccount`
      - `subtotal`
      - `taxAmount`
      - `taxRate`
      - `total`
      - `lineItems[]` with `description`, `quantity`, `rate`, `amount`
      - `confidence` object with numeric confidence fields
    - Connect this parser to **AI Invoice Extractor**

12. **Add a second OpenAI Chat Model node** named **OpenAI GPT-4 for Parser**.
    - Model: `gpt-4o`
    - Connect it as the parser model for **Invoice Schema Parser**

13. **Add a Code node** named **Validate and Recalculate Totals**.
    - Connect **AI Invoice Extractor** to it
    - Paste logic that:
      - reads invoice JSON
      - reads `validationTolerance` from **Workflow Configuration**
      - recalculates each line amount as `quantity * rate`
      - sums a calculated subtotal
      - computes tax from `taxRate`
      - computes total
      - compares these values within tolerance
      - outputs a `validation` object with match flags and `validationPassed`

14. **Add a Code node** named **Mask PII Data**.
    - Connect **Validate and Recalculate Totals** to it
    - Paste logic that:
      - masks PAN except last 4 digits
      - masks bank account except last 4 digits
      - masks GST except last 6 digits
      - stores originals in `_originalPII`
      - sets `_piiMasked` to `true`

15. **Add an IF node** named **Check if Review Needed**.
    - Connect **Mask PII Data** to it
    - Add one boolean expression:
      - `{{ !$json.validation?.validationPassed || Object.values($json.confidence || {}).some(c => c < $('Workflow Configuration').first().json.confidenceThreshold) }}`
    - True branch means review required
    - False branch means auto-process

16. **Add a Slack node** named **Send Review Request**.
    - Connect the true branch from **Check if Review Needed**
    - Action: send message to channel
    - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
    - Configure Slack credentials
    - Message should include:
      - invoice number
      - vendor
      - total
      - whether issue is validation failure or low confidence
      - validation detail flags

17. **Add a Wait node** named **Wait for Human Review**.
    - Connect **Send Review Request** to it
    - Resume mode: webhook
    - Response mode: last node
    - Save the workflow once so the webhook can be generated
    - Build an external approval mechanism that calls the resume webhook when approved

18. **Add a QuickBooks node** named **Create QuickBooks Bill**.
    - Connect the false branch of **Check if Review Needed** to it
    - Also connect **Wait for Human Review** to it
    - Resource: Bill
    - Operation: Create
    - Vendor reference from `{{ $json.vendorName }}`
    - Map at least one line item:
      - amount = `{{ $json.lineItems[0].amount }}`
      - item ID = `{{ $json.lineItems[0].itemId }}`
      - description = `{{ $json.lineItems[0].description }}`
    - Additional fields:
      - DueDate = `{{ $json.dueDate }}`
      - TxnDate = `{{ $json.invoiceDate }}`
      - TotalAmt = `{{ $json.totalAmount }}`
    - Configure QuickBooks OAuth credentials

19. **Important correction before production use:** adjust the QuickBooks node design.
    - The provided workflow references `totalAmount`, but the extracted schema provides `total`
    - Either:
      - change `TotalAmt` to `{{ $json.total }}`, or
      - create a field named `totalAmount` before this node
    - The workflow also expects `lineItems[0].itemId`, but no previous node creates it
    - Add logic to map invoice lines to valid QuickBooks item IDs, or use an expense-account-based bill structure instead
    - If you need all invoice lines, configure the QuickBooks node to send all line items, not just the first one

20. **Add a Set node** named **Prepare Audit Log Entry**.
    - Connect **Create QuickBooks Bill** to it
    - Keep other fields
    - Add:
      - `auditTimestamp` = `{{ $now.toISO() }}`
      - `quickbooksId` = `{{ $json.id }}`
      - `processingStatus` = `completed`
      - `humanReviewed` = `{{ $('Wait for Human Review').first() ? true : false }}`
      - `validationResults` = `{{ JSON.stringify($json.validation) }}`
      - `confidenceScores` = `{{ JSON.stringify($json.confidence) }}`
      - `maskedData` = `true`

21. **Add a Google Sheets node** named **Log to Audit Trail**.
    - Connect **Prepare Audit Log Entry** to it
    - Operation: Append or Update
    - Spreadsheet ID: `{{ $('Workflow Configuration').first().json.auditSheetId }}`
    - Sheet name: `Audit Log`
    - Matching column: `invoiceNumber`
    - Map these columns:
      - `invoiceNumber`
      - `vendorName`
      - `total`
      - `auditTimestamp`
      - `quickbooksId`
      - `processingStatus`
      - `humanReviewed`
      - `validationResults`
      - `confidenceScores`
      - `fileHash`
    - Configure Google Sheets credentials

22. **Prepare the target Google Sheet**.
    - Create a spreadsheet
    - Add a worksheet named **Audit Log**
    - Create the required columns exactly matching the mapped field names

23. **Add a Slack node** named **Send Success Notification**.
    - Connect **Log to Audit Trail** to it
    - Send a message to the same configured channel
    - Include:
      - invoice number
      - vendor name
      - total
      - QuickBooks ID
      - whether human reviewed

24. **Optionally add sticky notes** to match the original visual layout.
   - Email Trigger
   - Configuration
   - File Processing
   - AI Extraction
   - Validation and Masking
   - Review Check
   - Human Review
   - Accounting Sync
   - Audit Logging
   - Notifications
   - General setup/how-it-works note

25. **Test with multiple sample invoices**.
   - PDF with embedded text
   - Scanned PDF
   - Image invoice
   - Low-confidence invoice
   - Invoice with mismatched totals
   - Multi-line invoice

26. **Before enabling in production, address these design gaps explicitly**:
   - Handle multiple attachments, not only `attachment_0`
   - Add duplicate-invoice detection using `fileHash`
   - Add explicit approval/rejection payload handling for the Wait webhook
   - Remove or securely encrypt `_originalPII`
   - Add error branches for OCR, OpenAI, QuickBooks, Slack, and Sheets
   - Normalize vendor mapping for QuickBooks vendor IDs
   - Support all line items instead of only the first
   - Confirm Merge node behavior when only one extraction branch runs

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Monitors email for invoice attachments, extracts data using OCR and AI, validates totals, masks sensitive fields, and decides whether to auto-process or send for review. | General workflow behavior |
| Connect Gmail, OpenAI, OCR API, Slack, QuickBooks, and Google Sheets. Configure thresholds, audit sheet, and test with sample invoices before enabling. | Setup guidance |
| The workflow contains placeholder values for OCR endpoint, Slack channel, and Google Sheets document ID. | Must be replaced before use |
| The current QuickBooks mapping is incomplete for production because it references `lineItems[0].itemId` and `totalAmount`, which are not guaranteed by earlier nodes. | Implementation warning |
| Human review is only a pause-and-resume mechanism in the current design; no explicit approve/reject data handling is implemented. | Operational warning |
| `_originalPII` is stored in clear form inside the payload despite the comment suggesting encryption in production. | Security/compliance warning |