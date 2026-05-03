Generate and email PDF quotes from Airtable via Gmail and Google Drive

https://n8nworkflows.xyz/workflows/generate-and-email-pdf-quotes-from-airtable-via-gmail-and-google-drive-14174


# Generate and email PDF quotes from Airtable via Gmail and Google Drive

# 1. Workflow Overview

This workflow generates a branded PDF quote from an Airtable record, stores the PDF in Google Drive, emails it to the client via Gmail, and then updates the Airtable record to mark the quote as sent.

Typical use cases:
- Sending sales quotes directly from Airtable records
- Triggering quote generation from Airtable automations
- Generating standardized branded quote PDFs without manual document editing

## 1.1 Trigger Reception
The workflow starts from a webhook that accepts a POST request containing an Airtable record ID. This ID identifies the quote record to process.

## 1.2 Runtime Configuration
A Set node centralizes editable business settings such as business name, contact details, brand color, Airtable table name, and Google Drive folder ID. It also extracts the incoming Airtable record ID from the webhook payload or query string.

## 1.3 Quote Record Retrieval
Using the record ID from the trigger, the workflow fetches the quote record from Airtable.

## 1.4 Quote HTML Construction
A Code node parses quote fields, converts line items into structured rows, calculates subtotal/tax/total, and builds a complete branded HTML quote document.

## 1.5 PDF Generation and Delivery
The HTML is sent to pdf.co for conversion into PDF. The generated file is downloaded as binary, saved to Google Drive, and emailed to the client as an attachment.

## 1.6 Airtable Status Update
After the PDF is saved, the workflow updates the Airtable record with sent status, sent date, and the Google Drive file link.

---

# 2. Block-by-Block Analysis

## Block 1: Trigger Reception

### Overview
This block exposes an HTTP endpoint that starts the quote-generation process. It accepts a POST request and passes the incoming payload to the configuration stage.

### Nodes Involved
- Quote Generation Webhook

### Node Details

#### Quote Generation Webhook
- **Type and role:** `n8n-nodes-base.webhook` — entry point for external systems
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `generate-quote`
  - No advanced options configured
- **Key expressions or variables used:**
  - None directly in this node
  - Its payload is later read by the Set node via:
    - `$json.body.recordId`
    - `$json.query.recordId`
- **Input and output connections:**
  - Input: none
  - Output: `Configure Settings`
- **Version-specific requirements:**
  - Uses webhook node version `2`
- **Edge cases / failures:**
  - Request may omit `recordId`
  - If called with wrong method, the endpoint will reject the request
  - If test URL is used incorrectly instead of production URL after activation, calls may appear to fail
- **Sub-workflow reference:** none

---

## Block 2: Runtime Configuration

### Overview
This block defines all editable business and integration settings in one place and extracts the Airtable record ID from the webhook request. It acts as the workflow’s runtime configuration source.

### Nodes Involved
- Configure Settings

### Node Details

#### Configure Settings
- **Type and role:** `n8n-nodes-base.set` — assigns static configuration values and derives runtime variables
- **Configuration choices:**
  - Defines:
    - `airtableBaseId`
    - `airtableTableName`
    - `businessName`
    - `businessEmail`
    - `businessAddress`
    - `businessPhone`
    - `brandColor`
    - `googleDriveFolderId`
    - `recordId`
  - `recordId` is dynamically assigned from webhook body or query string
- **Key expressions or variables used:**
  - `={{ $json.body.recordId || $json.query.recordId }}`
- **Input and output connections:**
  - Input: `Quote Generation Webhook`
  - Output: `Read Quote from Airtable`
- **Version-specific requirements:**
  - Uses Set node version `3.4`
- **Edge cases / failures:**
  - Placeholder values like `YOUR_AIRTABLE_BASE_ID` and `YOUR_GOOGLE_DRIVE_FOLDER_ID` must be replaced
  - If `recordId` is undefined, downstream Airtable read will fail
  - If business data is not customized, generated quotes will contain placeholder branding/contact info
- **Sub-workflow reference:** none

---

## Block 3: Quote Record Retrieval

### Overview
This block retrieves a single quote record from Airtable using the record ID supplied earlier. The result becomes the source data for PDF generation.

### Nodes Involved
- Read Quote from Airtable

### Node Details

#### Read Quote from Airtable
- **Type and role:** `n8n-nodes-base.airtable` — fetches one Airtable record
- **Configuration choices:**
  - Record ID: `={{ $json.recordId }}`
  - Table name is dynamic from configuration
  - Base is intended to use Airtable credentials/base selection, but the JSON shows the base selector value is empty and must be configured manually
- **Key expressions or variables used:**
  - `={{ $json.recordId }}`
  - `={{ $json.airtableTableName }}`
- **Input and output connections:**
  - Input: `Configure Settings`
  - Output: `Build HTML Quote`
- **Version-specific requirements:**
  - Uses Airtable node version `2.1`
- **Edge cases / failures:**
  - Airtable credentials missing or invalid
  - Base not selected/configured
  - Record ID not found
  - Table name mismatch
  - Missing expected fields such as `Client Name`, `Line Items`, or `Tax Rate`
- **Sub-workflow reference:** none

---

## Block 4: Quote HTML Construction

### Overview
This block transforms the Airtable record into a finished HTML quote. It also calculates pricing values and prepares metadata needed for PDF naming, emailing, Drive upload, and Airtable update.

### Nodes Involved
- Build HTML Quote

### Node Details

#### Build HTML Quote
- **Type and role:** `n8n-nodes-base.code` — custom JavaScript transformation and HTML template rendering
- **Configuration choices:**
  - Reads config from the `Configure Settings` node
  - Reads the Airtable record from current input
  - Parses line items from a multiline text field using the format:
    - `Description | Qty | Price`
  - Computes:
    - `quoteRef`
    - `today`
    - `validUntil`
    - `subtotal`
    - `tax`
    - `grandTotal`
  - Returns HTML plus operational metadata
- **Key expressions or variables used:**
  - `$('Configure Settings').first().json`
  - `$input.first().json`
  - Airtable fields expected:
    - `Client Name`
    - `Client Email`
    - `Line Items`
    - `Tax Rate`
    - `Notes`
  - Output fields include:
    - `html`
    - `clientName`
    - `clientEmail`
    - `quoteRef`
    - `grandTotal`
    - `today`
    - `recordId`
    - `airtableBaseId`
    - `airtableTableName`
    - `googleDriveFolderId`
    - `businessName`
- **Input and output connections:**
  - Input: `Read Quote from Airtable`
  - Output: `Convert HTML to PDF`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases / failures:**
  - Invalid `Line Items` formatting can lead to partial or incorrect parsing
  - `parseInt` or `parseFloat` may yield `NaN` for malformed quantity or price values
  - Empty line items produce an empty table and zero totals
  - HTML content is not escaped; special characters in notes or line item descriptions could affect rendering
  - `quoteRef` is time-based and not persisted back as a dedicated Airtable field
- **Sub-workflow reference:** none

---

## Block 5: PDF Generation and Delivery

### Overview
This block converts the generated HTML into a PDF using pdf.co, downloads the file, uploads it to Google Drive, and emails it to the client with the PDF attached.

### Nodes Involved
- Convert HTML to PDF
- Download PDF File
- Save PDF to Google Drive
- Email Quote to Client

### Node Details

#### Convert HTML to PDF
- **Type and role:** `n8n-nodes-base.httpRequest` — sends HTML to pdf.co conversion API
- **Configuration choices:**
  - URL: `https://api.pdf.co/v1/pdf/convert/from/html`
  - Method: `POST`
  - Body sent as JSON
  - Auth type: predefined credential, `httpHeaderAuth`
  - Expected response: JSON
  - Body includes:
    - `html`
    - output file `name`
    - `margins`
    - `paperSize`
- **Key expressions or variables used:**
  - `={{ JSON.stringify({ html: $json.html, name: $json.quoteRef + '.pdf', margins: '20px', paperSize: 'A4' }) }}`
- **Input and output connections:**
  - Input: `Build HTML Quote`
  - Output: `Download PDF File`
- **Version-specific requirements:**
  - Uses HTTP Request version `4.2`
  - Requires an HTTP Header Auth credential configured with pdf.co API key
- **Edge cases / failures:**
  - Invalid or missing pdf.co API key
  - HTML too large or unsupported by service limits
  - API quota exhaustion on free plan
  - Conversion errors if HTML/CSS is unsupported
- **Sub-workflow reference:** none

#### Download PDF File
- **Type and role:** `n8n-nodes-base.httpRequest` — downloads the generated PDF as binary data
- **Configuration choices:**
  - URL comes from pdf.co response field `url`
  - Response format is `file`
- **Key expressions or variables used:**
  - `={{ $json.url }}`
- **Input and output connections:**
  - Input: `Convert HTML to PDF`
  - Outputs:
    - `Save PDF to Google Drive`
    - `Email Quote to Client`
- **Version-specific requirements:**
  - Uses HTTP Request version `4.2`
- **Edge cases / failures:**
  - Missing `url` in pdf.co response
  - Temporary download URL expired
  - Network timeout while downloading file
  - If binary property naming changes unexpectedly, attachment/upload steps may fail
- **Sub-workflow reference:** none

#### Save PDF to Google Drive
- **Type and role:** `n8n-nodes-base.googleDrive` — uploads the binary PDF file to a Drive folder
- **Configuration choices:**
  - File name: quote reference plus `.pdf`
  - Drive: `My Drive`
  - Folder ID is read from `Build HTML Quote`
- **Key expressions or variables used:**
  - `={{ $('Build HTML Quote').first().json.quoteRef }}.pdf`
  - `={{ $('Build HTML Quote').first().json.googleDriveFolderId }}`
- **Input and output connections:**
  - Input: `Download PDF File`
  - Output: `Update Airtable Status`
- **Version-specific requirements:**
  - Uses Google Drive node version `3`
  - Requires Google Drive credentials
- **Edge cases / failures:**
  - Folder ID invalid or inaccessible
  - Google Drive credentials missing or insufficient permissions
  - If the node is not configured to receive the correct binary property, upload may fail
  - Drive API rate limits or quota issues
- **Sub-workflow reference:** none

#### Email Quote to Client
- **Type and role:** `n8n-nodes-base.gmail` — sends the quote PDF by email
- **Configuration choices:**
  - Recipient is the client email from the generated quote data
  - Subject and message are dynamically composed
  - Attachment is taken from binary input using attachments UI
- **Key expressions or variables used:**
  - `={{ $('Build HTML Quote').first().json.clientEmail }}`
  - Subject:
    - `=Quote {{ $('Build HTML Quote').first().json.quoteRef }} from {{ $('Build HTML Quote').first().json.businessName }}`
  - Message:
    - includes client name, quote ref, total, and business name
- **Input and output connections:**
  - Input: `Download PDF File`
  - Output: none
- **Version-specific requirements:**
  - Uses Gmail node version `2.1`
  - Requires Gmail credentials/OAuth2
- **Edge cases / failures:**
  - Missing or invalid client email
  - Gmail OAuth scope/credential problems
  - Attachment may fail if binary data is not present
  - Gmail sending limits or account restrictions
- **Sub-workflow reference:** none

---

## Block 6: Airtable Status Update

### Overview
This block writes back to Airtable after Drive upload succeeds. It marks the quote as sent and stores a PDF link and sent date.

### Nodes Involved
- Update Airtable Status

### Node Details

#### Update Airtable Status
- **Type and role:** `n8n-nodes-base.airtable` — updates the original Airtable record
- **Configuration choices:**
  - Operation: `update`
  - Table name from `Build HTML Quote`
  - Columns explicitly mapped:
    - `Status` = `Sent`
    - `PDF Link` = Google Drive `webViewLink`
    - `Sent Date` = current quote date
  - The JSON shows base selection empty and requires manual setup
  - Important: an Airtable update operation also needs the record ID configured; this is not visible in the provided JSON and must be added manually when rebuilding
- **Key expressions or variables used:**
  - `={{ $('Build HTML Quote').first().json.airtableTableName }}`
  - `={{ $('Save PDF to Google Drive').first().json.webViewLink || '' }}`
  - `={{ $('Build HTML Quote').first().json.today }}`
- **Input and output connections:**
  - Input: `Save PDF to Google Drive`
  - Output: none
- **Version-specific requirements:**
  - Uses Airtable node version `2.1`
- **Edge cases / failures:**
  - Record ID missing in update configuration will prevent proper update
  - Airtable fields `Status`, `PDF Link`, or `Sent Date` may not exist
  - Credentials/base/table mismatches
  - If Drive upload succeeds but Airtable update fails, the quote is sent but status remains outdated
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Quote Generation Webhook | Webhook | Receives POST request with Airtable record ID |  | Configure Settings | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 1. Trigger<br>Webhook receives a POST with the Airtable record ID. You can call this from an Airtable automation or any HTTP client. |
| Configure Settings | Set | Stores editable business and integration settings, extracts record ID | Quote Generation Webhook | Read Quote from Airtable | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 2. Your settings<br>All the stuff you need to change is here. Business name, email, brand color, Drive folder, etc. |
| Read Quote from Airtable | Airtable | Fetches the quote record by Airtable record ID | Configure Settings | Build HTML Quote | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 3. Pull the quote<br>Grabs the record from Airtable using the ID from the webhook. |
| Build HTML Quote | Code | Parses Airtable fields, computes totals, renders branded HTML | Read Quote from Airtable | Convert HTML to PDF | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 4. Build the quote<br>This is where the magic happens. Parses line items, calculates totals, and builds a clean HTML document you can customize. |
| Convert HTML to PDF | HTTP Request | Calls pdf.co to convert generated HTML into a PDF | Build HTML Quote | Download PDF File | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 5. PDF, save, and send<br>Converts the HTML to PDF, uploads it to your Drive folder, and emails it to the client. The Airtable record gets updated with the link so you know it went out. |
| Download PDF File | HTTP Request | Downloads the generated PDF as binary | Convert HTML to PDF | Save PDF to Google Drive, Email Quote to Client | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 5. PDF, save, and send<br>Converts the HTML to PDF, uploads it to your Drive folder, and emails it to the client. The Airtable record gets updated with the link so you know it went out. |
| Save PDF to Google Drive | Google Drive | Uploads the PDF file to Google Drive | Download PDF File | Update Airtable Status | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 5. PDF, save, and send<br>Converts the HTML to PDF, uploads it to your Drive folder, and emails it to the client. The Airtable record gets updated with the link so you know it went out. |
| Email Quote to Client | Gmail | Sends email with the PDF quote attached | Download PDF File |  | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 5. PDF, save, and send<br>Converts the HTML to PDF, uploads it to your Drive folder, and emails it to the client. The Airtable record gets updated with the link so you know it went out. |
| Update Airtable Status | Airtable | Marks quote as sent and stores PDF link/date | Save PDF to Google Drive |  | ### Generate and email PDF quotes from Airtable records<br>If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there.<br>### How it works<br>1. You trigger the webhook with a record ID (either from an Airtable automation or a manual HTTP call)<br>2. It pulls the quote record from your Airtable base<br>3. A code node takes your business details and the line items, then builds a proper branded HTML quote with subtotals, tax, and a grand total<br>4. That HTML gets converted to a PDF via pdf.co (free tier works fine)<br>5. The PDF lands in your Google Drive folder and gets emailed to the client as an attachment<br>6. Finally, the Airtable record gets marked as "Sent" with the Drive link attached<br>### Setup<br>1. You need an Airtable base with a "Quotes" table. Columns: Client Name, Client Email, Line Items (long text field, one item per line like `Website Design | 1 | 2500`), Tax Rate, Notes, Status<br>2. Open the "Configure Settings" node and fill in your business name, email, address, and Drive folder ID<br>3. Connect your Airtable, Gmail, and Google Drive credentials<br>4. Grab a free API key from pdf.co and add it as an HTTP Header Auth credential<br>5. Activate and test with a real Airtable record ID<br>## 5. PDF, save, and send<br>Converts the HTML to PDF, uploads it to your Drive folder, and emails it to the client. The Airtable record gets updated with the link so you know it went out. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Name it: `Generate and email PDF quotes from Airtable via Gmail and Google Drive`.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - Name: `Quote Generation Webhook`
   - HTTP Method: `POST`
   - Path: `generate-quote`
   - Leave options at default unless you need authentication or custom response handling.
   - This node should accept a payload like:
     - JSON body: `{ "recordId": "recXXXXXXXXXXXXXX" }`
     - or query parameter: `?recordId=recXXXXXXXXXXXXXX`

3. **Add a Set node after the webhook**
   - Node type: `Set`
   - Name: `Configure Settings`
   - Create the following fields:
     - `airtableBaseId` = your Airtable base ID
     - `airtableTableName` = `Quotes`
     - `businessName` = your company name
     - `businessEmail` = your business email
     - `businessAddress` = your address
     - `businessPhone` = your phone
     - `brandColor` = for example `#2563eb`
     - `googleDriveFolderId` = target Google Drive folder ID
     - `recordId` = expression `{{ $json.body.recordId || $json.query.recordId }}`
   - Connect `Quote Generation Webhook -> Configure Settings`

4. **Add an Airtable node to read the quote**
   - Node type: `Airtable`
   - Name: `Read Quote from Airtable`
   - Operation: read/get a record by ID
   - Credentials: connect your Airtable credential
   - Base: select your Airtable base
   - Table: use expression `{{ $json.airtableTableName }}`
   - Record ID: use expression `{{ $json.recordId }}`
   - Connect `Configure Settings -> Read Quote from Airtable`

5. **Prepare the Airtable table structure**
   - Ensure your `Quotes` table has at least these fields:
     - `Client Name`
     - `Client Email`
     - `Line Items`
     - `Tax Rate`
     - `Notes`
     - `Status`
   - Recommended additional fields for this workflow:
     - `PDF Link`
     - `Sent Date`

6. **Add a Code node**
   - Node type: `Code`
   - Name: `Build HTML Quote`
   - Set mode to JavaScript
   - Paste logic equivalent to the provided code:
     - Read config from `Configure Settings`
     - Read current Airtable record
     - Parse `Line Items` where each line follows:
       - `Description | Qty | Price`
     - Calculate subtotal, tax, total
     - Generate a quote reference from the current timestamp
     - Build full HTML for the quote
     - Return JSON containing:
       - `html`
       - `clientName`
       - `clientEmail`
       - `quoteRef`
       - `grandTotal`
       - `today`
       - `recordId`
       - `airtableBaseId`
       - `airtableTableName`
       - `googleDriveFolderId`
       - `businessName`
   - Connect `Read Quote from Airtable -> Build HTML Quote`

7. **Add an HTTP Request node for PDF conversion**
   - Node type: `HTTP Request`
   - Name: `Convert HTML to PDF`
   - Method: `POST`
   - URL: `https://api.pdf.co/v1/pdf/convert/from/html`
   - Authentication: `Predefined Credential Type`
   - Credential type: `HTTP Header Auth`
   - Create an HTTP Header Auth credential for pdf.co:
     - Header name typically: `x-api-key`
     - Header value: your pdf.co API key
   - Send Body: enabled
   - Body Content Type: JSON
   - JSON body:
     - `html` = `{{ $json.html }}`
     - `name` = `{{ $json.quoteRef + '.pdf' }}`
     - `margins` = `20px`
     - `paperSize` = `A4`
   - Response format: JSON
   - Connect `Build HTML Quote -> Convert HTML to PDF`

8. **Add another HTTP Request node to download the PDF**
   - Node type: `HTTP Request`
   - Name: `Download PDF File`
   - Method: `GET`
   - URL: `{{ $json.url }}`
   - Response format: `File`
   - This should output binary data
   - Connect `Convert HTML to PDF -> Download PDF File`

9. **Add a Google Drive node**
   - Node type: `Google Drive`
   - Name: `Save PDF to Google Drive`
   - Credentials: connect Google Drive OAuth2 credentials
   - Operation: upload file
   - Drive: `My Drive`
   - Folder ID: `{{ $('Build HTML Quote').first().json.googleDriveFolderId }}`
   - File name: `{{ $('Build HTML Quote').first().json.quoteRef }}.pdf`
   - Ensure the node uses the binary file from `Download PDF File`
   - Connect `Download PDF File -> Save PDF to Google Drive`

10. **Add a Gmail node**
    - Node type: `Gmail`
    - Name: `Email Quote to Client`
    - Credentials: connect Gmail OAuth2 credentials
    - Operation: send email
    - To: `{{ $('Build HTML Quote').first().json.clientEmail }}`
    - Subject:
      - `Quote {{ $('Build HTML Quote').first().json.quoteRef }} from {{ $('Build HTML Quote').first().json.businessName }}`
    - HTML message body:
      - Greeting with client name
      - Mention attached quote reference
      - Mention total amount
      - Mention validity of 30 days
      - Signature with business name
    - Attachments:
      - Use the binary output from `Download PDF File`
    - Connect `Download PDF File -> Email Quote to Client`

11. **Add a second Airtable node for status update**
    - Node type: `Airtable`
    - Name: `Update Airtable Status`
    - Credentials: same Airtable credential
    - Operation: `Update`
    - Base: select the same Airtable base
    - Table: `{{ $('Build HTML Quote').first().json.airtableTableName }}`
    - Record ID: `{{ $('Build HTML Quote').first().json.recordId }}`
    - Map fields:
      - `Status` = `Sent`
      - `PDF Link` = `{{ $('Save PDF to Google Drive').first().json.webViewLink || '' }}`
      - `Sent Date` = `{{ $('Build HTML Quote').first().json.today }}`
    - Connect `Save PDF to Google Drive -> Update Airtable Status`

12. **Verify binary handling**
    - Confirm that:
      - `Download PDF File` produces binary output
      - `Save PDF to Google Drive` uses that binary property
      - `Email Quote to Client` attaches that same binary property
    - In some n8n versions, you may need to explicitly set the binary property name such as `data`.

13. **Configure credentials**
    - **Airtable**
      - API token or OAuth depending on your n8n setup
      - Must have access to the selected base and table
    - **Google Drive**
      - OAuth2 with permission to create files in the target folder
    - **Gmail**
      - OAuth2 with permission to send email
    - **pdf.co**
      - HTTP Header Auth credential using your API key

14. **Test with a sample Airtable record**
    - Create a record in Airtable with:
      - valid client name and email
      - line items formatted one per line, for example:
        - `Website Design | 1 | 2500`
        - `Hosting | 12 | 20`
      - tax rate like `20`
    - Trigger the webhook with the Airtable record ID.

15. **Example webhook call**
    - POST to your webhook URL with JSON body:
      - `{ "recordId": "rec1234567890" }`

16. **Validate outputs**
    - Confirm the quote is:
      - retrieved from Airtable
      - converted to HTML correctly
      - turned into a PDF by pdf.co
      - uploaded to Drive
      - emailed to the client
      - updated in Airtable as sent

17. **Recommended hardening improvements**
    - Add an IF node to stop execution when `recordId` is missing
    - Add validation for empty `clientEmail`
    - Add error handling if pdf.co returns no URL
    - Escape HTML-sensitive user content in notes and line items
    - Add a Merge or success-tracking pattern if you want Airtable updated only after both Drive upload and email succeed

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Generate and email PDF quotes from Airtable records. If you're managing quotes in Airtable and still copying data into a Google Doc or Word template every time, this saves you the hassle. Hit the webhook, and the workflow handles everything from there. | Workflow-level note |
| How it works: 1. Trigger webhook with record ID. 2. Pull Airtable record. 3. Build branded HTML quote with totals. 4. Convert to PDF via pdf.co. 5. Save to Google Drive and email to client. 6. Mark Airtable record as Sent with Drive link. | Workflow-level note |
| Setup: Airtable base with a `Quotes` table. Columns: Client Name, Client Email, Line Items, Tax Rate, Notes, Status. Configure business details and Drive folder in `Configure Settings`. Connect Airtable, Gmail, and Google Drive credentials. Add pdf.co API key as HTTP Header Auth. Test with a real Airtable record ID. | Workflow-level note |
| Trigger block note: Webhook receives a POST with the Airtable record ID. You can call this from an Airtable automation or any HTTP client. | Trigger context |
| Settings block note: All the stuff you need to change is here. Business name, email, brand color, Drive folder, etc. | Configuration context |
| Pull block note: Grabs the record from Airtable using the ID from the webhook. | Airtable read context |
| Build block note: Parses line items, calculates totals, and builds a clean HTML document you can customize. | HTML generation context |
| Delivery block note: Converts the HTML to PDF, uploads it to your Drive folder, and emails it to the client. The Airtable record gets updated with the link so you know it went out. | PDF / delivery context |