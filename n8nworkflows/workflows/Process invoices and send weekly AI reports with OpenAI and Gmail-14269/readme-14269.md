Process invoices and send weekly AI reports with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/process-invoices-and-send-weekly-ai-reports-with-openai-and-gmail-14269


# Process invoices and send weekly AI reports with OpenAI and Gmail

# 1. Workflow Overview

This workflow automates two related processes:

1. **Invoice intake and validation**
   - A user uploads an invoice through an n8n form.
   - The workflow extracts text from the uploaded file, uses OpenAI to structure the invoice content, validates core fields, stores successful submissions, and sends an email back to the submitter.

2. **Weekly AI reporting**
   - On a weekly schedule, the workflow retrieves stored validated invoices, uses OpenAI to generate an HTML spending report, and emails that report to a configured recipient.

The workflow is organized into the following logical blocks.

## 1.1 Form Intake
Receives an email address and an invoice file from the user through a form trigger.

## 1.2 Configuration and Raw Submission Storage
Adds reusable configuration values such as allowed currencies and weekly report recipient, then stores the raw submission in a Data Table for audit or traceability.

## 1.3 File Content Extraction
Attempts to extract text from the uploaded invoice file.

## 1.4 AI Invoice Structuring
Uses a chat model plus a structured output parser to convert raw invoice text into machine-readable invoice fields.

## 1.5 Validation and Routing
Validates invoice fields with custom JavaScript logic, then routes the flow toward a success or failure branch.

## 1.6 Success Branch
Prepares a confirmation email, stores the validated invoice, and sends a success message.

## 1.7 Error Branch
Prepares and sends a validation failure email.

## 1.8 Weekly Reporting
Runs every Monday at 08:00, fetches validated invoices, generates an AI-written HTML report, and emails it.

---

# 2. Block-by-Block Analysis

## 2.1 Form Intake

**Overview:**  
This block is the primary entry point for invoice processing. It exposes a form where a user submits their email address and one invoice file.

**Nodes Involved:**  
- Invoice Upload Form

### Node Details

#### Invoice Upload Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that creates a hosted form and starts the workflow when submitted.
- **Configuration choices:**
  - Form title: **Invoice Upload - AI Processing System**
  - Description asks the user to upload a PDF or image invoice.
  - Fields:
    - Required email field: **Your Email Address**
    - Required file field: **Invoice File (PDF or Image)**
  - Accepted file extensions: `.pdf, .jpg, .jpeg, .png`
  - Single file only
  - Submission response text customized to confirm processing
  - Attribution disabled
- **Key expressions or variables used:**  
  None directly in this node.
- **Input and output connections:**
  - **Input:** none, this is a trigger
  - **Output:** Workflow Configuration
- **Version-specific requirements:**  
  Uses `typeVersion 2.3`; available options depend on recent n8n versions with Form Trigger support.
- **Edge cases or potential failure types:**
  - Users may upload non-PDF images even though downstream extraction is configured only for PDF extraction.
  - File binary property naming must match downstream assumptions.
  - Form submission may succeed while later nodes fail due to extraction or AI issues.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Configuration and Raw Submission Storage

**Overview:**  
This block enriches the submitted item with workflow-level configuration values and stores the original form payload in a Data Table for auditability.

**Nodes Involved:**  
- Workflow Configuration
- Store Raw Form Submission

### Node Details

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds configurable fields to the current item.
- **Configuration choices:**
  - Adds `reportRecipientEmail` as a placeholder string to be replaced by the operator
  - Adds `validCurrencies` as the string `USD,EUR,GBP,CAD,AUD,JPY,CHF`
  - `includeOtherFields` is enabled, so original form fields remain available
- **Key expressions or variables used:**  
  Static values only.
- **Input and output connections:**
  - **Input:** Invoice Upload Form
  - **Output:** Store Raw Form Submission
- **Version-specific requirements:**  
  `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - `validCurrencies` is stored as a string, but the validation code later treats it like an array and calls `.includes()` and `.join()`. `.includes()` works on strings, but `.join()` does not. This is a design issue.
  - `reportRecipientEmail` is defined here but not referenced correctly in the weekly email node.
- **Sub-workflow reference:**  
  None.

#### Store Raw Form Submission
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Writes incoming form data into a Data Table.
- **Configuration choices:**
  - Data Table ID: `invoice_form_submissions`
  - Column mapping mode: auto-map input data
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - **Input:** Workflow Configuration
  - **Output:** Extract Invoice Content
- **Version-specific requirements:**  
  `typeVersion 1`; requires n8n Data Tables support.
- **Edge cases or potential failure types:**
  - The table must already exist and be accessible.
  - Binary data handling depends on Data Table support and schema expectations.
  - Auto-mapping may create inconsistent field storage if incoming shapes change.
- **Sub-workflow reference:**  
  None.

---

## 2.3 File Content Extraction

**Overview:**  
This block converts the uploaded invoice file into text that the AI model can process.

**Nodes Involved:**  
- Extract Invoice Content

### Node Details

#### Extract Invoice Content
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts textual content from binary files.
- **Configuration choices:**
  - Operation: `pdf`
  - Binary property: `invoiceFile`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - **Input:** Store Raw Form Submission
  - **Output:** Extract Invoice Data with AI
- **Version-specific requirements:**  
  `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - This node is explicitly configured for **PDF extraction only**.
  - The form accepts JPG/JPEG/PNG, but this node will likely fail or produce no useful output for images unless OCR-compatible extraction is added separately.
  - If the uploaded binary field is not named `invoiceFile`, extraction will fail.
  - Scanned PDFs with no embedded text may return empty or poor text without OCR.
- **Sub-workflow reference:**  
  None.

---

## 2.4 AI Invoice Structuring

**Overview:**  
This block sends extracted invoice text to an OpenAI chat model and forces the response into a structured JSON schema suitable for downstream validation.

**Nodes Involved:**  
- Extract Invoice Data with AI
- Invoice Data Schema Parser
- OpenAI GPT-5-mini Model

### Node Details

#### Extract Invoice Data with AI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent node that uses a chat model and a structured output parser to extract invoice fields.
- **Configuration choices:**
  - Input text: `{{ $json.text }}`
  - Prompt type: defined manually
  - Output parser enabled
  - System instruction asks for:
    - invoice number
    - invoice date
    - vendor name
    - currency
    - total amount
    - subtotal
    - tax amount
    - line items
  - Missing fields should be returned as `null`
- **Key expressions or variables used:**
  - `{{ $json.text }}`
- **Input and output connections:**
  - **Main input:** Extract Invoice Content
  - **AI language model input:** OpenAI GPT-5-mini Model
  - **AI output parser input:** Invoice Data Schema Parser
  - **Main output:** Validate Invoice Data
- **Version-specific requirements:**  
  `typeVersion 3`; requires LangChain-compatible AI nodes in n8n.
- **Edge cases or potential failure types:**
  - Empty extracted text will reduce output quality or cause hallucinated fields.
  - The model may still produce schema-compliant but semantically inaccurate data.
  - Large invoices may exceed token limits depending on full extracted text size.
  - OpenAI credential or model availability issues can stop execution.
- **Sub-workflow reference:**  
  None.

#### Invoice Data Schema Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a manual JSON schema on model output.
- **Configuration choices:**
  - Manual schema with these properties:
    - `invoiceNumber` string
    - `invoiceDate` string
    - `vendorName` string
    - `currency` string
    - `totalAmount` number
    - `lineItems` array of objects
    - `taxAmount` number
    - `subtotal` number
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - **Output parser connection to:** Extract Invoice Data with AI
- **Version-specific requirements:**  
  `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - Schema does not require fields, so nulls or omissions may still pass parser logic depending on model output.
  - The parser expects field names like `invoiceDate`, but the downstream code checks `date`, creating a mismatch.
- **Sub-workflow reference:**  
  None.

#### OpenAI GPT-5-mini Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model used by the invoice extraction agent.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - No additional options or built-in tools configured
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - **AI language model connection to:** Extract Invoice Data with AI
- **Version-specific requirements:**  
  `typeVersion 1.3`; requires valid OpenAI credentials and model access.
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model unavailable in account/region
  - Rate limits or timeout
- **Sub-workflow reference:**  
  None.

---

## 2.5 Validation and Routing

**Overview:**  
This block runs custom checks on the extracted invoice fields and branches the flow based on whether validation succeeds.

**Nodes Involved:**  
- Validate Invoice Data
- Check Validation Result

### Node Details

#### Validate Invoice Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes custom JavaScript for invoice validation.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Builds a `validationResult` object:
    - `isValid`
    - `errors`
  - Checks:
    - date format must match `YYYY-MM-DD`
    - currency must be in configured allowed list
    - total amount must be greater than zero
  - Returns original data plus `validation`
- **Key expressions or variables used:**
  - `$('Workflow Configuration').item.json.validCurrencies`
  - `$input.item.json`
- **Input and output connections:**
  - **Input:** Extract Invoice Data with AI
  - **Output:** Check Validation Result
- **Version-specific requirements:**  
  `typeVersion 2`
- **Edge cases or potential failure types:**
  - **Field mismatch bug:** code checks `invoiceData.date`, but the schema produces `invoiceDate`. This will mark valid invoices as invalid.
  - **Currency config type mismatch:** `validCurrencies` is a comma-separated string, not an array. `.includes()` may produce accidental substring matches; `.join()` will throw because strings do not have `.join()`.
  - **Output mismatch bug:** returns `validation: validationResult`, but downstream IF node checks `isValid` at the root level instead of `validation.isValid`.
  - Error branch email expects `validationErrors`, but validation code stores errors under `validation.errors`.
- **Sub-workflow reference:**  
  None.

#### Check Validation Result
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes items based on whether invoice validation passed.
- **Configuration choices:**
  - Boolean equals check against:
    - `{{ $('Validate Invoice Data').item.json.isValid }}`
  - True branch goes to success
  - False branch goes to error
- **Key expressions or variables used:**
  - `{{ $('Validate Invoice Data').item.json.isValid }}`
- **Input and output connections:**
  - **Input:** Validate Invoice Data
  - **True output:** Prepare Success Email Data
  - **False output:** Prepare Error Email Data
- **Version-specific requirements:**  
  `typeVersion 2.3`
- **Edge cases or potential failure types:**
  - **Critical logic bug:** the code node returns `validation.isValid`, not root-level `isValid`. This IF condition will likely evaluate as undefined and route incorrectly.
  - If the code node throws due to `.join()` on a string, this IF node is never reached.
- **Sub-workflow reference:**  
  None.

---

## 2.6 Success Branch

**Overview:**  
If validation passes, this block prepares an HTML confirmation email, stores the validated invoice, and sends the message.

**Nodes Involved:**  
- Prepare Success Email Data
- Store Validated Invoice
- Send Success Email

### Node Details

#### Prepare Success Email Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates email subject and HTML body for successful processing.
- **Configuration choices:**
  - `emailSubject`: `Invoice Processed Successfully`
  - `emailBody`: HTML template including:
    - invoice number
    - vendor
    - date
    - total amount
  - Keeps other fields
- **Key expressions or variables used:**
  - `{{ $json.invoiceNumber }}`
  - `{{ $json.vendorName }}`
  - `{{ $json.invoiceDate }}`
  - `{{ $json.totalAmount }}`
- **Input and output connections:**
  - **Input:** Check Validation Result (true branch)
  - **Output:** Store Validated Invoice
- **Version-specific requirements:**  
  `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - If upstream field names differ or are null, placeholders render empty values.
  - Currency symbol is hardcoded as `$`, even when invoice currency may be EUR, GBP, etc.
- **Sub-workflow reference:**  
  None.

#### Store Validated Invoice
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Stores validated invoice records for later analysis and reporting.
- **Configuration choices:**
  - Data Table name: `validated_invoices`
  - Auto-map input data
  - No type conversion forced
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - **Input:** Prepare Success Email Data
  - **Output:** Send Success Email
- **Version-specific requirements:**  
  `typeVersion 1`
- **Edge cases or potential failure types:**
  - Table must exist.
  - Auto-mapping may store extra presentation fields like `emailSubject` and `emailBody`, not only business invoice data.
- **Sub-workflow reference:**  
  None.

#### Send Success Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the success notification by Gmail.
- **Configuration choices:**
  - Subject: `{{ $json.emailSubject }}`
  - Body: `{{ $json.emailBody }}`
  - Recipient: `{{ $('Workflow Configuration').first().json.userEmail }}`
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.userEmail }}`
  - `{{ $json.emailSubject }}`
  - `{{ $json.emailBody }}`
- **Input and output connections:**
  - **Input:** Store Validated Invoice
  - **Output:** none
- **Version-specific requirements:**  
  `typeVersion 2.2`; requires Gmail credentials.
- **Edge cases or potential failure types:**
  - **Recipient bug:** the form field is likely not stored as `userEmail`; the label is “Your Email Address”. Depending on n8n’s generated field key, this expression may fail.
  - HTML body rendering depends on Gmail node behavior and options.
  - OAuth authorization can expire.
- **Sub-workflow reference:**  
  None.

---

## 2.7 Error Branch

**Overview:**  
If validation fails, this block creates an HTML email listing the problems and sends it back to the user.

**Nodes Involved:**  
- Prepare Error Email Data
- Send Error Email

### Node Details

#### Prepare Error Email Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds an HTML error response.
- **Configuration choices:**
  - `emailSubject`: `Invoice Validation Failed`
  - `emailBody`: HTML template with:
    - validation errors list
    - submission time
    - file name
    - resubmission prompt
- **Key expressions or variables used:**
  - `{{ $json.validationErrors ? $json.validationErrors.map(...) : ... }}`
  - `{{ $json.submissionTime || 'N/A' }}`
  - `{{ $json.fileName || 'N/A' }}`
- **Input and output connections:**
  - **Input:** Check Validation Result (false branch)
  - **Output:** Send Error Email
- **Version-specific requirements:**  
  `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - **Field mismatch bug:** validation errors are stored upstream in `validation.errors`, not `validationErrors`.
  - `submissionTime` and `fileName` are not explicitly created upstream, so these likely render as `N/A`.
- **Sub-workflow reference:**  
  None.

#### Send Error Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the validation failure notice by Gmail.
- **Configuration choices:**
  - Subject: `{{ $json.emailSubject }}`
  - Body: `{{ $json.emailBody }}`
  - Recipient: `{{ $('Workflow Configuration').first().json.userEmail }}`
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.userEmail }}`
  - `{{ $json.emailSubject }}`
  - `{{ $json.emailBody }}`
- **Input and output connections:**
  - **Input:** Prepare Error Email Data
  - **Output:** none
- **Version-specific requirements:**  
  `typeVersion 2.2`; requires Gmail credentials.
- **Edge cases or potential failure types:**
  - Same likely recipient field mismatch as the success email node.
  - OAuth or Gmail send quota issues may block delivery.
- **Sub-workflow reference:**  
  None.

---

## 2.8 Weekly Reporting

**Overview:**  
This block is the second entry point. It runs on a weekly schedule, retrieves stored invoices, asks OpenAI to produce an HTML financial summary, and emails the result.

**Nodes Involved:**  
- Weekly Report Schedule
- Fetch Weekly Invoices
- Generate Weekly Report
- OpenAI GPT-5-mini Model for Reports
- Send Weekly Report Email

### Node Details

#### Weekly Report Schedule
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger for periodic reporting.
- **Configuration choices:**
  - Cron expression: `0 8 * * 1`
  - Runs every Monday at 08:00
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - **Input:** none, this is a trigger
  - **Output:** Fetch Weekly Invoices
- **Version-specific requirements:**  
  `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - Timezone depends on workflow/n8n instance settings.
  - It fetches all invoices, not only the previous week’s data.
- **Sub-workflow reference:**  
  None.

#### Fetch Weekly Invoices
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Reads invoice records from the validated invoices Data Table.
- **Configuration choices:**
  - Operation: `get`
  - Return all: true
  - Data Table ID: `validated_invoices`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - **Input:** Weekly Report Schedule
  - **Output:** Generate Weekly Report
- **Version-specific requirements:**  
  `typeVersion 1`
- **Edge cases or potential failure types:**
  - It does not filter by date range, so the report is cumulative over all validated invoices.
  - Empty table may still produce a report request with no meaningful data.
- **Sub-workflow reference:**  
  None.

#### Generate Weekly Report
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Uses AI to generate a formatted weekly invoice report in HTML.
- **Configuration choices:**
  - Input text: `{{ JSON.stringify($input.all()) }}`
  - System message requests:
    1. executive summary
    2. spending by vendor
    3. spending by currency
    4. top 5 largest invoices
    5. trends and insights
  - Output should be professional HTML
- **Key expressions or variables used:**
  - `{{ JSON.stringify($input.all()) }}`
- **Input and output connections:**
  - **Main input:** Fetch Weekly Invoices
  - **AI language model input:** OpenAI GPT-5-mini Model for Reports
  - **Main output:** Send Weekly Report Email
- **Version-specific requirements:**  
  `typeVersion 3`
- **Edge cases or potential failure types:**
  - Large historical invoice datasets may exceed token context.
  - Since no parser is attached, output structure is freeform HTML and may vary.
  - Because invoices are not date-filtered, “weekly” report semantics are misleading.
- **Sub-workflow reference:**  
  None.

#### OpenAI GPT-5-mini Model for Reports
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Language model backing the report generation agent.
- **Configuration choices:**
  - Model: `gpt-5-mini`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - **AI language model connection to:** Generate Weekly Report
- **Version-specific requirements:**  
  `typeVersion 1.3`; requires OpenAI credentials and access to the selected model.
- **Edge cases or potential failure types:**
  - Same OpenAI auth, quota, and availability risks as the extraction model node.
- **Sub-workflow reference:**  
  None.

#### Send Weekly Report Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated report by email.
- **Configuration choices:**
  - Recipient expression:
    `{{ $('Weekly Report Schedule').first().json.reportRecipientEmail || "<__PLACEHOLDER_VALUE__Weekly report recipient email__>" }}`
  - Message: `{{ $json.output }}`
  - Subject: `Weekly Invoice Processing Report - {{ $now.format("MMMM DD, YYYY") }}`
- **Key expressions or variables used:**
  - `{{ $('Weekly Report Schedule').first().json.reportRecipientEmail || ... }}`
  - `{{ $json.output }}`
  - `{{ $now.format("MMMM DD, YYYY") }}`
- **Input and output connections:**
  - **Input:** Generate Weekly Report
  - **Output:** none
- **Version-specific requirements:**  
  `typeVersion 2.2`; requires Gmail credentials.
- **Edge cases or potential failure types:**
  - **Recipient bug:** `Weekly Report Schedule` does not provide `reportRecipientEmail`. That field is set only in the invoice-processing branch’s Set node and is unavailable here.
  - As configured, the node may always fall back to the placeholder unless manually corrected.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Invoice Upload Form | n8n-nodes-base.formTrigger | Accept user email and invoice file |  | Workflow Configuration | ## Form Input<br>Collects user email and invoice file via form trigger. |
| Workflow Configuration | n8n-nodes-base.set | Add config values for currencies and report recipient | Invoice Upload Form | Store Raw Form Submission | ## Configuration and Submission Storage<br>Defines report email and valid currencies used in validation. and<br>Stores raw form data for tracking and auditing purposes. |
| Store Raw Form Submission | n8n-nodes-base.dataTable | Store original submitted data | Workflow Configuration | Extract Invoice Content | ## Configuration and Submission Storage<br>Defines report email and valid currencies used in validation. and<br>Stores raw form data for tracking and auditing purposes. |
| Extract Invoice Content | n8n-nodes-base.extractFromFile | Extract text from uploaded invoice file | Store Raw Form Submission | Extract Invoice Data with AI | ## File Extraction<br>Extracts text content from uploaded invoice files. |
| Extract Invoice Data with AI | @n8n/n8n-nodes-langchain.agent | Convert invoice text into structured fields | Extract Invoice Content | Validate Invoice Data | ## AI Processing<br>AI converts extracted text into structured invoice data. |
| Invoice Data Schema Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON schema for extracted invoice data |  | Extract Invoice Data with AI | ## AI Processing<br>AI converts extracted text into structured invoice data. |
| OpenAI GPT-5-mini Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model for invoice extraction |  | Extract Invoice Data with AI | ## AI Processing<br>AI converts extracted text into structured invoice data. |
| Validate Invoice Data | n8n-nodes-base.code | Validate extracted invoice fields | Extract Invoice Data with AI | Check Validation Result | ## Validation<br>Checks date format, currency, and total amount accuracy. |
| Check Validation Result | n8n-nodes-base.if | Route item to success or error branch | Validate Invoice Data | Prepare Success Email Data; Prepare Error Email Data | ## Validation<br>Checks date format, currency, and total amount accuracy. |
| Prepare Success Email Data | n8n-nodes-base.set | Build success email subject and HTML body | Check Validation Result | Store Validated Invoice | ## Success Path<br>Stores valid invoices and sends confirmation email. |
| Store Validated Invoice | n8n-nodes-base.dataTable | Save validated invoice data | Prepare Success Email Data | Send Success Email | ## Success Path<br>Stores valid invoices and sends confirmation email. |
| Send Success Email | n8n-nodes-base.gmail | Send confirmation email | Store Validated Invoice |  | ## Success Path<br>Stores valid invoices and sends confirmation email. |
| Prepare Error Email Data | n8n-nodes-base.set | Build validation error email | Check Validation Result | Send Error Email | ## Error Path<br>Sends email with validation errors for correction. |
| Send Error Email | n8n-nodes-base.gmail | Send validation failure email | Prepare Error Email Data |  | ## Error Path<br>Sends email with validation errors for correction. |
| Weekly Report Schedule | n8n-nodes-base.scheduleTrigger | Weekly trigger for reporting |  | Fetch Weekly Invoices | ## Weekly Trigger<br>Runs weekly to process stored invoices for reporting. |
| Fetch Weekly Invoices | n8n-nodes-base.dataTable | Retrieve validated invoices | Weekly Report Schedule | Generate Weekly Report | ## Weekly Trigger<br>Runs weekly to process stored invoices for reporting. |
| Generate Weekly Report | @n8n/n8n-nodes-langchain.agent | Create HTML spending report with AI | Fetch Weekly Invoices | Send Weekly Report Email | ## Report Generation<br>AI generates a formatted weekly invoice summary report. |
| OpenAI GPT-5-mini Model for Reports | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model for report generation |  | Generate Weekly Report | ## Report Generation<br>AI generates a formatted weekly invoice summary report. |
| Send Weekly Report Email | n8n-nodes-base.gmail | Email the weekly HTML report | Generate Weekly Report |  | ## Report Email<br>Sends weekly report to configured recipient. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## How it works<br>This workflow helps you automatically process invoices using AI.<br><br>A user uploads an invoice file (PDF or image) through a form. The file is read and the text is extracted. Then AI pulls out important details like invoice number, vendor name, date, currency, and total amount.<br><br>After that, the data is checked to make sure it looks correct. For example, it verifies the date format, allowed currencies, and that the total amount is valid.<br><br>If everything is correct, the invoice is saved and a confirmation email is sent to the user. If something is wrong, the user receives an email with the issues so they can fix and resend the invoice.<br><br>The workflow also runs weekly. It collects all processed invoices and creates a simple summary report using AI, which is sent by email.<br><br>## Setup steps<br>1. Add your OpenAI credentials<br>2. Connect Gmail for sending emails<br>3. Set up Data Tables for storing invoices<br>4. Add your report email in the config node<br>5. Test with a sample invoice<br>6. Turn on the workflow |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## Form Input<br>Collects user email and invoice file via form trigger. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## Configuration and Submission Storage<br>Defines report email and valid currencies used in validation. and<br>Stores raw form data for tracking and auditing purposes. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## File Extraction<br>Extracts text content from uploaded invoice files. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## AI Processing<br>AI converts extracted text into structured invoice data. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## Validation<br>Checks date format, currency, and total amount accuracy. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## Success Path<br>Stores valid invoices and sends confirmation email. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## Error Path<br>Sends email with validation errors for correction. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## Weekly Trigger<br>Runs weekly to process stored invoices for reporting. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## Report Generation<br>AI generates a formatted weekly invoice summary report. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation canvas note |  |  | ## Report Email<br>Sends weekly report to configured recipient. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence. It includes the workflow as implemented, plus important corrections you should apply if you want it to work reliably.

## 4.1 Prerequisites
1. Create or prepare:
   - **OpenAI credentials** in n8n
   - **Gmail credentials** in n8n
2. Ensure **Data Tables** are available in your n8n instance.
3. Create two Data Tables:
   - `invoice_form_submissions`
   - `validated_invoices`

## 4.2 Build the invoice intake branch

1. **Create a Form Trigger node**
   - Name: `Invoice Upload Form`
   - Type: `Form Trigger`
   - Set:
     - Form title: `Invoice Upload - AI Processing System`
     - Description: `Upload your invoice (PDF or image) for automated processing and validation`
   - Add two fields:
     - Email field, required, label `Your Email Address`
     - File field, required, label `Invoice File (PDF or Image)`
   - Allow one file only
   - Accepted file types: `.pdf, .jpg, .jpeg, .png`
   - In response options, set submitted text to:
     - `Thank you! Your invoice is being processed. You will receive an email confirmation shortly.`

2. **Create a Set node**
   - Name: `Workflow Configuration`
   - Add fields:
     - `reportRecipientEmail` = your weekly report recipient email
     - `validCurrencies` = preferably an **array** such as `["USD","EUR","GBP","CAD","AUD","JPY","CHF"]`
   - Keep input fields enabled.

   **Important:** the original workflow stores currencies as a comma-separated string. Use an array instead to avoid code errors.

3. **Create a Data Table node**
   - Name: `Store Raw Form Submission`
   - Configure it to insert/store records into `invoice_form_submissions`
   - Use auto-mapping

4. **Create an Extract From File node**
   - Name: `Extract Invoice Content`
   - Set operation to `PDF`
   - Binary property: use the actual binary field name created by the form file upload
   - If your form node uses a different binary property than `invoiceFile`, update this setting accordingly

   **Important:** the original workflow accepts image files but only uses PDF extraction.  
   If you need real image support:
   - add OCR/image extraction logic, or
   - restrict the form to PDFs only.

5. **Create an OpenAI Chat Model node**
   - Name: `OpenAI GPT-5-mini Model`
   - Type: OpenAI Chat Model
   - Select model `gpt-5-mini`
   - Attach your OpenAI credentials

6. **Create a Structured Output Parser node**
   - Name: `Invoice Data Schema Parser`
   - Use manual schema
   - Define fields:
     - `invoiceNumber` string
     - `invoiceDate` string
     - `vendorName` string
     - `currency` string
     - `totalAmount` number
     - `subtotal` number
     - `taxAmount` number
     - `lineItems` array of objects with:
       - `description` string
       - `quantity` number
       - `unitPrice` number
       - `amount` number

7. **Create an AI Agent node**
   - Name: `Extract Invoice Data with AI`
   - Set input text to:
     - `{{ $json.text }}`
   - Add a system message instructing the model to extract:
     - invoice number
     - invoice date in `YYYY-MM-DD`
     - vendor/supplier name
     - currency
     - total amount
     - subtotal
     - tax amount
     - line items
     - null for missing fields
   - Enable structured output parser
   - Connect:
     - OpenAI model node to the agent’s language model input
     - Schema parser to the agent’s output parser input

8. **Create a Code node**
   - Name: `Validate Invoice Data`
   - Mode: `Run Once for Each Item`
   - Add validation logic.

   To make it work properly, use logic conceptually equivalent to:
   - read `invoiceDate`, not `date`
   - read `validation.isValid` consistently downstream
   - expect `validCurrencies` as an array
   - return fields like:
     - `validation.isValid`
     - `validation.errors`

   Also consider adding:
   - check for missing invoice number
   - check subtotal + tax ≈ total
   - check line items sum

9. **Create an IF node**
   - Name: `Check Validation Result`
   - Condition:
     - `{{ $json.validation.isValid }}`
     - equals `true`

   **Important:** this corrects the original mismatch.

## 4.3 Build the success path

10. **Create a Set node**
    - Name: `Prepare Success Email Data`
    - Add:
      - `emailSubject` = `Invoice Processed Successfully`
      - `emailBody` = HTML body with invoice summary
    - Use expressions for:
      - invoice number
      - vendor
      - invoice date
      - total amount

11. **Create a Data Table node**
    - Name: `Store Validated Invoice`
    - Insert into `validated_invoices`
    - Use auto-mapping

12. **Create a Gmail node**
    - Name: `Send Success Email`
    - Operation: send email
    - Attach Gmail credentials
    - Recipient: use the actual email field coming from the form submission
      - for example, `{{ $json["Your Email Address"] }}` or the exact generated field name in your build
    - Subject: `{{ $json.emailSubject }}`
    - Message/body: `{{ $json.emailBody }}`

    **Important:** the original expression uses `userEmail`, which is not clearly defined by the form node.

## 4.4 Build the error path

13. **Create a Set node**
    - Name: `Prepare Error Email Data`
    - Add:
      - `emailSubject` = `Invoice Validation Failed`
      - `emailBody` = HTML showing validation errors
    - Reference:
      - `{{ $json.validation.errors }}`
      - and render them as list items

14. **Create a Gmail node**
    - Name: `Send Error Email`
    - Use the same Gmail credentials
    - Recipient: the submitter email from the form
    - Subject: `{{ $json.emailSubject }}`
    - Message/body: `{{ $json.emailBody }}`

## 4.5 Build the weekly report branch

15. **Create a Schedule Trigger node**
    - Name: `Weekly Report Schedule`
    - Configure cron:
      - `0 8 * * 1`
    - This runs every Monday at 08:00

16. **Create a Data Table node**
    - Name: `Fetch Weekly Invoices`
    - Operation: `Get`
    - Table: `validated_invoices`
    - Return all: enabled

    **Recommended improvement:** filter invoices to the last 7 days instead of reading all rows.

17. **Create another OpenAI Chat Model node**
    - Name: `OpenAI GPT-5-mini Model for Reports`
    - Model: `gpt-5-mini`
    - Same or separate OpenAI credentials

18. **Create an AI Agent node**
    - Name: `Generate Weekly Report`
    - Input text:
      - `{{ JSON.stringify($input.all()) }}`
    - System message should ask for an HTML report containing:
      - executive summary
      - vendor breakdown
      - currency totals
      - top 5 largest invoices
      - trends and insights
    - Connect the report model node to this agent

19. **Create a Gmail node**
    - Name: `Send Weekly Report Email`
    - Recipient: directly configure a static email or use a config source available in this branch
    - Subject:
      - `Weekly Invoice Processing Report - {{ $now.format("MMMM DD, YYYY") }}`
    - Message/body:
      - `{{ $json.output }}`

    **Important:** the original workflow incorrectly tries to read `reportRecipientEmail` from the Schedule Trigger output.  
    Fix by either:
    - hardcoding the recipient here,
    - adding a Set node in the weekly branch before sending, or
    - reading from a shared config source accessible in this branch.

## 4.6 Connect the nodes

20. Connect the invoice-processing branch in this order:
   - `Invoice Upload Form` → `Workflow Configuration`
   - `Workflow Configuration` → `Store Raw Form Submission`
   - `Store Raw Form Submission` → `Extract Invoice Content`
   - `Extract Invoice Content` → `Extract Invoice Data with AI`
   - `OpenAI GPT-5-mini Model` → AI model input of `Extract Invoice Data with AI`
   - `Invoice Data Schema Parser` → parser input of `Extract Invoice Data with AI`
   - `Extract Invoice Data with AI` → `Validate Invoice Data`
   - `Validate Invoice Data` → `Check Validation Result`

21. Connect the success branch:
   - `Check Validation Result` true → `Prepare Success Email Data`
   - `Prepare Success Email Data` → `Store Validated Invoice`
   - `Store Validated Invoice` → `Send Success Email`

22. Connect the error branch:
   - `Check Validation Result` false → `Prepare Error Email Data`
   - `Prepare Error Email Data` → `Send Error Email`

23. Connect the weekly reporting branch:
   - `Weekly Report Schedule` → `Fetch Weekly Invoices`
   - `Fetch Weekly Invoices` → `Generate Weekly Report`
   - `OpenAI GPT-5-mini Model for Reports` → AI model input of `Generate Weekly Report`
   - `Generate Weekly Report` → `Send Weekly Report Email`

## 4.7 Optional canvas documentation

24. Add Sticky Notes matching these areas:
   - Form Input
   - Configuration and Submission Storage
   - File Extraction
   - AI Processing
   - Validation
   - Success Path
   - Error Path
   - Weekly Trigger
   - Report Generation
   - Report Email

25. Add one larger note with setup guidance:
   - Add OpenAI credentials
   - Connect Gmail
   - Set up Data Tables
   - Add report email in config
   - Test with sample invoice
   - Activate workflow

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow contains two entry points: a Form Trigger for invoice uploads and a Schedule Trigger for weekly reporting. | Architecture note |
| The current implementation contains several field-reference mismatches that should be corrected before production use. | Validation and email routing logic |
| The form accepts image files, but extraction is configured only for PDF text extraction. Add OCR support or limit uploads to PDFs. | File processing design |
| The weekly report currently reads all validated invoices, not only the last 7 days. Add date filtering for true weekly reporting. | Reporting logic |
| The workflow relies on OpenAI chat model nodes using `gpt-5-mini`. Ensure your account has access to this model. | OpenAI dependency |
| Gmail nodes require authenticated Gmail credentials with sending permission. | Email delivery dependency |
| Data Tables `invoice_form_submissions` and `validated_invoices` must exist before running the workflow. | Storage prerequisite |