Generate & Deliver Invoices from JotForm to Google Drive & Email with PDF Conversion

https://n8nworkflows.xyz/workflows/generate---deliver-invoices-from-jotform-to-google-drive---email-with-pdf-conversion-9447


# Generate & Deliver Invoices from JotForm to Google Drive & Email with PDF Conversion

### 1. Workflow Overview

This workflow automates the entire process of generating and delivering invoices based on order data received from JotForm submissions. It targets e-commerce or service businesses using JotForm to collect order information and aims to streamline invoicing by automating invoice creation, storage, and emailing.

**Logical Blocks:**

- **1.1 Input Reception:** Receives order data from JotForm via webhook.
- **1.2 Data Formatting:** Extracts and structures relevant order details to prepare for invoice generation.
- **1.3 Invoice HTML Generation:** Produces a professional, branded HTML invoice from formatted data.
- **1.4 PDF Conversion:** Converts the HTML invoice into a PDF file.
- **1.5 File Handling:** Downloads the generated PDF and saves it to Google Drive.
- **1.6 Customer Notification:** Emails the PDF invoice to the customer.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  Captures new order submissions from JotForm via a webhook trigger node.

- **Nodes Involved:**  
  - JotForm Trigger

- **Node Details:**

  - **JotForm Trigger**  
    - **Type & Role:** Trigger node that listens for new form submissions from JotForm.  
    - **Configuration:** Linked to a specific JotForm form ID (`252815424602048`), authenticates via stored JotForm API credentials.  
    - **Expressions/Variables:** Outputs raw form submission data under `body`.  
    - **Connections:** Output feeds into "Format Invoice Data" node.  
    - **Version Requirements:** n8n version supporting JotForm Trigger node (v1+).  
    - **Edge Cases:**  
      - Authentication failure if API credentials expire or are revoked.  
      - Network issues causing webhook delivery failures.  
      - Invalid or incomplete form data submissions.  
    - **Sub-workflow:** None.

---

#### Block 1.2: Data Formatting

- **Overview:**  
  Processes raw order data, calculates totals, and formats customer, billing, shipping, and item information into a structured invoice data object.

- **Nodes Involved:**  
  - Format Invoice Data

- **Node Details:**

  - **Format Invoice Data**  
    - **Type & Role:** Code node executing JavaScript to transform raw order JSON into a clean, structured invoice data object.  
    - **Configuration:**  
      - Parses order line items, calculates subtotal, tax, shipping, and total amounts.  
      - Extracts customer name, email, phone, billing and shipping addresses, payment method, notes.  
      - Ensures fallback values for optional fields (e.g., SKU defaults to 'N/A').  
    - **Expressions/Variables:** Uses `$input.first().json.body` to access incoming order data.  
    - **Input:** Output from "JotForm Trigger".  
    - **Output:** JSON object under key `invoiceData`.  
    - **Version Requirements:** Supports n8n Code node v2.  
    - **Edge Cases:**  
      - Missing or malformed order fields could cause runtime errors (e.g., missing `line_items`).  
      - Data type mismatches (e.g., price as string vs number).  
      - Null or undefined optional fields handled gracefully.  
    - **Sub-workflow:** None.

---

#### Block 1.3: Invoice HTML Generation

- **Overview:**  
  Constructs a visually professional and branded HTML invoice using the formatted invoice data.

- **Nodes Involved:**  
  - Generate HTML Invoice

- **Node Details:**

  - **Generate HTML Invoice**  
    - **Type & Role:** Code node generating a full HTML document string for the invoice.  
    - **Configuration:**  
      - Uses template literals to inject invoice data into styled HTML with sections for company info, billing, shipping, payment, line items, totals, and notes.  
      - Applies CSS styles inline for layout and branding.  
      - Formats currency and dates for readability.  
    - **Expressions/Variables:** Accesses data as `$input.first().json.invoiceData`.  
    - **Input:** Output from "Format Invoice Data".  
    - **Output:** Key `html` with full HTML invoice content.  
    - **Version Requirements:** Code node v2.  
    - **Edge Cases:**  
      - Missing optional fields like shipping address or notes handled by conditional rendering.  
      - Large line items lists could affect HTML size and PDF conversion performance.  
    - **Sub-workflow:** None.

---

#### Block 1.4: PDF Conversion

- **Overview:**  
  Converts the generated HTML invoice into a PDF document suitable for download and distribution.

- **Nodes Involved:**  
  - Generate PDF Invoice  
  - Download File PDF

- **Node Details:**

  - **Generate PDF Invoice**  
    - **Type & Role:** HTML to PDF conversion node (htmlcsstopdf).  
    - **Configuration:**  
      - Takes HTML content from previous node and requests PDF generation.  
      - Uses default settings (no additional parameters specified).  
    - **Input:** Output from "Generate HTML Invoice" (HTML string).  
    - **Output:** URL or metadata for the generated PDF file (with `pdf_url`).  
    - **Version Requirements:** htmlcsstopdf node v1.  
    - **Edge Cases:**  
      - PDF conversion may fail due to malformed HTML or service errors.  
      - Timeout or API key quota limits (if external service required).  
    - **Sub-workflow:** None.

  - **Download File PDF**  
    - **Type & Role:** HTTP Request node to download the PDF file from the generated URL.  
    - **Configuration:**  
      - Uses dynamic URL from previous node output (`{{$json.pdf_url}}`).  
      - Default HTTP GET request.  
    - **Input:** Output from "Generate PDF Invoice".  
    - **Output:** Binary PDF file data for attachment and storage.  
    - **Version Requirements:** HTTP Request node v4.2+.  
    - **Edge Cases:**  
      - Download failure due to invalid URL or network issues.  
      - Large file sizes may cause memory or timeout problems.  
    - **Sub-workflow:** None.

---

#### Block 1.5: File Handling

- **Overview:**  
  Saves the generated PDF invoice file to a specified Google Drive folder for record-keeping.

- **Nodes Involved:**  
  - Save to Google Drive

- **Node Details:**

  - **Save to Google Drive**  
    - **Type & Role:** Google Drive node for file upload.  
    - **Configuration:**  
      - File name dynamically set using order ID from JotForm data (`Invoice - {{ $('Shopify Order Webhook').item.json.body.id }}`).  
      - Target folder specified by folder ID (`1A2B3C4D5E6F7G8H9I0J`).  
      - Drive set to "My Drive".  
      - Uploads binary PDF data received from "Download File PDF".  
    - **Input:** Output binary data from "Download File PDF".  
    - **Output:** Confirmation metadata of uploaded file.  
    - **Version Requirements:** Google Drive node v3.  
    - **Edge Cases:**  
      - Authentication/permission issues with Google Drive OAuth2 credentials.  
      - Folder ID invalid or deleted.  
      - Upload failure due to file size or quota.  
    - **Sub-workflow:** None.

---

#### Block 1.6: Customer Notification

- **Overview:**  
  Sends an email to the customer with the PDF invoice attached, confirming the order and payment.

- **Nodes Involved:**  
  - Email to Customer

- **Node Details:**

  - **Email to Customer**  
    - **Type & Role:** Gmail node to send emails with attachments.  
    - **Configuration:**  
      - Recipient email dynamically set using customer email field from JotForm data.  
      - Subject line personalized with order number.  
      - HTML email body with company branding, polite messaging, and instructions.  
      - PDF invoice attached as binary data from "Download File PDF".  
      - Uses Gmail API OAuth2 credentials configured in n8n.  
    - **Input:** Binary PDF data and order info from "Download File PDF".  
    - **Output:** Email sent confirmation.  
    - **Version Requirements:** Gmail node v2.1+.  
    - **Edge Cases:**  
      - OAuth token expiration or scope issues.  
      - Email address missing or invalid.  
      - Attachment size exceeding Gmail limits.  
      - Network or SMTP errors.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name            | Node Type                 | Functional Role                       | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                           |
|----------------------|---------------------------|------------------------------------|------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------|
| JotForm Trigger      | jotFormTrigger             | Receive order submissions from JotForm | —                      | Format Invoice Data            | Setup this webhook URL to receive order information from Jotform. [Sign up for Jotform for free](https://www.jotform.com/?partner=mediajade) |
| Format Invoice Data  | Code                      | Parse and format order data for invoice | JotForm Trigger        | Generate HTML Invoice          | Prepare Line Item data for next step                                                                |
| Generate HTML Invoice| Code                      | Generate professional invoice HTML | Format Invoice Data     | Generate PDF Invoice           | Generate custom HTML for the invoice                                                                |
| Generate PDF Invoice | htmlcsstopdf              | Convert HTML invoice to PDF        | Generate HTML Invoice   | Download File PDF              | Convert HTML to PDF Invoice. Get your API Key from [PDFMunk](https://pdfmunk.com)                     |
| Download File PDF    | httpRequest               | Download generated PDF file        | Generate PDF Invoice    | Email to Customer, Save to Google Drive |                                                                                                     |
| Save to Google Drive | googleDrive               | Save PDF invoice to Google Drive   | Download File PDF       | —                             | Save Invoice PDF. Store Invoice PDF to Google Drive                                                 |
| Email to Customer    | gmail                     | Email PDF invoice to customer      | Download File PDF       | —                             | Email Invoice. Send Invoice PDF Email to the Customer                                               |
| Sticky Note          | stickyNote                | Informational                      | —                      | —                             | Setup this webhook URL to receive order information from Jotform. [Sign up for Jotform for free](https://www.jotform.com/?partner=mediajade) |
| Sticky Note1         | stickyNote                | Informational                      | —                      | —                             | Generate custom HTML for the invoice                                                                |
| Sticky Note2         | stickyNote                | Informational                      | —                      | —                             | Generate PDF. Get your API Key from [PDFMunk](https://pdfmunk.com)                                   |
| Sticky Note3         | stickyNote                | Informational                      | —                      | —                             | Prepare Line Item data for next step                                                                |
| Sticky Note4         | stickyNote                | Informational                      | —                      | —                             | Email Invoice. Send Invoice PDF Email to the Customer                                               |
| Sticky Note5         | stickyNote                | Informational                      | —                      | —                             | Save Invoice PDF. Store Invoice PDF to Google Drive                                                 |
| Sticky Note8         | stickyNote                | Workflow Summary                   | —                      | —                             | 🎯 What This Template Does: Transform your Jotform order fulfillment with complete invoice automation. Automatic receipt, HTML invoice generation, PDF conversion, Google Drive saving, and email delivery. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node:**  
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials.  
   - Set Form ID to your specific JotForm order form (e.g., `252815424602048`).  
   - This node listens for new form submissions.

2. **Add Code Node "Format Invoice Data":**  
   - Type: Code (JavaScript)  
   - Input: Connect from JotForm Trigger.  
   - Paste the provided JavaScript to parse order data, calculate totals, and structure invoice data.  
   - Outputs JSON object `invoiceData` with all necessary invoice fields.

3. **Add Code Node "Generate HTML Invoice":**  
   - Type: Code (JavaScript)  
   - Input: Connect from "Format Invoice Data".  
   - Paste the HTML template code that dynamically builds a complete invoice using `invoiceData`.  
   - Output is the full HTML content under key `html`.

4. **Add Node "Generate PDF Invoice":**  
   - Type: HTML to PDF (htmlcsstopdf).  
   - Input: Connect from "Generate HTML Invoice".  
   - No special parameters needed; defaults suffice.  
   - This converts the HTML into a PDF document and provides a URL to the PDF.

5. **Add Node "Download File PDF":**  
   - Type: HTTP Request  
   - Input: Connect from "Generate PDF Invoice".  
   - Set URL parameter to `{{$json.pdf_url}}` to dynamically download the generated PDF.  
   - Use HTTP GET method.  
   - This node downloads the PDF as binary data for further use.

6. **Add Node "Save to Google Drive":**  
   - Type: Google Drive  
   - Input: Connect from "Download File PDF".  
   - Configure Google Drive OAuth2 credentials.  
   - Set `Name` parameter to dynamically name file as `Invoice - {{ $('JotForm Trigger').item.json.body.id }}` or equivalent unique order identifier.  
   - Select your target folder by Folder ID (e.g., `1A2B3C4D5E6F7G8H9I0J`).  
   - Upload the binary PDF file received from the previous node.

7. **Add Node "Email to Customer":**  
   - Type: Gmail  
   - Input: Connect from "Download File PDF".  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient to customer email: `={{ $('JotForm Trigger').item.json.body.customer.email }}`.  
   - Subject: "Invoice for your Order# {{ $('JotForm Trigger').item.json.body.order_number }}".  
   - Message: Use provided HTML email template with dynamic substitutions for customer name and order number.  
   - Attach the binary PDF from "Download File PDF".  
   - Send the email with the invoice attached.

8. **Connect Nodes Sequentially:**  
   - JotForm Trigger → Format Invoice Data → Generate HTML Invoice → Generate PDF Invoice → Download File PDF → (parallel to) Save to Google Drive and Email to Customer.

**Notes:**  
- Ensure all credentials (JotForm API, Google Drive OAuth2, Gmail OAuth2) are properly configured before activation.  
- Validate form data field mappings match your JotForm form structure.  
- Test each step with sample data to confirm formatting and delivery.  

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                             |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Setup this webhook URL to receive order information from Jotform                                               | [Jotform](https://www.jotform.com/?partner=mediajade)       |
| Get your API Key from PDFMunk to enable HTML to PDF conversion                                                 | [PDFMunk](https://pdfmunk.com)                              |
| Workflow automates receiving, processing, saving, and emailing invoices from JotForm order submissions         | Internal project documentation                               |
| For email customization, adapt the HTML template in "Email to Customer" node to match your branding and tone  | N/A                                                         |
| Google Drive folder ID must be pre-created and accessible via OAuth2 credentials                                | Google Drive API documentation                              |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.