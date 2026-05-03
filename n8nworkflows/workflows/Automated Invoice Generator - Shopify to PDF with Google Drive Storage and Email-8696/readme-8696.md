Automated Invoice Generator - Shopify to PDF with Google Drive Storage and Email

https://n8nworkflows.xyz/workflows/automated-invoice-generator---shopify-to-pdf-with-google-drive-storage-and-email-8696


# Automated Invoice Generator - Shopify to PDF with Google Drive Storage and Email

### 1. Workflow Overview

This workflow automates the generation and delivery of invoices for Shopify orders. Upon receiving order data from Shopify via webhook, it validates if the order is paid, then formats the order data, generates a professional HTML invoice, converts it to a PDF, saves the PDF to Google Drive, emails the PDF invoice to the customer, and finally sends a confirmation webhook response.

**Target Use Cases:**
- Automating invoice generation and delivery for Shopify orders.
- Ensuring only paid orders are processed.
- Maintaining organized invoice storage in Google Drive.
- Sending professional PDF invoices to customers automatically.
- Providing webhook responses for order processing status.

**Logical Blocks:**

- **1.1 Input Reception & Validation**  
  Receives Shopify order data and checks payment status.

- **1.2 Invoice Data Preparation**  
  Extracts and formats order details into structured invoice data.

- **1.3 Invoice HTML Generation**  
  Creates a styled HTML invoice document.

- **1.4 PDF Invoice Generation**  
  Converts the HTML invoice into a PDF file.

- **1.5 PDF Handling & Storage**  
  Downloads the PDF and saves it to Google Drive.

- **1.6 Customer Notification**  
  Sends the invoice PDF to the customer via email.

- **1.7 Webhook Response Handling**  
  Sends appropriate webhook responses back to Shopify depending on payment status or completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

**Overview:**  
Receives order data from Shopify via webhook and checks if the payment status is "paid" to decide if invoice generation should proceed.

**Nodes Involved:**  
- Shopify Order Webhook  
- Check Payment Status  
- Response for Unpaid Orders

**Node Details:**

- **Shopify Order Webhook**  
  - *Type:* Webhook  
  - *Role:* Entry point; listens for Shopify order creation events.  
  - *Config:* POST HTTP method at path `shopify-webhook`, responds via response node downstream.  
  - *Inputs:* External Shopify HTTP POST webhook call.  
  - *Outputs:* Passes order JSON data downstream.  
  - *Edge Cases:* Missed webhook calls, malformed payloads, network issues.  
  - *Sticky Note:* "Setup this webhook URL to receive order information from Shopify."

- **Check Payment Status**  
  - *Type:* If node  
  - *Role:* Conditional gate; verifies if `financial_status` equals "paid1". *(Note: Possible typo, should be "paid")*  
  - *Config:* Checks `{{$json.body.financial_status}}` equals "paid1".  
  - *Inputs:* Output from Shopify Order Webhook.  
  - *Outputs:* Two branches:  
    - True: Proceed to invoice formatting.  
    - False: Send unpaid order response.  
  - *Edge Cases:* Typo in payment status string could block all invoices, missing financial_status field.  
  - *Sticky Note:* None.

- **Response for Unpaid Orders**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends JSON response indicating invoice not generated because order is unpaid.  
  - *Config:* Responds with success=false, includes order number and payment status from payload.  
  - *Inputs:* False branch of Check Payment Status.  
  - *Outputs:* Ends workflow for unpaid orders.  
  - *Edge Cases:* None specific; ensures Shopify receives clear feedback.  
  - *Sticky Note:* "Send Webhook response for Unpaid Invoice."

---

#### 1.2 Invoice Data Preparation

**Overview:**  
Transforms raw Shopify order JSON into structured invoice data with line items, customer info, totals, and addresses.

**Nodes Involved:**  
- Format Invoice Data

**Node Details:**

- **Format Invoice Data**  
  - *Type:* Code  
  - *Role:* Parses Shopify order JSON, calculates totals, formats customer and address info into a new object `invoiceData`.  
  - *Config:* Uses JavaScript to extract line items, compute subtotal, tax, shipping, total, and arrange customer and billing/shipping address data.  
  - *Key Expressions:* `$input.first().json.body` to access the raw Shopify order data.  
  - *Inputs:* True branch from Check Payment Status.  
  - *Outputs:* Outputs a single JSON object `{ invoiceData }`.  
  - *Edge Cases:* Missing optional fields like phone, shipping address, or notes handled with defaults.  
  - *Sticky Note:* "Prepare Line Item data for next step."

---

#### 1.3 Invoice HTML Generation

**Overview:**  
Generates a professional and styled HTML invoice based on the formatted invoice data.

**Nodes Involved:**  
- Generate HTML Invoice

**Node Details:**

- **Generate HTML Invoice**  
  - *Type:* Code  
  - *Role:* Creates a complete HTML document string using template literals and embedded invoice data.  
  - *Config:* Includes CSS styles inline for layout, colors, fonts; dynamically inserts order number, customer details, line item table, totals, and optional notes.  
  - *Key Expressions:* Uses `data` object from previous node output.  
  - *Inputs:* Output from Format Invoice Data.  
  - *Outputs:* Outputs `{ html }` containing full HTML string.  
  - *Edge Cases:* Handles optional billing/shipping addresses and notes gracefully.  
  - *Sticky Note:* "Generate custom HTML for the invoice."

---

#### 1.4 PDF Invoice Generation

**Overview:**  
Converts the generated HTML invoice into a PDF file using an external HTML-to-PDF API.

**Nodes Involved:**  
- Generate PDF Invoice

**Node Details:**

- **Generate PDF Invoice**  
  - *Type:* HTML CSS to PDF  
  - *Role:* Converts HTML content to PDF format.  
  - *Config:* Receives HTML string from Generate HTML Invoice node; includes custom CSS for A4 page size and print styles to ensure professional PDF output.  
  - *Inputs:* Output from Generate HTML Invoice.  
  - *Outputs:* Produces a PDF URL in JSON output at `pdf_url`.  
  - *Credentials:* Requires API key for HTML to PDF conversion service (e.g., pdfmunk.com).  
  - *Edge Cases:* API key invalid, conversion failures, large HTML causing timeout.  
  - *Sticky Note:* "Convert HTML to PDF Invoice. Get your API Key from https://pdfmunk.com."

---

#### 1.5 PDF Handling & Storage

**Overview:**  
Downloads the generated PDF and stores it securely in a predefined Google Drive folder.

**Nodes Involved:**  
- Download File PDF  
- Save to Google Drive

**Node Details:**

- **Download File PDF**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the PDF file binary from the PDF generation service URL.  
  - *Config:* Uses the URL from `pdf_url` field in previous node output.  
  - *Inputs:* Output from Generate PDF Invoice.  
  - *Outputs:* Provides PDF binary data for email attachment and saving.  
  - *Edge Cases:* URL invalid, network errors, file size too large.  
  - *Sticky Note:* None.

- **Save to Google Drive**  
  - *Type:* Google Drive  
  - *Role:* Uploads the PDF invoice to a designated folder in Google Drive.  
  - *Config:*  
    - File name: `Invoice - {{Shopify Order ID}}`  
    - Drive: "My Drive"  
    - Folder ID: Fixed folder `1A2B3C4D5E6F7G8H9I0J`  
    - Credentials: Google Drive OAuth2 account configured.  
  - *Inputs:* PDF binary from Download File PDF.  
  - *Outputs:* Passes confirmation downstream.  
  - *Edge Cases:* Authentication failure, insufficient permissions, folder missing.  
  - *Sticky Note:* "Save Invoice PDF Store Invoice PDF to Google Drive."

---

#### 1.6 Customer Notification

**Overview:**  
Sends the invoice PDF as an email attachment to the customer.

**Nodes Involved:**  
- Email to Customer

**Node Details:**

- **Email to Customer**  
  - *Type:* Gmail node  
  - *Role:* Sends a well-formatted HTML email including an invoice attachment to the customer.  
  - *Config:*  
    - Recipient: Customer email from Shopify order.  
    - Subject: "Invoice for your Order# [Order Number]"  
    - Message: HTML email body with company branding, personalized greeting, and attachment notice.  
    - Attachment: PDF binary from Download File PDF node.  
    - Credentials: Gmail OAuth2 account configured.  
  - *Inputs:* PDF binary and customer email from Download File PDF and Shopify Order Webhook.  
  - *Outputs:* Passes to webhook response node.  
  - *Edge Cases:* Email sending failure, invalid email address, credentials expired.  
  - *Sticky Note:* "Send Invoice PDF Email to the Customer."

---

#### 1.7 Webhook Response Handling

**Overview:**  
Sends structured JSON responses back to Shopify to acknowledge invoice processing status.

**Nodes Involved:**  
- Send Webhook Response

**Node Details:**

- **Send Webhook Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends success JSON including order number, customer email, and timestamp.  
  - *Config:* Uses data from Format Invoice Data node for order number and email.  
  - *Inputs:* After Email to Customer or Save to Google Drive (both paths present).  
  - *Outputs:* Ends workflow successfully.  
  - *Edge Cases:* Response failure, node execution failure.  
  - *Sticky Note:* "Acknowledge for successful invoice processing."

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                          | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                       |
|-------------------------|------------------------|----------------------------------------|---------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| Shopify Order Webhook    | Webhook                | Receive Shopify order data              | External Shopify webhook  | Check Payment Status     | Setup this webhook URL to receive order information from Shopify                                |
| Check Payment Status     | If                     | Validate payment status                 | Shopify Order Webhook     | Format Invoice Data, Response for Unpaid Orders |                                                                                                 |
| Response for Unpaid Orders| Respond to Webhook     | Respond if order not paid               | Check Payment Status      | None                    | Send Webhook response for Unpaid Invoice                                                        |
| Format Invoice Data      | Code                   | Format order data into invoice object  | Check Payment Status      | Generate HTML Invoice    | Prepare Line Item data for next step                                                            |
| Generate HTML Invoice    | Code                   | Generate styled HTML invoice            | Format Invoice Data       | Generate PDF Invoice     | Generate custom HTML for the invoice                                                           |
| Generate PDF Invoice     | HTML CSS to PDF        | Convert HTML invoice to PDF             | Generate HTML Invoice     | Download File PDF        | Convert HTML to PDF Invoice. Get your API Key from https://pdfmunk.com                          |
| Download File PDF        | HTTP Request           | Download PDF file binary                | Generate PDF Invoice      | Email to Customer, Save to Google Drive |                                                                                                 |
| Email to Customer        | Gmail                  | Email PDF invoice to customer           | Download File PDF         | Send Webhook Response    | Send Invoice PDF Email to the Customer                                                         |
| Save to Google Drive     | Google Drive           | Save PDF invoice to Google Drive folder| Download File PDF         | Send Webhook Response    | Save Invoice PDF Store Invoice PDF to Google Drive                                             |
| Send Webhook Response    | Respond to Webhook     | Send success response to Shopify        | Email to Customer, Save to Google Drive | None                 | Acknowledge for successful invoice processing                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Shopify Order Webhook**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `shopify-webhook`  
   - Response Mode: Response Node  
   - Purpose: Receive JSON order data from Shopify.

2. **Add If Node: Check Payment Status**  
   - Condition: String equals  
   - Expression: `{{$json.body.financial_status}} == "paid1"` (correct to `"paid"` to avoid typo)  
   - True branch: Proceed with invoice generation  
   - False branch: Respond with unpaid order message.

3. **Add Respond to Webhook Node: Response for Unpaid Orders**  
   - Response Body (JSON):  
     ```json
     {
       "success": false,
       "message": "Order not paid yet, invoice not generated",
       "orderNumber": "={{$json.body.order_number}}",
       "paymentStatus": "={{$json.body.financial_status}}"
     }
     ```  
   - Connect False branch of Check Payment Status here.

4. **Add Code Node: Format Invoice Data**  
   - JavaScript: Extract order details from `$input.first().json.body`, calculate totals, format customer and address info.  
   - Output single object with `invoiceData`.  
   - Connect True branch of Check Payment Status here.

5. **Add Code Node: Generate HTML Invoice**  
   - JavaScript: Use template literals to generate full HTML invoice document with embedded CSS styles and invoice data from previous node.  
   - Output `{ html }`.  
   - Connect from Format Invoice Data node.

6. **Add HTML CSS To PDF Node: Generate PDF Invoice**  
   - Input: HTML content from previous node (`{{$json.html}}`).  
   - Configure CSS for A4 page size and print styles.  
   - Credentials: Setup API key with HTML to PDF service (e.g., pdfmunk.com).  
   - Connect from Generate HTML Invoice node.

7. **Add HTTP Request Node: Download File PDF**  
   - Method: GET  
   - URL: `{{$json.pdf_url}}` (output from previous node)  
   - Connect from Generate PDF Invoice node.

8. **Add Gmail Node: Email to Customer**  
   - Recipient: `{{$json.body.customer.email}}` from Shopify payload.  
   - Subject: `Invoice for your Order# {{$json.body.order_number}}`  
   - Message: HTML email with company branding and invoice attachment.  
   - Attachments: Use binary data from Download File PDF node.  
   - Credentials: Configure Gmail OAuth2 account.  
   - Connect from Download File PDF node.

9. **Add Google Drive Node: Save to Google Drive**  
   - File Name: `Invoice - {{$json.body.id}}`  
   - Drive: My Drive  
   - Folder ID: Set to your invoices folder ID.  
   - Credentials: Google Drive OAuth2 configured.  
   - Connect from Download File PDF node.

10. **Add Respond to Webhook Node: Send Webhook Response**  
    - Respond with JSON success message including order number, customer email, and current timestamp.  
    - Connect outputs from Email to Customer and Save to Google Drive nodes (both connect here).

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                          |
|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| This workflow automates Shopify invoice generation: from order reception, validation, invoice creation, PDF conversion, storage, and emailing. | Overall project description              |
| HTML to PDF conversion requires an API key from https://pdfmunk.com                                                                | PDF conversion service                   |
| Ensure Google Drive OAuth2 credentials have permission to write to the specified folder.                                            | Google Drive integration note            |
| Correct the payment status check condition from `"paid1"` to `"paid"` to avoid logical errors.                                     | Critical fix for payment validation      |
| Email node uses Gmail OAuth2—make sure the Gmail account is authorized and allowed to send emails programmatically.                | Gmail OAuth2 credentials note            |
| Shopify webhook URL must be configured in Shopify admin to point to the n8n webhook URL with path `/shopify-webhook`                | Shopify webhook setup                    |
| For better reliability, handle edge cases such as missing customer phone or addresses gracefully, as done in the code nodes.       | Data formatting robustness               |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.