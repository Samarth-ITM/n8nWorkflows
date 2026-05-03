Extract Invoice Data from Google Drive to Sheets using PDF.co AI Parser

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-google-drive-to-sheets-using-pdf-co-ai-parser-7862


# Extract Invoice Data from Google Drive to Sheets using PDF.co AI Parser

### 1. Workflow Overview

This workflow automates the extraction of invoice data stored as PDF files in a specific Google Drive folder. It leverages PDF.co’s AI Invoice Parser to interpret and extract key invoice details, then appends or updates this structured data into a Google Sheets document for record-keeping and analysis.

**Target Use Cases:**  
- Automated bookkeeping and invoice management  
- Centralizing invoice metadata for accounting or finance teams  
- Reducing manual data entry from PDF invoices  

**Logical Blocks:**

- **1.1 Workflow Trigger** – Manual start of the workflow.  
- **1.2 Google Drive Folder Access** – Locates the parent folder and retrieves all invoice PDFs inside it.  
- **1.3 URL Conversion** – Converts Google Drive file IDs into accessible URLs for parsing.  
- **1.4 AI Invoice Parsing** – Uses PDF.co AI to extract structured invoice data from PDFs.  
- **1.5 Data Formatting** – Extracts and formats relevant fields from the parsed JSON.  
- **1.6 Google Sheets Storage** – Appends or updates the invoice data into a defined Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Workflow Trigger

- **Overview:** Initiates the entire workflow manually.  
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
- **Node Details:**  
  - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
  - Configuration: No parameters; triggers workflow execution upon manual command.  
  - Inputs: None  
  - Outputs: Starts flow to “Get Parent Folder ID”  
  - Edge cases: None significant; manual trigger requires user action.

---

#### 1.2 Google Drive Folder Access

- **Overview:** Finds the Google Drive folder named "n8n Invoices" and retrieves all files (invoices) within that folder.  
- **Nodes Involved:**  
  - Get Parent Folder ID  
  - Get Invoice ID's  
- **Node Details:**  

  **Get Parent Folder ID**  
  - Type: Google Drive node (fileFolder resource)  
  - Role: Search for folder named “n8n Invoices” to get its ID  
  - Configuration: Query string set to "n8n Invoices"  
  - Credentials: Google Drive OAuth2 configured  
  - Inputs: Trigger node output  
  - Outputs: Folder metadata including folder ID to next node  
  - Edge cases: Folder not found (no results or API error), permission denied

  **Get Invoice ID's**  
  - Type: Google Drive node (fileFolder resource)  
  - Role: List files inside the folder identified by previous node’s ID  
  - Configuration: Filter by folderId using expression `={{ $json.id }}` (dynamic from prior node)  
  - Credentials: Same as above  
  - Inputs: Output of “Get Parent Folder ID”  
  - Outputs: List of invoice files (IDs and names)  
  - Edge cases: Folder empty or inaccessible, API rate limits, permission errors

---

#### 1.3 URL Conversion

- **Overview:** Converts each invoice file ID into a Google Drive accessible URL to feed into the AI parser.  
- **Nodes Involved:**  
  - Convert to URL  
- **Node Details:**  
  - Type: Set node (data transformation)  
  - Role: Creates a new field `link` with a URL template `"https://drive.google.com/file/d/{{ $json.id }}"` for each file  
  - Configuration: Assignment of `link` field as string expression  
  - Inputs: List of invoice files from “Get Invoice ID’s”  
  - Outputs: Items enriched with `link`  
  - Edge cases: Missing or malformed file ID, invalid URL format

---

#### 1.4 AI Invoice Parsing

- **Overview:** Sends each Google Drive URL of a PDF invoice to PDF.co’s AI Invoice Parser to extract structured data.  
- **Nodes Involved:**  
  - AI Invoice Parser  
- **Node Details:**  
  - Type: PDF.co API node  
  - Role: Calls PDF.co’s AI parser with invoice URL  
  - Configuration: URL parameter dynamically set to `{{ $json.link }}`; uses PDF.co API credentials  
  - Inputs: URLs from “Convert to URL”  
  - Outputs: JSON response containing parsed invoice details  
  - Version-specific: Requires valid PDF.co API key  
  - Edge cases: API key invalid or expired, rate limiting, invalid PDF file, network timeout, malformed response

---

#### 1.5 Data Formatting

- **Overview:** Extracts relevant fields (Vendor, Invoice Date, Total, Due Date, URL) from the parsed JSON response and formats them into a flattened structure.  
- **Nodes Involved:**  
  - Set Fields  
- **Node Details:**  
  - Type: Set node  
  - Role: Maps nested JSON fields to top-level properties for Google Sheets  
  - Configuration: Assignments with expressions such as:  
    - Vendor: `{{$json.body.vendor.name}}`  
    - Invoice Date: `{{$json.body.invoice.invoiceDate}}`  
    - Total: `{{$json.body.paymentDetails.total}}`  
    - Due Date: `{{$json.body.paymentDetails.dueDate}}`  
    - url: `{{$('Convert to URL').item.json.link}}` (refers back to original URL)  
  - Inputs: Parsed JSON from “AI Invoice Parser”  
  - Outputs: Structured invoice data for saving  
  - Edge cases: Missing fields in JSON, expression evaluation errors, null values

---

#### 1.6 Google Sheets Storage

- **Overview:** Adds or updates the extracted invoice data in a Google Sheet named “Invoice Template” in the “Due” tab, matching existing rows by the URL to avoid duplicates.  
- **Nodes Involved:**  
  - Store Data in Google Sheets  
- **Node Details:**  
  - Type: Google Sheets node  
  - Role: Append or update rows in spreadsheet  
  - Configuration:  
    - Operation: Append or Update  
    - Document ID: Spreadsheet ID for “Invoice Template”  
    - Sheet Name: Tab “Due” by GID  
    - Columns mapped: Url, Vendor, Invoice Date, Total, Due Date  
    - Matching column: Url (to prevent duplicates)  
  - Credentials: Google Sheets OAuth2 configured  
  - Inputs: Structured data from “Set Fields”  
  - Outputs: None (terminal node)  
  - Edge cases: Spreadsheet access errors, permission denied, schema mismatch, API limits

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                              | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                               |
|------------------------------|-------------------------------|----------------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                | Starts the workflow manually                  | None                         | Get Parent Folder ID         |                                                                                                                                           |
| Get Parent Folder ID          | Google Drive                  | Finds the folder named “n8n Invoices”         | When clicking ‘Execute workflow’ | Get Invoice ID's             | ### 1) Connect Google Drive (OAuth2) Setup instructions present in sticky note near this node                                              |
| Get Invoice ID's              | Google Drive                  | Lists files inside the parent folder           | Get Parent Folder ID          | Convert to URL              | See above; uses folder ID from previous node                                                                                              |
| Convert to URL                | Set                          | Converts file IDs to accessible Google Drive URLs | Get Invoice ID's              | AI Invoice Parser           | ### 2) Connect PDF.co (AI Invoice Parser) Setup instructions in sticky note near this node                                                 |
| AI Invoice Parser             | PDF.co API                   | Extracts invoice data from PDFs using AI      | Convert to URL                | Set Fields                  | See above; requires PDF.co API credentials                                                                                                |
| Set Fields                   | Set                          | Formats extracted data fields for Google Sheets | AI Invoice Parser             | Store Data in Google Sheets |                                                                                                                                           |
| Store Data in Google Sheets   | Google Sheets                | Appends/updates invoice data in Google Sheet  | Set Fields                   | None                        | ### 3) Connect Google Sheets (OAuth2) Setup instructions present in sticky note near this node                                             |
| Sticky Note2                  | Sticky Note                  | Setup instructions for Google Drive, PDF.co, Google Sheets | None                         | None                        | Contains detailed setup instructions for all three integrations (Google Drive, PDF.co, Google Sheets)                                     |
| Sticky Note55                 | Sticky Note                  | Workflow summary description                   | None                         | None                        | Describes overall workflow purpose: Google Drive → AI Invoice Parser → Google Sheets (Due Log)                                            |
| Sticky Note61                 | Sticky Note                  | Google Sheets OAuth2 setup details             | None                         | None                        | Focused on Google Sheets connection setup                                                                                                |
| Sticky Note62                 | Sticky Note                  | Google Drive OAuth2 setup details              | None                         | None                        | Focused on Google Drive connection setup                                                                                                 |
| Sticky Note64                 | Sticky Note                  | PDF.co API setup details                        | None                         | None                        | Focused on PDF.co API connection setup                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed. This node will manually trigger the workflow run.

3. **Add a Google Drive node:**  
   - Name: `Get Parent Folder ID`  
   - Resource: `fileFolder`  
   - Operation: List files/folders  
   - Query String: `n8n Invoices` (the folder name to search)  
   - Credentials: Create or select Google Drive OAuth2 credentials linked to the account owning the invoices.  
   - Connect output from Manual Trigger node.

4. **Add another Google Drive node:**  
   - Name: `Get Invoice ID's`  
   - Resource: `fileFolder`  
   - Operation: List files/folders  
   - Filter: Set `folderId` to use the expression `={{ $json.id }}` referencing the previous node’s folder ID.  
   - Credentials: Use same Google Drive OAuth2 credentials.  
   - Connect output from `Get Parent Folder ID`.

5. **Add a Set node:**  
   - Name: `Convert to URL`  
   - Purpose: Create a new field `link` with value `https://drive.google.com/file/d/{{ $json.id }}`  
   - Configuration: Under assignments, add a string field `link` with value `=https://drive.google.com/file/d/{{ $json.id }}`  
   - Connect output from `Get Invoice ID's`.

6. **Add a PDF.co API node:**  
   - Name: `AI Invoice Parser`  
   - Operation: Call the AI Invoice Parser with URL parameter set to `={{ $json.link }}` from previous node.  
   - Credentials: Set up PDF.co API credentials with your API key.  
   - Connect output from `Convert to URL`.

7. **Add another Set node:**  
   - Name: `Set Fields`  
   - Purpose: Extract and flatten necessary invoice fields from the parsed JSON.  
   - Assign these fields:  
     - Vendor: `={{ $json.body.vendor.name }}`  
     - Invoice Date: `={{ $json.body.invoice.invoiceDate }}`  
     - Total: `={{ $json.body.paymentDetails.total }}`  
     - Due Date: `={{ $json.body.paymentDetails.dueDate }}`  
     - url: `={{ $('Convert to URL').item.json.link }}` (reference link from earlier)  
   - Connect output from `AI Invoice Parser`.

8. **Add a Google Sheets node:**  
   - Name: `Store Data in Google Sheets`  
   - Operation: Append or Update  
   - Document ID: Set to the Google Sheets file ID (e.g., `1a6QBIQkr7RsZtUZBi87NwhwgTbnr5hQl4J_ZOkr3F1U`)  
   - Sheet Name: Use the “Due” tab (gid: `1002294955`)  
   - Columns to map:  
     - Url → `{{$json.url}}`  
     - Vendor → `{{$json.Vendor}}`  
     - Invoice Date → `{{$json['Invoice Date']}}`  
     - Total → `{{$json.Total}}`  
     - Due Date → `{{$json['Due Date']}}`  
   - Matching Columns: Use `Url` to prevent duplicates  
   - Credentials: Configure Google Sheets OAuth2 credentials for the account with access to the spreadsheet.  
   - Connect output from `Set Fields`.

9. **Save and test the workflow:**  
   - Trigger manually and verify invoices in the specified Google Drive folder are parsed and data appears in the Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup instructions cover how to connect Google Drive, PDF.co, and Google Sheets with OAuth2 credentials and API keys.            | Sticky Note2 near nodes; critical for credential setup                                             |
| Workflow purpose: Automated extraction of invoice data from Drive PDFs to Sheets using AI.                                         | Sticky Note55                                                                                       |
| Contact info for customization or support: Robert Breen, robert@ynteractive.com, [LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/), [ynteractive.com](https://ynteractive.com) | Included in Sticky Note2 for user assistance                                                       |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.