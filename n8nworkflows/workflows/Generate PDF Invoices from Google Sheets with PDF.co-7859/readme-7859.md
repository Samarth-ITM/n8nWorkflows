Generate PDF Invoices from Google Sheets with PDF.co

https://n8nworkflows.xyz/workflows/generate-pdf-invoices-from-google-sheets-with-pdf-co-7859


# Generate PDF Invoices from Google Sheets with PDF.co

---

### 1. Workflow Overview

This workflow automates the generation of professional PDF invoices from invoice data stored in a Google Sheets spreadsheet. It is ideal for small businesses or freelancers who manage invoice line items in Google Sheets and want to produce polished, ready-to-send PDF invoices using PDF.co's HTML template-to-PDF conversion.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger and Data Retrieval:** Manually triggered workflow that fetches invoice rows from a specified Google Sheets spreadsheet.
- **1.2 Data Transformation:** Processes and normalizes the raw Google Sheets data into a structured JSON format tailored for the PDF.co Mustache HTML template.
- **1.3 PDF Generation:** Converts the structured JSON data into a PDF invoice by invoking PDF.co’s HTML Template to PDF operation using a predefined template.

Supporting the workflow, several sticky notes provide setup instructions, configuration tips, and useful links.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and Data Retrieval

- **Overview:**  
  Initiates the workflow manually and retrieves invoice rows from a configured Google Sheets file.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Invoice Rows (Google Sheets)

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Starts the workflow on user command, no external event dependency.  
     - Configuration: Default, no parameters.  
     - Inputs: None  
     - Outputs: Triggers the next node “Get Invoice Rows”.  
     - Failure Modes: None typical; manual trigger unlikely to fail except UI errors.

  2. **Get Invoice Rows**  
     - Type: Google Sheets (OAuth2)  
     - Role: Reads all invoice line item rows from the specified Google Sheets document and sheet.  
     - Configuration:  
       - Spreadsheet ID: Linked to a Google Sheet titled “Invoice Template”.  
       - Sheet Name: “Sheet1” (gid=0)  
       - Credentials: Google Sheets OAuth2 account configured with user’s Google account access.  
     - Inputs: Triggered by manual trigger node.  
     - Outputs: Emits rows as items downstream.  
     - Edge Cases:  
       - Authentication errors if OAuth token expired or revoked.  
       - Empty sheet or malformed data returns empty or unexpected results.  
       - Permission errors if spreadsheet access is removed.  
     - Version: 4.7 (Google Sheets node version)

#### 1.2 Data Transformation

- **Overview:**  
  Processes the raw rows from Google Sheets, normalizes fields, computes totals and invoice metadata, and structures data for PDF generation.

- **Nodes Involved:**  
  - Convert to html import (Code node)

- **Node Details:**

  1. **Convert to html import**  
     - Type: Code (JavaScript)  
     - Role: Transforms spreadsheet row data into a single JSON object with all invoice details formatted for the PDF.co Mustache template.  
     - Configuration Highlights:  
       - Parses input rows, supporting both array of rows or individual items.  
       - Normalizes field names (case-insensitive) for Description, Quantity, Price, Total, Company, Address, Email.  
       - Filters out invalid rows (missing description or non-positive qty/price).  
       - Calculates line totals, subtotal, tax (default 0%), discount (default 0), grand total.  
       - Generates invoice metadata: unique invoice number with prefix “INV-2025-”, invoice date (today), due date (+14 days).  
       - Defines static company info (name, address, phone, email, logo URL).  
       - Outputs a single item with root-level keys for Mustache template consumption.  
     - Key Expressions/Variables:  
       - `INVOICE_PREFIX`, `TAX_RATE`, `DISCOUNT`, `TERMS`, `NOTES` constants.  
       - Helper functions: `fmtUSD` (formats currency), `toNum` (parses number).  
       - Output JSON object with keys like `invoiceNumber`, `company`, `billTo`, `items`, `subTotalFmt`, `taxAmountFmt`, `totalFmt`.  
     - Inputs: Receives rows from “Get Invoice Rows” node.  
     - Outputs: Single item JSON object for PDF generation.  
     - Edge Cases:  
       - Malformed or missing data fields cause filtering out rows.  
       - If no valid rows, invoice will have empty items and default billTo values.  
       - Random invoice number generation could theoretically collide (low risk).  
     - Version: 2 (Code node version)

#### 1.3 PDF Generation

- **Overview:**  
  Converts the processed JSON invoice data into a PDF file by invoking PDF.co’s HTML Template to PDF API.

- **Nodes Involved:**  
  - Create PDF (PDF.co API node)

- **Node Details:**

  1. **Create PDF**  
     - Type: PDF.co API  
     - Role: Calls PDF.co’s service to convert an HTML template populated with the invoice data into a PDF document.  
     - Configuration:  
       - Operation: “URL/HTML to PDF” using “HTML Template to PDF” conversion.  
       - Template ID: “12010” (placeholder ID, user to replace with their own template ID from PDF.co dashboard).  
       - Template Data: Passes JSON stringified invoice data from prior node.  
       - Credentials: PDF.co API key credential configured in n8n.  
     - Inputs: Single JSON object from “Convert to html import”.  
     - Outputs: PDF file or a URL to the generated PDF (depending on PDF.co response).  
     - Edge Cases:  
       - API key invalid or expired causing authorization errors.  
       - Template ID invalid or missing.  
       - Network timeouts or API rate limits.  
       - Malformed JSON input causing template rendering errors.  
     - Version: 1 (PDF.co node version)

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                          | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                      |
|-------------------------------|-------------------------|----------------------------------------|-----------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Initiates workflow manually             | —                           | Get Invoice Rows          |                                                                                                |
| Get Invoice Rows              | Google Sheets           | Reads invoice rows from Google Sheets   | When clicking ‘Execute workflow’ | Convert to html import    | See setup instructions for Google Sheets in Sticky Note at position [-1088,-416]               |
| Convert to html import        | Code (JavaScript)       | Normalizes and structures invoice data  | Get Invoice Rows             | Create PDF                |                                                                                                |
| Create PDF                   | PDF.co API              | Generates PDF invoice from template     | Convert to html import       | —                        | See setup instructions for PDF.co in Sticky Notes at positions [-384,64] and [256,-256]        |
| Sticky Note                  | Sticky Note             | Setup instructions and workflow info    | —                           | —                        | Multiple notes providing setup guidance for Google Sheets and PDF.co API                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Add a “Manual Trigger” node named `When clicking ‘Execute workflow’`.  
   - No special configuration needed.

2. **Create a Google Sheets Node to Retrieve Invoice Rows**  
   - Add a “Google Sheets” node named `Get Invoice Rows`.  
   - Configure Credentials: Set up a Google Sheets OAuth2 credential with access to your Google Drive.  
   - Set the Spreadsheet ID to your copied Invoice Template spreadsheet ID.  
   - Set the Sheet Name to “Sheet1” or your relevant sheet (gid=0).  
   - Connect the output of `When clicking ‘Execute workflow’` to the input of `Get Invoice Rows`.

3. **Add a Code Node to Transform Data**  
   - Add a “Code” node named `Convert to html import`.  
   - Paste the provided JavaScript code (see detailed code logic in Section 2.2).  
   - No credentials required.  
   - Connect the output of `Get Invoice Rows` to the input of this Code node.  
   - This node expects input as multiple row items or a single array item and outputs one JSON object containing invoice metadata and line items.

4. **Add and Configure PDF.co Node to Create PDF**  
   - Add a `PDF.co API` node named `Create PDF`.  
   - Configure Credentials: Create and set up a PDF.co API key credential.  
   - Set Operation to “URL/HTML to PDF” with sub-operation “HTML Template to PDF”.  
   - Set the Template ID to your own PDF.co HTML invoice template ID (replace default `12010`).  
   - For Template Data, bind to the JSON stringified output of the previous Code node (`={{ JSON.stringify($json) }}`).  
   - Connect the output of `Convert to html import` to this node.

5. **Final Connections**  
   - Ensure nodes are connected in this sequence:  
     `When clicking ‘Execute workflow’` → `Get Invoice Rows` → `Convert to html import` → `Create PDF`.

6. **Setup Credentials**  
   - Google Sheets OAuth2: Authenticate with your Google account and grant access to your invoice spreadsheet.  
   - PDF.co API: Register for a free PDF.co account, copy API key, and add it in n8n credentials.

7. **HTML Template Setup on PDF.co**  
   - Log into PDF.co dashboard.  
   - Navigate to Templates → New Template.  
   - Create a new HTML template using your desired invoice design, referencing the JSON keys output by the Code node.  
   - Save and copy the Template ID.  
   - Replace the `templateId` field in the PDF.co node with your new Template ID.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                 | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automatically pulls invoice rows from Google Sheets and generates a PDF invoice using a PDF.co template. Ideal for small businesses managing invoices in Sheets needing professional PDFs.                                                                                     | Sticky Note at position [-640, -416]                                                             |
| Setup Instructions: 1. Copy the provided Invoice Template Sheet into your Google Drive. 2. Configure Google Sheets OAuth2 credentials in n8n. 3. Create a PDF.co account and configure API key credentials. 4. Create and use a custom PDF.co HTML template.                                   | Sticky Note at position [-1088, -416] with detailed step-by-step setup instructions and links.   |
| Need help customizing this workflow (e.g., filtering by sender, auto-reply, Slack notifications)? Contact: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/, Website: https://ynteractive.com                                                                 | Contact info included in Sticky Note at [-1088, -416]                                           |
| PDF.co HTML Template creation guide: Create templates in the PDF.co dashboard under Templates → New Template; paste your HTML code; save and get Template ID for use in the workflow.                                                                                                        | Sticky Note at position [256, -256]                                                              |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---