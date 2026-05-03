Generate Invoices, Save to Drive and Send Email to Customer with JS + G Sheets

https://n8nworkflows.xyz/workflows/generate-invoices--save-to-drive-and-send-email-to-customer-with-js---g-sheets-4105


# Generate Invoices, Save to Drive and Send Email to Customer with JS + G Sheets

---

### 1. Workflow Overview

This workflow automates the entire process of generating invoices from order form submissions. It ensures that each invoice has a unique order ID, creates a formatted HTML invoice, converts it into a PDF, stores the PDF in Google Drive, sends the invoice via email to the customer, and logs the invoice data into a Google Sheet for record-keeping.

Logical blocks grouped by function and dependency:

- **1.1 Input Reception**: Captures incoming order details via webhook simulation.
- **1.2 Order ID Generation and Validation**: Generates a unique invoice ID and checks for duplicates in the Google Sheet.
- **1.3 Invoice Data Preparation**: Formats invoice details into HTML.
- **1.4 PDF Conversion and Storage**: Converts HTML to PDF and uploads it to Google Drive.
- **1.5 Customer Notification and Logging**: Emails the invoice to the customer and logs invoice metadata into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block receives order form data, simulating an HTTP webhook trigger with customer and order details.
- **Nodes Involved:**  
  - Webhook Simulator  
  - Sticky Note (documentation aid)

- **Node Details:**

  - **Webhook Simulator**  
    - *Type & Role:* Webhook node (simulated) capturing incoming order form data.  
    - *Configuration:* Path set to a unique identifier; receives JSON with customer name, email, products (array with name, price, quantity), and total.  
    - *Expressions:* Data accessed downstream from node JSON output.  
    - *Connections:* Outputs to "Generate Invoice ID" node.  
    - *Edge Cases:* Simulated webhook; in production, replace with real webhook URL. Possible failure if data structure changes or missing fields.

  - **Sticky Note**  
    - *Purpose:* Contains pinned example data to simulate webhook input for testing.

---

#### 2.2 Order ID Generation and Validation

- **Overview:** Generates a unique invoice order ID, checks Google Sheets for duplicates, and loops to regenerate if the ID exists.
- **Nodes Involved:**  
  - Generate Invoice ID (Code)  
  - Check if ID Already Exists (Google Sheets)  
  - If Does not Exist (IF Node)

- **Node Details:**

  - **Generate Invoice ID**  
    - *Type & Role:* Code node generating a random alphanumeric invoice ID prefixed with "INV-".  
    - *Configuration:* JavaScript function generates a 6-character string from uppercase letters and digits, concatenated with "INV-".  
    - *Key Expressions:* Output JSON contains `order_id`.  
    - *Connections:* Feeds into Google Sheets lookup node.  
    - *Edge Cases:* Random collisions possible but unlikely; mitigated by lookup and loop.

  - **Check if ID Already Exists**  
    - *Type & Role:* Google Sheets node for searching an existing order ID in the invoice log.  
    - *Configuration:* Looks up "Invoice ID" column in the Google Sheet "Uppfy Shop Invoices", sheet named "Invoices".  
    - *Credentials:* Google Sheets OAuth2 required.  
    - *Input:* `order_id` from previous node.  
    - *Output:* Returns rows matching order ID; if empty, ID is unique.  
    - *Edge Cases:* API failures, credential expiration, sheet renaming or schema changes.

  - **If Does not Exist**  
    - *Type & Role:* IF node evaluating if the `row_number` field in the Google Sheets result is empty (meaning no duplicate found).  
    - *Configuration:* Condition checks if `row_number` is empty (string empty).  
    - *Connections:*  
      - True branch: proceeds to prepare invoice data ("Set Fields").  
      - False branch: loops back to "Generate Invoice ID" to create a new ID if duplicate found.  
    - *Edge Cases:* Expression syntax errors, edge cases if sheet returns unexpected data structure.

---

#### 2.3 Invoice Data Preparation

- **Overview:** Prepares and formats all invoice data into structured JSON and then generates a styled HTML invoice.
- **Nodes Involved:**  
  - Set Fields (Set Node)  
  - Create Invoice HTML (Code Node)

- **Node Details:**

  - **Set Fields**  
    - *Type & Role:* Set node to consolidate and normalize invoice-related variables for downstream use.  
    - *Configuration:* Maps and copies data fields: `order_id`, `customer_name`, `customer_email`, `products` (array), and `total` from previous nodes.  
    - *Expressions:* Uses expressions to pull data from "Generate Invoice ID" and "Webhook Simulator" nodes.  
    - *Connections:* Outputs to "Create Invoice HTML" node.  
    - *Edge Cases:* Missing or malformed input data could cause empty or invalid fields.

  - **Create Invoice HTML**  
    - *Type & Role:* Code node generating a full HTML invoice string with inline styles, based on the consolidated data.  
    - *Configuration:*  
      - Iterates over `products` array to create rows with product name, quantity, unit price, subtotal.  
      - Outputs a full HTML document with customer info and a table of items and totals.  
    - *Key Expressions:* JavaScript variables destructure input JSON; uses template literals for HTML.  
    - *Connections:* Outputs HTML string to "HTML to PDF" HTTP Request node.  
    - *Edge Cases:* Products array empty or malformed could break table rendering; ensure numeric fields are valid numbers.

---

#### 2.4 PDF Conversion and Storage

- **Overview:** Converts generated HTML invoice into a PDF via an external API, then uploads the PDF to Google Drive.
- **Nodes Involved:**  
  - HTML to PDF (HTTP Request)  
  - Download PDF from API (HTTP Request)  
  - Upload PDF to GDrive (Google Drive)

- **Node Details:**

  - **HTML to PDF**  
    - *Type & Role:* HTTP Request node calling RapidAPI's HTML to PDF conversion service.  
    - *Configuration:*  
      - POST method to `https://html2pdf2.p.rapidapi.com/html2pdf`  
      - Sends invoice HTML as a body parameter.  
      - Includes required headers: `x-rapidapi-host` and `x-rapidapi-key` (API key placeholder).  
    - *Output:* Returns response likely containing a PDF URL or file blob.  
    - *Edge Cases:* API key missing/expired, rate limiting, malformed HTML causing conversion errors.

  - **Download PDF from API**  
    - *Type & Role:* HTTP Request node used to download the PDF file from a static URL (note: in this workflow, it points to a fixed S3 URL, likely placeholder or debug step).  
    - *Configuration:* GET request to a hardcoded S3 PDF URL.  
    - *Connections:* Outputs binary PDF data to "Upload PDF to GDrive".  
    - *Edge Cases:* URL may expire, or be a placeholder; in production, should dynamically use URL from previous step.

  - **Upload PDF to GDrive**  
    - *Type & Role:* Google Drive node to upload the PDF file blob.  
    - *Configuration:*  
      - File name set as `order_id - customer_name`.  
      - Uploads to Google Drive root folder or specified folder.  
    - *Credentials:* Google Drive OAuth2 required.  
    - *Input:* Binary PDF data from previous node.  
    - *Output:* Provides Google Drive file metadata including `webViewLink`.  
    - *Edge Cases:* Credential expiry, insufficient permissions, file size limits.

---

#### 2.5 Customer Notification and Logging

- **Overview:** Sends the invoice PDF via email to the customer and logs invoice details into Google Sheets for record keeping.
- **Nodes Involved:**  
  - Email Invoice to Customer (Email Send)  
  - Append Details to Invoices Sheet (Google Sheets Append/Update)

- **Node Details:**

  - **Email Invoice to Customer**  
    - *Type & Role:* Email Send node dispatching the invoice email with the PDF attached.  
    - *Configuration:*  
      - To: customer email from Set Fields node.  
      - From: configured sender email (e.g., "Uppfy Shop <your_from_email>").  
      - Subject: includes `order_id`.  
      - Body: includes a greeting, the HTML invoice, and a styled "Pay Now" button with placeholder payment link.  
      - Attachment: references the PDF uploaded to Drive (implicit or explicit depending on configuration).  
    - *Credentials:* SMTP configured for sending email.  
    - *Edge Cases:* SMTP auth failures, invalid email addresses, large attachments causing send failures.

  - **Append Details to Invoices Sheet**  
    - *Type & Role:* Google Sheets node appending or updating invoice metadata row.  
    - *Configuration:*  
      - Columns: Status ("Unpaid"), Invoice ID, Invoice URL (Google Drive link), Invoice Amount, Customer Name, Customer Email.  
      - Operation: Append or update based on Invoice ID.  
      - Sheet: "Invoices" sheet in the same Google Sheet document.  
    - *Credentials:* Google Sheets OAuth2.  
    - *Edge Cases:* Sheet access errors, data format mismatches, duplicate rows if matching fails.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                      | Input Node(s)              | Output Node(s)                 | Sticky Note                                                      |
|----------------------------|---------------------|------------------------------------|----------------------------|-------------------------------|-----------------------------------------------------------------|
| Webhook Simulator          | Webhook             | Receive order form data             | —                          | Generate Invoice ID            | Contains pinned sample data to simulate webhook input           |
| Generate Invoice ID        | Code                | Generate unique order ID            | Webhook Simulator          | Check if ID Already Exists     |                                                                 |
| Check if ID Already Exists | Google Sheets       | Lookup order ID to avoid duplicates| Generate Invoice ID        | If Does not Exist              |                                                                 |
| If Does not Exist          | IF                  | Conditional check on ID existence  | Check if ID Already Exists | Set Fields / Generate Invoice ID (loop) |                                                                 |
| Set Fields                 | Set                 | Prepare invoice data variables     | If Does not Exist          | Create Invoice HTML            |                                                                 |
| Create Invoice HTML        | Code                | Generate HTML invoice content      | Set Fields                 | HTML to PDF                   |                                                                 |
| HTML to PDF                | HTTP Request        | Convert HTML to PDF via API        | Create Invoice HTML        | Download PDF from API          | API URL for RapidAPI HTML to PDF service [see doc link]         |
| Download PDF from API      | HTTP Request        | Download PDF binary data           | HTML to PDF                | Upload PDF to GDrive           |                                                                 |
| Upload PDF to GDrive       | Google Drive        | Upload PDF file to Google Drive    | Download PDF from API      | Email Invoice to Customer      |                                                                 |
| Email Invoice to Customer  | Email Send          | Email invoice to customer          | Upload PDF to GDrive       | Append Details to Invoices Sheet |                                                                 |
| Append Details to Invoices Sheet | Google Sheets | Log invoice metadata               | Email Invoice to Customer  | —                             |                                                                 |
| Sticky Note                | Sticky Note         | Documentation aid                  | —                          | —                             | Contains pinned example data to simulate webhook input           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: e.g., `7f884a07-861e-49da-bb0e-1ac6ca40ea00`  
   - Purpose: Receive incoming order data (customer name, email, products array, total)  

2. **Add Code Node "Generate Invoice ID"**  
   - Type: Code (JavaScript)  
   - Code: Generate a random 6-character string prefixed with "INV-" from uppercase alphanumeric characters.  
   - Output: JSON field `order_id`  
   - Connect from Webhook node  

3. **Add Google Sheets Node "Check if ID Already Exists"**  
   - Operation: Lookup rows in Google Sheets  
   - Document ID: Your invoice log Google Sheet ID  
   - Sheet Name: "Invoices" (or your invoice log sheet)  
   - Lookup Column: "Invoice ID"  
   - Lookup Value: `={{ $json.order_id }}` from previous node  
   - Credentials: Google Sheets OAuth2  
   - Connect from "Generate Invoice ID" node  

4. **Add IF Node "If Does not Exist"**  
   - Condition: Check if `row_number` field from Google Sheets is empty (meaning no existing order ID)  
   - True branch: proceed to "Set Fields" node  
   - False branch: connect back to "Generate Invoice ID" to regenerate the ID (loop)  

5. **Add Set Node "Set Fields"**  
   - Assign variables:  
     - `order_id` from "Generate Invoice ID" node  
     - `customer_name`, `customer_email`, `products`, `total` from Webhook node  
   - Connect from IF node (true branch)  

6. **Add Code Node "Create Invoice HTML"**  
   - Input: JSON from "Set Fields"  
   - Code: Generate full HTML invoice string including customer info, order ID, products table, and total with inline CSS for styling  
   - Connect from "Set Fields"  

7. **Add HTTP Request Node "HTML to PDF"**  
   - Method: POST  
   - URL: `https://html2pdf2.p.rapidapi.com/html2pdf`  
   - Headers:  
     - `x-rapidapi-host: html2pdf2.p.rapidapi.com`  
     - `x-rapidapi-key: your_rapid_api_key` (replace with your actual key)  
   - Body Parameter: `html` set to `={{ $json.html }}` from previous node  
   - Connect from "Create Invoice HTML"  

8. **Add HTTP Request Node "Download PDF from API"**  
   - Method: GET  
   - URL: Set dynamically from previous HTTP Request node response if available (PDF file URL)  
   - Connect from "HTML to PDF"  

9. **Add Google Drive Node "Upload PDF to GDrive"**  
   - Operation: Upload file  
   - File name: `={{ $json.order_id }} - {{ $json.customer_name }}`  
   - File data: Use binary data from "Download PDF from API"  
   - Folder: Select desired Google Drive folder (e.g., root)  
   - Credentials: Google Drive OAuth2  
   - Connect from "Download PDF from API"  

10. **Add Email Send Node "Email Invoice to Customer"**  
    - To: `={{ $json.customer_email }}` from "Set Fields"  
    - From: Your sender email configured in SMTP credentials  
    - Subject: `={{ $json.order_id }} - New Invoice from Uppfy Shop`  
    - Body: Include HTML invoice and a "Pay Now" button with a payment link placeholder  
    - Attach: PDF file from "Upload PDF to GDrive" (or attach binary directly if available)  
    - Credentials: SMTP account  
    - Connect from "Upload PDF to GDrive"  

11. **Add Google Sheets Node "Append Details to Invoices Sheet"**  
    - Operation: Append or update row  
    - Document ID & Sheet Name: Same as "Check if ID Already Exists"  
    - Columns to populate:  
      - Status: "Unpaid"  
      - Invoice ID: `={{ $json.order_id }}`  
      - Invoice URL: `={{ $json.webViewLink }}` from Google Drive upload  
      - Invoice Amount: `={{ $json.total }}`  
      - Customer Name: `={{ $json.customer_name }}`  
      - Customer Email: `={{ $json.customer_email }}`  
    - Credentials: Google Sheets OAuth2  
    - Connect from "Email Invoice to Customer"  

12. (Optional) **Add Sticky Note** for documentation and sample data simulation.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Customize the invoice HTML template in "Create Invoice HTML" node to match your branding and style. | Adjust inline CSS and HTML structure within the code node as needed.                                                     |
| Obtain RapidAPI key for HTML to PDF conversion at: https://rapidapi.com/rhodium/api/html2pdf2        | Required for the HTTP Request node converting HTML invoices to PDF.                                                      |
| Google Sheets invoice log template available here: https://docs.google.com/spreadsheets/d/1QW_Lg1aoEBku8GaxwPbZfBY5ITAuSdvoAXRyNdCEujM/edit?gid=0 | Use or customize this sheet for logging invoice data like order IDs and PDF links.                                        |
| Ensure Google Drive and SMTP credentials are properly configured in n8n before running this workflow.| Credential setup is mandatory for file upload and email sending nodes.                                                   |
| Expand workflow potential by adding payment processing, SMS notifications, or error handling nodes.  | Modular design allows easy extension with additional integrations and alerts.                                            |
| For support or custom workflows, contact: joseph@uppfy.com                                          | Direct contact for professional assistance or bespoke automation solutions.                                              |

---

**Disclaimer:**  
The provided workflow text and details originate solely from an automated n8n workflow. The content complies strictly with relevant content policies and contains no illegal or protected elements. All manipulated data is legal and publicly accessible.

---