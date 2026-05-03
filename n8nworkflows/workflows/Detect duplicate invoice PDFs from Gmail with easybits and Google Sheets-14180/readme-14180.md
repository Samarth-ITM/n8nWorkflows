Detect duplicate invoice PDFs from Gmail with easybits and Google Sheets

https://n8nworkflows.xyz/workflows/detect-duplicate-invoice-pdfs-from-gmail-with-easybits-and-google-sheets-14180


# Detect duplicate invoice PDFs from Gmail with easybits and Google Sheets

# 1. Workflow Overview

## Purpose
This workflow detects duplicate invoice PDFs received via Gmail. It extracts structured invoice data from attached PDF files using easybits, checks whether the invoice number already exists in a Google Sheets master register, and then either alerts Finance in Slack or appends the invoice to the sheet.

## Target use cases
- Accounts payable intake from a shared Gmail inbox
- Duplicate invoice prevention before payment processing
- Lightweight invoice registration without a full ERP integration
- Automated triage of incoming invoice PDFs

## Logical blocks

### 1.1 Email Intake
The workflow starts from Gmail and polls for incoming emails on a schedule. Attachments are downloaded so invoice PDFs can be processed immediately.

### 1.2 PDF Preparation and Extraction
The attachment is converted into a format suitable for the easybits extractor API. The API returns structured invoice metadata such as invoice number and total amount.

### 1.3 Duplicate Lookup Against Google Sheets
The workflow reads rows from the master finance spreadsheet and compares the extracted invoice number with existing sheet values using a JavaScript Code node.

### 1.4 Duplicate Decision and Notification
An IF node evaluates whether the invoice already exists. If it does, the workflow sends a Slack direct message alert.

### 1.5 New Invoice Registration
If the invoice is not found in the sheet, the workflow appends the invoice number and amount to the Google Sheets master list.

---

# 2. Block-by-Block Analysis

## 2.1 Email Intake

### Overview
This block receives new emails from Gmail and downloads their attachments. It is the only workflow entry point and supplies the binary invoice file used downstream.

### Nodes Involved
- Gmail Trigger

### Node Details

#### Gmail Trigger
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger node that checks Gmail for new matching emails.
- **Configuration choices:**
  - Uses polling every minute
  - Attachment download is enabled
  - `simple` mode is disabled, meaning fuller Gmail data is returned
  - Label filtering is present structurally, but in this JSON the `labelIds` array is empty
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: none, this is the workflow trigger
  - Output: `Extract from File`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires Gmail OAuth2 credentials in n8n
- **Edge cases or potential failure types:**
  - OAuth token expiration or revoked Gmail access
  - No attachments present on the email
  - Label filter not actually configured despite the sticky note instructions
  - Multiple attachments may be present, but the downstream node explicitly targets `attachment_0`
  - Polling delay means processing is not truly instant
- **Sub-workflow reference:** None

---

## 2.2 PDF Preparation and Extraction

### Overview
This block reads the first Gmail attachment, converts it into a base64 data URI, and submits it to the easybits extraction API. The API is expected to return structured invoice fields used later for duplicate detection and sheet insertion.

### Nodes Involved
- Extract from File
- Edit Fields
- HTTP Request

### Node Details

#### Extract from File
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Converts binary file content into a property in the JSON payload.
- **Configuration choices:**
  - Operation: binary to property
  - Reads binary property `attachment_0`
- **Key expressions or variables used:**
  - `=attachment_0`
- **Input and output connections:**
  - Input: `Gmail Trigger`
  - Output: `Edit Fields`
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - Fails if the email has no `attachment_0`
  - If the first attachment is not the invoice PDF, wrong file may be processed
  - If there are multiple PDF attachments, only the first is considered
- **Sub-workflow reference:** None

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a field containing a PDF data URI string from the extracted base64 content.
- **Configuration choices:**
  - Assigns a new string field named `data`
  - The value is prefixed with `data:application/pdf;base64,`
- **Key expressions or variables used:**
  - `=data:application/pdf;base64,{{ $json.data }}`
- **Input and output connections:**
  - Input: `Extract from File`
  - Output: `HTTP Request`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - Assumes `Extract from File` outputs base64 into `$json.data`
  - If the attachment is not actually a PDF, the MIME prefix becomes misleading
  - Large files may cause payload size issues downstream
- **Sub-workflow reference:** None

#### HTTP Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the PDF to easybits Extractor for structured document extraction.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID`
  - Body format: JSON
  - Body contains a `files` array with the base64 PDF data URI
  - Authentication uses predefined credential type with bearer token
- **Key expressions or variables used:**
  - JSON body:
    - `{{ $json.data }}`
- **Input and output connections:**
  - Input: `Edit Fields`
  - Output: `Check Google Sheets`
- **Version-specific requirements:**
  - Type version `4.3`
  - Requires an `httpBearerAuth` credential with the easybits API key
- **Edge cases or potential failure types:**
  - Placeholder pipeline ID must be replaced or the request will fail
  - 401/403 authentication errors if bearer token is invalid
  - 404 if pipeline ID is wrong
  - Timeout or service unavailability from easybits
  - Response schema must include `data.invoice_number` and `data.total_amount`, or downstream expressions will fail
- **Sub-workflow reference:** None

---

## 2.3 Duplicate Lookup Against Google Sheets

### Overview
This block reads the master sheet and compares every existing invoice number to the one extracted from easybits. The comparison logic is custom JavaScript and returns a compact duplicate status payload.

### Nodes Involved
- Check Google Sheets
- Code in JavaScript

### Node Details

#### Check Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the configured spreadsheet sheet.
- **Configuration choices:**
  - Spreadsheet selected by `documentId`
  - Sheet selected as `gid=0` / `Sheet1`
  - `alwaysOutputData` is enabled, so the workflow continues even if no rows are found
  - No explicit operation is shown in the JSON; in this context it functions as a row read/list step
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `HTTP Request`
  - Output: `Code in JavaScript`
- **Version-specific requirements:**
  - Type version `4`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Spreadsheet not selected in the exported JSON, so this must be configured manually
  - Missing permissions to the target spreadsheet
  - Sheet schema mismatch if `Invoice Number` column does not exist
  - Empty sheet is supported because `alwaysOutputData` is true
- **Sub-workflow reference:** None

#### Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`  
  Performs the actual duplicate detection by comparing the extracted invoice number against sheet rows.
- **Configuration choices:**
  - Reads the invoice number from the `HTTP Request` node output
  - Collects all incoming rows from Google Sheets
  - Filters rows whose `Invoice Number` exactly matches after string conversion and trimming
  - Returns one object with:
    - `duplicate`
    - `invoice_number`
    - `found_in_sheet`
    - `rows_checked`
- **Key expressions or variables used:**
  - `$('HTTP Request').first().json.data.invoice_number`
  - `row.json["Invoice Number"]`
- **Input and output connections:**
  - Input: `Check Google Sheets`
  - Output: `Already Exists?`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - If `HTTP Request` returns no `data.invoice_number`, matching logic becomes unreliable
  - Matching is exact except for trimming, so casing, formatting differences, prefixes, or OCR variation are not normalized
  - Duplicate entries in the sheet are not separately handled; any match counts as duplicate
  - Very large sheets may degrade performance because all rows are loaded and iterated
- **Sub-workflow reference:** None

---

## 2.4 Duplicate Decision and Notification

### Overview
This block checks the duplicate flag produced by the Code node. If the invoice already exists, a Slack message is sent for human review.

### Nodes Involved
- Already Exists?
- Slack: Alert Finance

### Node Details

#### Already Exists?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branching node that routes execution based on the duplicate flag.
- **Configuration choices:**
  - Compares `$json.duplicate` to the string `"true"`
  - True branch goes to Slack
  - False branch goes to Google Sheets append
- **Key expressions or variables used:**
  - `={{ $json.duplicate }}`
  - compared to `true` as string
- **Input and output connections:**
  - Input: `Code in JavaScript`
  - Output 0 (true): `Slack: Alert Finance`
  - Output 1 (false): `Add to Master List`
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases or potential failure types:**
  - Uses string values `"true"` / `"false"` rather than booleans
  - If the Code node output format changes, branching may fail silently
- **Sub-workflow reference:** None

#### Slack: Alert Finance
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack direct message alert when a duplicate invoice is detected.
- **Configuration choices:**
  - Sends text message containing the duplicate invoice number
  - `select` is set to `user`, implying a direct user-targeted message
  - User value is empty in the exported JSON and must be configured
- **Key expressions or variables used:**
  - `{{ $('HTTP Request').first().json.data.invoice_number }}`
- **Input and output connections:**
  - Input: `Already Exists?` true branch
  - Output: none
- **Version-specific requirements:**
  - Type version `2`
  - Requires Slack credential with sufficient scopes
- **Edge cases or potential failure types:**
  - Missing Slack user selection will prevent sending
  - Slack app may lack required scopes
  - If invoice number is missing from the extractor response, message content is incomplete
  - Direct messages may fail depending on workspace permissions/app installation
- **Sub-workflow reference:** None

---

## 2.5 New Invoice Registration

### Overview
This block appends new invoice data to the finance spreadsheet if the invoice is not a duplicate. It writes only the extracted invoice number and total amount to predefined columns.

### Nodes Involved
- Add to Master List

### Node Details

#### Add to Master List
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to the Google Sheets master register.
- **Configuration choices:**
  - Operation: `append`
  - Target sheet: `gid=0` / `Sheet1`
  - Explicit column mapping is enabled
  - Writes:
    - `Invoice Number` from extracted invoice number
    - `Final Amount (EUR)` from extracted total amount
  - Other schema columns exist but are not populated
- **Key expressions or variables used:**
  - `={{ $('HTTP Request').first().json.data.invoice_number }}`
  - `={{ $('HTTP Request').first().json.data.total_amount }}`
- **Input and output connections:**
  - Input: `Already Exists?` false branch
  - Output: none
- **Version-specific requirements:**
  - Type version `4`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Spreadsheet and sheet must be manually selected
  - Missing `Invoice Number` or `Final Amount (EUR)` columns causes write issues
  - Extracted amount may not actually be in EUR despite the target column name
  - Duplicate prevention depends entirely on prior logic; there is no write-time uniqueness constraint
- **Sub-workflow reference:** None

---

## 2.6 Documentation and In-Canvas Notes

### Overview
These nodes are non-executing sticky notes used to explain setup, behavior, and branch logic directly on the canvas. They do not affect runtime but are important for maintainability.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for email intake.
- **Configuration choices:** Describes Gmail polling every minute, invoice label, and attachment download.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes PDF extraction and easybits API use.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes Google Sheets duplicate comparison logic.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes duplicate alert branch.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes non-duplicate append branch.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Provides overall workflow description and setup guide for easybits, Gmail, Google Sheets, Slack, and activation.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | Gmail Trigger | Poll Gmail for incoming invoice emails and download attachments |  | Extract from File | ## 📧 Email Intake<br>Polls Gmail every minute for emails with the **invoice** label.<br>Downloads attachments automatically.<br><br># 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| Extract from File | Extract From File | Convert Gmail binary attachment into a JSON property | Gmail Trigger | Edit Fields | ## 📄 Invoice Extraction with easybits<br>Extracts the PDF attachment, converts it to base64, and sends it to the **easybits' extractor API** to pull structured invoice data (invoice number, amount, etc.).<br><br># 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| Edit Fields | Set | Build a PDF data URI for the extractor API | Extract from File | HTTP Request | ## 📄 Invoice Extraction with easybits<br>Extracts the PDF attachment, converts it to base64, and sends it to the **easybits' extractor API** to pull structured invoice data (invoice number, amount, etc.).<br><br># 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| HTTP Request | HTTP Request | Send PDF to easybits extractor API | Edit Fields | Check Google Sheets | ## 📄 Invoice Extraction with easybits<br>Extracts the PDF attachment, converts it to base64, and sends it to the **easybits' extractor API** to pull structured invoice data (invoice number, amount, etc.).<br><br># 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| Check Google Sheets | Google Sheets | Read existing invoice rows from the master sheet | HTTP Request | Code in JavaScript | ## 🔍 Duplicate Check<br>Reads all rows from the **Master Finance File** in Google Sheets. A Code node compares the extracted invoice number against existing entries. Returns `duplicate: true/false`.<br><br># 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| Code in JavaScript | Code | Compare extracted invoice number with sheet rows | Check Google Sheets | Already Exists? | ## 🔍 Duplicate Check<br>Reads all rows from the **Master Finance File** in Google Sheets. A Code node compares the extracted invoice number against existing entries. Returns `duplicate: true/false`.<br><br># 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| Already Exists? | If | Branch based on duplicate status | Code in JavaScript | Slack: Alert Finance; Add to Master List | # 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| Slack: Alert Finance | Slack | Send duplicate alert to Finance user in Slack | Already Exists? |  | ## 🚨 Duplicate Found<br>If the invoice **already exists** in the sheet, a Slack DM is sent to the user with the duplicate invoice number for manual review.<br><br># 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| Add to Master List | Google Sheets | Append new invoice to the master sheet | Already Exists? |  | ## ✅ New Invoice<br>If the invoice is **not** a duplicate, it gets appended to the Master Finance File in Google Sheets with the invoice number and total amount.<br><br># 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |
| Sticky Note | Sticky Note | Canvas documentation for Gmail intake |  |  | ## 📧 Email Intake<br>Polls Gmail every minute for emails with the **invoice** label.<br>Downloads attachments automatically. |
| Sticky Note1 | Sticky Note | Canvas documentation for extraction block |  |  | ## 📄 Invoice Extraction with easybits<br>Extracts the PDF attachment, converts it to base64, and sends it to the **easybits' extractor API** to pull structured invoice data (invoice number, amount, etc.). |
| Sticky Note2 | Sticky Note | Canvas documentation for duplicate check block |  |  | ## 🔍 Duplicate Check<br>Reads all rows from the **Master Finance File** in Google Sheets. A Code node compares the extracted invoice number against existing entries. Returns `duplicate: true/false`. |
| Sticky Note3 | Sticky Note | Canvas documentation for duplicate alert branch |  |  | ## 🚨 Duplicate Found<br>If the invoice **already exists** in the sheet, a Slack DM is sent to the user with the duplicate invoice number for manual review. |
| Sticky Note4 | Sticky Note | Canvas documentation for new invoice branch |  |  | ## ✅ New Invoice<br>If the invoice is **not** a duplicate, it gets appended to the Master Finance File in Google Sheets with the invoice number and total amount. |
| Sticky Note5 | Sticky Note | Overall canvas documentation and setup guide |  |  | # 🔍 Invoice Duplicate Checker<br>## How It Works<br>This workflow automatically detects duplicate invoices from Gmail. Incoming PDF attachments are scanned by the easybits data extraction solution, then checked against the Master Finance File in Google Sheets. Duplicates trigger a Slack alert – new invoices get added to the sheet.<br><br>**Flow overview:**<br>1. Gmail picks up new emails labeled as invoices (polls every minute)<br>2. The PDF attachment is extracted and converted to base64<br>3. easybits Extractor reads the document and returns structured data<br>4. The invoice number is compared against all existing entries in Google Sheets<br>5. If duplicate → Slack DM alert to user<br>6. If new → Invoice is appended to the Master Finance File<br><br>---<br><br>## Setup Guide<br><br>**1. easybits Extractor** – Create a pipeline at extractor.easybits.tech with fields: `invoice_number` (String) and `total_amount` (Number). Copy your Pipeline ID and API Key.<br><br>**2. HTTP Request node** – Replace the Pipeline ID in the URL. Add a Bearer Auth credential with your API Key.<br><br>**3. Gmail Trigger** – Connect via OAuth2. Create an "invoice" label in Gmail and set it as the filter. Enable Download Attachments.<br><br>**4. Google Sheets** – Connect both sheet nodes via OAuth2. Select your spreadsheet. Required columns: **Invoice Number** and **Final Amount (EUR)**.<br><br>**5. Slack** – Create a Slack App at [https://api.slack.com/apps](https://api.slack.com/apps). Add scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read`. Install via Settings → Install App. Add the Bot Token as a Slack API credential in n8n.<br><br>**6. Activate** – Toggle the workflow on, label an invoice email, and test. Send the same invoice twice to verify the duplicate alert. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `Invoice Duplicate Checker v2`.

2. **Add a Gmail Trigger node**
   - Node type: **Gmail Trigger**
   - Connect Gmail OAuth2 credentials.
   - Set polling to **every minute**.
   - Disable simple mode if you want the fuller Gmail payload.
   - Enable **Download Attachments**.
   - If you want label-based filtering, configure the Gmail label such as `invoice`.
   - Important: in the provided JSON, label filtering is not actually populated, so set it manually if required.

3. **Add an Extract from File node**
   - Node type: **Extract From File**
   - Connect it after `Gmail Trigger`.
   - Set **Operation** to convert binary to property.
   - Set **Binary Property Name** to `attachment_0`.
   - This means the workflow expects the first email attachment to be the invoice PDF.

4. **Add a Set node named `Edit Fields`**
   - Node type: **Set**
   - Connect it after `Extract from File`.
   - Add one field:
     - Name: `data`
     - Type: `String`
     - Value: `data:application/pdf;base64,{{ $json.data }}`
   - This creates a data URI expected by the easybits API.

5. **Prepare easybits**
   - In easybits Extractor, create a pipeline.
   - Define at least these fields:
     - `invoice_number` as String
     - `total_amount` as Number
   - Copy the pipeline ID and API key.

6. **Add an HTTP Request node**
   - Node type: **HTTP Request**
   - Connect it after `Edit Fields`.
   - Configure:
     - Method: `POST`
     - URL: `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID`
       - Replace `YOUR_PIPELINE_ID` with your real pipeline ID
     - Authentication: **Predefined Credential Type**
     - Credential type: **HTTP Bearer Auth**
     - Create/select a bearer credential containing your easybits API key
     - Send Body: enabled
     - Specify Body: `JSON`
   - Set JSON body to:
     - `{"files":["{{ $json.data }}"]}`
   - Expected response should contain:
     - `data.invoice_number`
     - `data.total_amount`

7. **Prepare Google Sheets**
   - Create or choose a spreadsheet, for example `Master Finance File`.
   - Add a sheet such as `Sheet1`.
   - Ensure the header row contains at least:
     - `Invoice Number`
     - `Final Amount (EUR)`
   - Optional extra columns may include:
     - `Original Amount`
     - `Currency`
     - `Exchange Rate`

8. **Add a Google Sheets node named `Check Google Sheets`**
   - Node type: **Google Sheets**
   - Connect it after `HTTP Request`.
   - Connect Google Sheets OAuth2 credentials.
   - Select the spreadsheet document.
   - Select the target sheet, e.g. `Sheet1`.
   - Configure it to read/list rows from the sheet.
   - Enable **Always Output Data** so the workflow still continues when the sheet is empty.

9. **Add a Code node**
   - Node type: **Code**
   - Connect it after `Check Google Sheets`.
   - Set language to JavaScript.
   - Use logic equivalent to:
     - Read extracted invoice number from the HTTP response
     - Load all incoming rows from Google Sheets
     - Compare each row’s `Invoice Number`
     - Return one item containing `duplicate: "true"` or `duplicate: "false"`
   - Use this logic:
     - Read invoice number from `$('HTTP Request').first().json.data.invoice_number`
     - Compare against `row.json["Invoice Number"]`
     - Trim both values before comparing

10. **Add an IF node named `Already Exists?`**
    - Node type: **If**
    - Connect it after the Code node.
    - Configure a string condition:
      - Value 1: `{{ $json.duplicate }}`
      - Operation: equals
      - Value 2: `true`
    - This routes duplicates to the true branch.

11. **Add a Slack node for duplicate alerts**
    - Node type: **Slack**
    - Connect it to the **true** output of `Already Exists?`.
    - Create a Slack app at [https://api.slack.com/apps](https://api.slack.com/apps).
    - Add scopes:
      - `chat:write`
      - `chat:write.public`
      - `channels:read`
      - `groups:read`
      - `users:read`
      - `users.profile:read`
    - Install the app and create/select Slack credentials in n8n using the bot token.
    - Configure the node to message a user.
    - Select the target Slack user manually.
    - Message text:
      - `🚨 *Duplicate Invoice Detected* Invoice number {{ $('HTTP Request').first().json.data.invoice_number }} was already submitted. Please review before processing.`

12. **Add a Google Sheets node named `Add to Master List`**
    - Node type: **Google Sheets**
    - Connect it to the **false** output of `Already Exists?`.
    - Use the same Google Sheets credentials as above.
    - Select the same spreadsheet and sheet.
    - Set **Operation** to `Append`.
    - Enable manual column mapping.
    - Map:
      - `Invoice Number` → `{{ $('HTTP Request').first().json.data.invoice_number }}`
      - `Final Amount (EUR)` → `{{ $('HTTP Request').first().json.data.total_amount }}`
    - Leave other columns empty unless you also extract and map them.

13. **Optionally add sticky notes**
    - Add notes for:
      - Email intake
      - easybits extraction
      - Duplicate check
      - Duplicate branch
      - New invoice branch
      - Overall setup guide

14. **Test with a non-duplicate invoice**
    - Send or label an email containing one invoice PDF.
    - Confirm:
      - Gmail trigger runs
      - easybits extracts fields correctly
      - Google Sheets append occurs
      - No Slack alert is sent

15. **Test with a duplicate**
    - Send the same invoice again or process the same invoice number twice.
    - Confirm:
      - Code node returns `duplicate: "true"`
      - IF node follows the true branch
      - Slack alert is sent
      - No additional Google Sheets row is appended

16. **Activate the workflow**
    - Once all credentials and fields are validated, activate the workflow.

## Credential configuration summary
- **Gmail Trigger:** Gmail OAuth2
- **HTTP Request:** HTTP Bearer Auth with easybits API key
- **Google Sheets nodes:** Google Sheets OAuth2
- **Slack:** Slack API credential using bot token

## Input/output expectations
- **Input email:** Should contain at least one attachment, with the invoice PDF as `attachment_0`
- **Extractor response:** Must expose `data.invoice_number` and `data.total_amount`
- **Google Sheet:** Must include `Invoice Number` and `Final Amount (EUR)` columns
- **Slack:** Must have a valid recipient user selected

## No sub-workflows
This workflow does not invoke any sub-workflow and has only one entry point: `Gmail Trigger`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| easybits Extractor pipeline should define `invoice_number` as String and `total_amount` as Number | easybits setup |
| Replace `YOUR_PIPELINE_ID` in the HTTP Request URL with your real extractor pipeline ID | easybits API configuration |
| Gmail should ideally use an `invoice` label filter, but this is not fully configured in the exported JSON and must be set manually | Gmail Trigger configuration |
| Required Google Sheets columns are **Invoice Number** and **Final Amount (EUR)** | Spreadsheet structure |
| Slack app scopes listed in the canvas note: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users.profile:read` | Slack app setup / [https://api.slack.com/apps](https://api.slack.com/apps) |
| The workflow assumes the invoice is the first attachment (`attachment_0`) | Attachment handling |
| Duplicate detection is based only on exact invoice number matching after trimming whitespace | Matching logic |
| The amount is stored in a column named `Final Amount (EUR)`, but the extractor output is written directly without currency validation or conversion | Data modeling note |