Custom Branded QuickBooks Invoices to PDF with Gotenberg & Email

https://n8nworkflows.xyz/workflows/custom-branded-quickbooks-invoices-to-pdf-with-gotenberg---email-7051


# Custom Branded QuickBooks Invoices to PDF with Gotenberg & Email

### 1. Workflow Overview

This workflow automates the creation of fully branded PDF invoices from QuickBooks Online (QBO) invoice data and emails the generated PDF directly to the customer. It triggers automatically when a new invoice is created in QuickBooks, fetches all relevant invoice details, company branding assets (logo and signature), and combines them into a professionally designed multi-page-aware HTML invoice. The HTML is then converted to a PDF using the Gotenberg API, and the PDF is emailed to the customer.

**Target Use Cases:**  
- Businesses using QuickBooks Online who want automated, custom-branded invoices sent instantly upon creation.  
- Users requiring invoices with a modern design, multi-page support, and embedded branding elements like logos and signatures.  
- Automations integrating QBO with PDF generation and outbound email delivery.

**Logical Blocks:**

- **1.1 Input Reception:** Receive webhook notification when a new invoice is created in QuickBooks.  
- **1.2 Data Fetching:** Retrieve detailed invoice data from QuickBooks, and fetch company logo and signature images from external URLs.  
- **1.3 Data Preparation:** Convert images to Base64, merge all data, and prepare a structured data object with formatted fields and conditional HTML elements.  
- **1.4 HTML Invoice Generation:** Build the HTML invoice template dynamically using the prepared data.  
- **1.5 PDF Generation:** Convert the HTML invoice to a PDF file via Gotenberg API.  
- **1.6 Email Dispatch:** Send the generated PDF as an email attachment to the customer with a styled email body.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming webhook calls from QuickBooks Online when a new invoice is created. It triggers the workflow to start the invoice processing.

**Nodes Involved:**  
- Listen for New QuickBooks Invoice

**Node Details:**  

- **Listen for New QuickBooks Invoice**  
  - Type: Webhook  
  - Role: Entry point for the workflow; receives POST requests from QuickBooks webhook on new invoice events.  
  - Configuration:  
    - HTTP Method: POST  
    - Path: `qbo-invoice-webhook`  
  - Key expressions: Uses webhook payload to extract invoice ID in downstream nodes: `{{$json.body.eventNotifications[0].dataChangeEvent.entities[0].id}}`  
  - Input: External webhook call from QuickBooks  
  - Output: JSON payload containing event data  
  - Potential Failures: Missing or invalid webhook call, malformed payload, unauthorized QuickBooks requests if webhook is misconfigured.  
  - Notes: Requires QuickBooks webhook configuration to point to this node’s Production URL.

---

#### 1.2 Data Fetching

**Overview:**  
This block fetches detailed invoice data from QuickBooks using the invoice ID from the webhook and retrieves company branding assets (logo and signature) from external URLs.

**Nodes Involved:**  
- Get Invoice Data from QuickBooks  
- Fetch Company Logo Image  
- Fetch Company Signature Image

**Node Details:**  

- **Get Invoice Data from QuickBooks**  
  - Type: QuickBooks node  
  - Role: Fetch invoice details via QuickBooks API using invoice ID  
  - Configuration:  
    - Resource: Invoice  
    - Invoice ID: Expression extracting from webhook payload `={{ $json.body.eventNotifications[0].dataChangeEvent.entities[0].id }}`  
  - Credentials: QuickBooks OAuth2 credentials required  
  - Input: Invoice ID from webhook node  
  - Output: Full invoice JSON data including line items, customer info, payment terms, balances, etc.  
  - Potential Failures: Authentication errors, invalid invoice ID, API rate limits, network errors.

- **Fetch Company Logo Image**  
  - Type: HTTP Request  
  - Role: Download company logo image file from a public URL  
  - Configuration:  
    - URL: Public direct link to logo image (PNG expected)  
    - Response Format: File (binary)  
  - Input: Triggered in parallel from webhook node  
  - Output: Binary file data under property `logodata`  
  - Potential Failures: Invalid or inaccessible URL, HTTP errors, unsupported image format.

- **Fetch Company Signature Image**  
  - Type: HTTP Request  
  - Role: Download company signature image file from a public URL  
  - Configuration:  
    - URL: Public direct link to signature image (PNG expected)  
    - Response Format: File (binary)  
  - Input: Triggered in parallel from webhook node  
  - Output: Binary file data under property `signaturedata`  
  - Potential Failures: Similar to logo image fetch node.

---

#### 1.3 Data Preparation

**Overview:**  
This block converts the binary images to Base64 strings, merges invoice data with branding assets, and prepares a structured JSON object with formatted fields and conditional HTML for later rendering.

**Nodes Involved:**  
- Convert Logo to Base64  
- Convert Signature to Base64  
- Combine Invoice, Logo & Signature  
- Prepare All Data for Template

**Node Details:**  

- **Convert Logo to Base64**  
  - Type: Extract From File  
  - Role: Convert binary logo image to Base64 string stored in JSON property `logodata`  
  - Configuration:  
    - Operation: binaryToProperty  
    - Binary Property: `logodata`  
    - Destination Key: `logodata`  
  - Input: Binary file from Fetch Company Logo Image  
  - Output: JSON with Base64 string for logo image  
  - Potential Failures: Missing binary data, conversion errors.

- **Convert Signature to Base64**  
  - Type: Extract From File  
  - Role: Convert binary signature image to Base64 string stored in JSON property `signaturedata`  
  - Configuration: Same as logo conversion but for `signaturedata`  
  - Input: Binary file from Fetch Company Signature Image  
  - Output: JSON with Base64 string for signature image  
  - Potential Failures: Same as logo conversion node.

- **Combine Invoice, Logo & Signature**  
  - Type: Merge  
  - Role: Combine JSON outputs from invoice data, logo Base64, and signature Base64 into one data object for unified processing  
  - Configuration:  
    - Number Inputs: 3 (invoice data, logo Base64, signature Base64)  
    - Merge Mode: Append (default)  
  - Input:  
    - From Get Invoice Data node  
    - From Convert Logo to Base64  
    - From Convert Signature to Base64  
  - Output: Combined JSON with all relevant data  
  - Potential Failures: Missing inputs, merge conflicts.

- **Prepare All Data for Template**  
  - Type: Code (JavaScript)  
  - Role: Final data shaping and formatting before HTML rendering  
  - Key Logic:  
    - Extract invoice, logo, and signature data from merged inputs  
    - Validate presence of Base64 image data  
    - Create full Data URI strings for images (`data:image/png;base64,...`)  
    - Format invoice dates to readable US format (e.g., "January 31, 2024")  
    - Build HTML rows for each invoice line item with subtle styling on descriptions  
    - Conditionally generate a "PAID" stamp if balance is zero or less  
    - Compile all these into a final JSON object returned for the template  
  - Input: Combined data from Merge node  
  - Output: JSON with all fields required by the HTML template, including prepared HTML snippets  
  - Potential Failures: Missing fields, date parsing errors, JavaScript exceptions.

---

#### 1.4 HTML Invoice Generation

**Overview:**  
This block creates the full HTML invoice document using a premium designed template and the prepared data JSON.

**Nodes Involved:**  
- Build HTML Invoice from Data

**Node Details:**  

- **Build HTML Invoice from Data**  
  - Type: HTML Node  
  - Role: Render a professional, multi-page capable invoice as HTML using data from the previous node  
  - Configuration:  
    - HTML code includes embedded CSS for print styling, fonts, multi-column layout, page numbers, headers, footers, tables, and brand colors  
    - Uses expressions to inject JSON data (e.g., `{{$json.invoiceNumber}}`, `{{$json.logoSrc}}`, `{{$json.lineItems}}`)  
    - Supports multi-page with CSS page-break controls and fixed footer with page counters  
  - Input: JSON data from Prepare All Data for Template node  
  - Output: HTML string with full invoice content  
  - Potential Failures: Malformed HTML, missing data causing empty placeholders.

---

#### 1.5 PDF Generation

**Overview:**  
This block converts the generated invoice HTML into a PDF file using the Gotenberg API.

**Nodes Involved:**  
- Convert HTML to Binary File  
- Generate PDF via Gotenberg

**Node Details:**  

- **Convert HTML to Binary File**  
  - Type: Convert To File  
  - Role: Convert the HTML string into a binary file object to send to Gotenberg  
  - Configuration:  
    - Operation: toText  
    - Encoding: UTF-8  
    - File Name: `index.html`  
    - Source Property: `html` (from previous node)  
  - Input: HTML string from Build HTML Invoice from Data  
  - Output: Binary file format suitable for multipart upload  
  - Potential Failures: Encoding issues, missing HTML input.

- **Generate PDF via Gotenberg**  
  - Type: HTTP Request  
  - Role: Send multipart form-data POST request to Gotenberg API to convert HTML to PDF  
  - Configuration:  
    - URL: User must set to their running Gotenberg instance URL, e.g., `http://yourGotenBergInstanceURL/forms/chromium/convert/html`  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Body Parameters:  
      - `files`: binary file from previous node  
      - `scale`: 1 (default zoom scale)  
      - `metadata`: JSON string with PDF metadata (author, creator, dates) dynamically inserted with date expressions  
    - Headers: Include `Gotenberg-Output-Filename` set to invoice number for PDF naming  
  - Input: Binary file from Convert HTML to Binary File  
  - Output: PDF binary file stream  
  - Potential Failures: Connectivity to Gotenberg, invalid URL, file upload errors, response errors.

---

#### 1.6 Email Dispatch

**Overview:**  
This block sends the generated PDF invoice as an email attachment to the customer’s email address with a clean, branded email body.

**Nodes Involved:**  
- Email PDF Invoice to Customer

**Node Details:**  

- **Email PDF Invoice to Customer**  
  - Type: Email Send  
  - Role: Send email with PDF attachment to customer  
  - Configuration:  
    - From Email: Dynamically set from customer email in prepared data (can be customized)  
    - To Email: Hardcoded to `company.ebtech@gmail.com` (likely intended recipient or for testing; can be customized to customer email)  
    - Subject: Includes invoice number dynamically  
    - Email Body: Rich HTML with brand styling, personalized greeting by customer name, invoice summary, contact info, and footer  
    - Attachments: The PDF file binary from Gotenberg node  
  - Credentials: SMTP credentials required  
  - Input: PDF file from Generate PDF node, customer and invoice data from Prepare All Data for Template node  
  - Output: Email sent confirmation  
  - Potential Failures: SMTP authentication errors, invalid email addresses, attachment size limits, network issues.

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                               | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                                      |
|--------------------------------|--------------------|-----------------------------------------------|----------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Listen for New QuickBooks Invoice | Webhook            | Entry trigger on new QuickBooks invoice       | External webhook call             | Get Invoice Data from QuickBooks, Fetch Company Logo Image, Fetch Company Signature Image | **1. Configure Webhook** Copy this node's Production URL and paste it into QuickBooks webhook settings.          |
| Get Invoice Data from QuickBooks | QuickBooks         | Fetch detailed invoice data from QBO          | Listen for New QuickBooks Invoice | Combine Invoice, Logo & Signature | **2. Connect QuickBooks** Select your QuickBooks Online account credentials.                                     |
| Fetch Company Logo Image         | HTTP Request       | Download company logo image                    | Listen for New QuickBooks Invoice | Convert Logo to Base64           | **3. Add Your Logo URL** Replace placeholder URL with your company's logo URL.                                  |
| Fetch Company Signature Image    | HTTP Request       | Download company signature image               | Listen for New QuickBooks Invoice | Convert Signature to Base64      | **4. Add Your Signature URL** Replace placeholder URL with your signature image URL.                            |
| Convert Logo to Base64           | Extract From File  | Convert logo binary image to Base64 string     | Fetch Company Logo Image          | Combine Invoice, Logo & Signature |                                                                                                                 |
| Convert Signature to Base64      | Extract From File  | Convert signature binary image to Base64 string| Fetch Company Signature Image     | Combine Invoice, Logo & Signature |                                                                                                                 |
| Combine Invoice, Logo & Signature| Merge              | Combine invoice data and branding Base64 images| Get Invoice Data, Convert Logo, Convert Signature | Prepare All Data for Template |                                                                                                                 |
| Prepare All Data for Template    | Code (JavaScript)  | Format and prepare final data for invoice HTML| Combine Invoice, Logo & Signature | Build HTML Invoice from Data     |                                                                                                                 |
| Build HTML Invoice from Data     | HTML               | Generate full invoice HTML document             | Prepare All Data for Template     | Convert HTML to Binary File      |                                                                                                                 |
| Convert HTML to Binary File      | Convert To File    | Convert HTML string to binary file              | Build HTML Invoice from Data      | Generate PDF via Gotenberg       |                                                                                                                 |
| Generate PDF via Gotenberg       | HTTP Request       | Convert HTML binary to PDF via Gotenberg API   | Convert HTML to Binary File       | Email PDF Invoice to Customer    | **5. Set Your Gotenberg URL** Replace placeholder URL with your running Gotenberg instance URL.                  |
| Email PDF Invoice to Customer    | Email Send         | Send email with PDF invoice attached            | Generate PDF via Gotenberg        | None                           | **6. Configure Your Email** Select SMTP credentials, customize From, Subject, and email body.                   |
| Sticky Note                     | Sticky Note        | Workflow overview and setup instructions        | None                            | None                           | See detailed guidance and setup instructions for the entire workflow and nodes.                                 |
| Sticky Note1                    | Sticky Note        | Explains webhook setup                           | None                            | None                           | **1. Configure Webhook** Copy this node's Production URL and paste it into QuickBooks Webhook.                   |
| Sticky Note2                    | Sticky Note        | Explains QuickBooks credential setup            | None                            | None                           | **2. Connect QuickBooks** Select your QuickBooks Online account credentials.                                     |
| Sticky Note3                    | Sticky Note        | Explains logo URL configuration                  | None                            | None                           | **3. Add Your Logo URL** Replace placeholder URL with your company logo URL.                                    |
| Sticky Note4                    | Sticky Note        | Explains signature URL configuration             | None                            | None                           | **4. Add Your Signature URL** Replace placeholder URL with your signature image URL.                            |
| Sticky Note5                    | Sticky Note        | Explains Gotenberg URL configuration             | None                            | None                           | **5. Set Your Gotenberg URL** Replace placeholder URL with your running Gotenberg instance URL.                  |
| Sticky Note6                    | Sticky Note        | Explains email node setup                         | None                            | None                           | **6. Configure Your Email** Select SMTP credentials, customize From, Subject, and email body.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: Listen for New QuickBooks Invoice  
   - HTTP Method: POST  
   - Path: `qbo-invoice-webhook`  
   - Save and copy Production URL for QuickBooks webhook setup.

2. **Create QuickBooks Node:**  
   - Type: QuickBooks  
   - Name: Get Invoice Data from QuickBooks  
   - Resource: Invoice  
   - Invoice ID: Expression: `={{ $json.body.eventNotifications[0].dataChangeEvent.entities[0].id }}`  
   - Credentials: Select or create QuickBooks OAuth2 credentials.

3. **Create HTTP Request Node for Logo:**  
   - Type: HTTP Request  
   - Name: Fetch Company Logo Image  
   - Method: GET  
   - URL: Replace with your public logo image URL (PNG recommended)  
   - Response Format: File (binary)  
   - Data Property Name: `logodata`

4. **Create HTTP Request Node for Signature:**  
   - Type: HTTP Request  
   - Name: Fetch Company Signature Image  
   - Method: GET  
   - URL: Replace with your public signature image URL (PNG recommended)  
   - Response Format: File (binary)  
   - Data Property Name: `signaturedata`

5. **Create Extract From File Node for Logo:**  
   - Type: Extract From File  
   - Name: Convert Logo to Base64  
   - Operation: binaryToProperty  
   - Binary Property Name: `logodata`  
   - Destination Key: `logodata`

6. **Create Extract From File Node for Signature:**  
   - Type: Extract From File  
   - Name: Convert Signature to Base64  
   - Operation: binaryToProperty  
   - Binary Property Name: `signaturedata`  
   - Destination Key: `signaturedata`

7. **Create Merge Node:**  
   - Type: Merge  
   - Name: Combine Invoice, Logo & Signature  
   - Number of Inputs: 3  
   - Connect inputs from:  
     - Get Invoice Data from QuickBooks  
     - Convert Logo to Base64  
     - Convert Signature to Base64

8. **Create Code Node:**  
   - Type: Code  
   - Name: Prepare All Data for Template  
   - Paste the JavaScript to:  
     - Extract invoice, logo, signature data from merged inputs  
     - Validate and convert images to Data URIs  
     - Format dates professionally  
     - Generate line items HTML  
     - Conditionally generate PAID stamp  
     - Return JSON object with all necessary fields for template  
   - Input: Output from Merge node

9. **Create HTML Node:**  
   - Type: HTML  
   - Name: Build HTML Invoice from Data  
   - Paste the entire HTML template with style and placeholders  
   - Use expressions to inject prepared data fields from previous node  
   - Input: Output from Prepare All Data for Template node

10. **Create Convert To File Node:**  
    - Type: Convert To File  
    - Name: Convert HTML to Binary File  
    - Operation: toText  
    - Encoding: UTF-8  
    - File Name: `index.html`  
    - Source Property: `html` (from HTML node output)

11. **Create HTTP Request Node for PDF Generation:**  
    - Type: HTTP Request  
    - Name: Generate PDF via Gotenberg  
    - Method: POST  
    - URL: Set to your Gotenberg instance URL `http://yourGotenBergInstanceURL/forms/chromium/convert/html`  
    - Content-Type: multipart/form-data  
    - Body Parameters:  
      - `files`: binary data from Convert HTML to Binary File node  
      - `scale`: `1`  
      - `metadata`: JSON string with dynamic dates and author/creator info  
    - Header Parameter: `Gotenberg-Output-Filename` set dynamically to invoice number  
    - Input: Output from Convert HTML to Binary File node

12. **Create Email Send Node:**  
    - Type: Email Send  
    - Name: Email PDF Invoice to Customer  
    - From Email: Expression using prepared data customer email or a fixed sender  
    - To Email: Customer email or your company email address  
    - Subject: Include invoice number dynamically, e.g., `Invoice {{$json.invoiceNumber}} from YOUR COMPANY NAME`  
    - Email Body: Paste the provided HTML email template with inline styles and dynamic fields  
    - Attachments: PDF binary from Gotenberg node output  
    - Credentials: SMTP or email service credentials

13. **Connect Workflow:**  
    - Connect `Listen for New QuickBooks Invoice` node outputs to:  
      - `Get Invoice Data from QuickBooks`  
      - `Fetch Company Logo Image`  
      - `Fetch Company Signature Image`  
    - Connect `Get Invoice Data` to input 1 of `Combine Invoice, Logo & Signature`  
    - Connect `Convert Logo to Base64` to input 2 of `Combine Invoice, Logo & Signature`  
    - Connect `Convert Signature to Base64` to input 3 of `Combine Invoice, Logo & Signature`  
    - Connect `Combine Invoice, Logo & Signature` to `Prepare All Data for Template`  
    - Connect `Prepare All Data for Template` to `Build HTML Invoice from Data`  
    - Connect `Build HTML Invoice from Data` to `Convert HTML to Binary File`  
    - Connect `Convert HTML to Binary File` to `Generate PDF via Gotenberg`  
    - Connect `Generate PDF via Gotenberg` to `Email PDF Invoice to Customer`

14. **Final Steps:**  
    - Replace all placeholder URLs and credentials with your own (logo, signature, QuickBooks OAuth2, SMTP, Gotenberg URL).  
    - Test webhook reception by creating a new invoice in QuickBooks.  
    - Activate the workflow in n8n.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| This workflow requires a running Gotenberg instance for HTML to PDF conversion. Gotenberg is an open-source API-based PDF generator. Learn more at: https://gotenberg.dev/                                                                   | Gotenberg official website                          |
| Public URLs for logo and signature images must be direct links accessible by n8n HTTP request nodes. Hosting on your website or image hosting services like Imgur is recommended.                                                             | Image hosting best practices                        |
| QuickBooks Online Webhook configuration is essential to receive invoice events. Set the webhook endpoint URL to the webhook node Production URL, and subscribe to the Invoice event only.                                                     | QuickBooks Developer Dashboard                      |
| SMTP credentials are required for sending emails. Use any SMTP provider or email service that supports SMTP authentication.                                                                                                               | Email service setup                                 |
| The HTML invoice template is multi-page aware and uses CSS for page breaks and footers, resulting in professional PDFs even for long invoices.                                                                                              | CSS print styling for PDF generation                |
| The "PAID" stamp appears only if the invoice balance is zero or less, ensuring visual clarity on paid invoices.                                                                                                                             | Conditional HTML in code node                        |
| The email is sent from the customer’s email by default, which may need adjustment based on your SMTP provider's policies. Modify the "From Email" field accordingly.                                                                          | Email sender customization                           |
| The workflow is inactive by default; activate it after completing all configuration steps.                                                                                                                                                   | Workflow activation in n8n UI                        |
| Contact Elegant Biz Tech (https://www.elegantbiztech.com/) for support or customization inquiries related to this workflow.                                                                                                                | Creator contact                                     |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.