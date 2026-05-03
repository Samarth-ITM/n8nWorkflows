Google services (Sheets/Drive/Gmail) instead of 'Google Suite'

https://n8nworkflows.xyz/workflows/google-services--sheets-drive-gmail--instead-of--google-suite--10387


# Google services (Sheets/Drive/Gmail) instead of 'Google Suite'

### 1. Workflow Overview

This workflow automates the generation, encryption, storage, and delivery of invoice documents using PDF Generator API and Google services (Sheets, Drive, Gmail). It is designed to handle incoming invoice requests containing customer and line item data, generate a unique invoice ID, ensure no duplicates in Google Sheets, create a PDF invoice, encrypt it, upload it to Google Drive, and send it via Gmail to the customer with the encrypted PDF attached.

Logical blocks:

- **1.1 Input Reception:** Receives incoming invoice data via a webhook.
- **1.2 Unique Invoice ID Generation & Validation:** Generates a unique invoice ID and checks for duplicates in Google Sheets.
- **1.3 PDF Invoice Generation & Encryption:** Creates the invoice PDF using PDF Generator API and encrypts it with a customer-specific password.
- **1.4 File Upload & Email Sending:** Uploads the encrypted PDF to Google Drive and emails it to the customer via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Captures incoming invoice data via a webhook to trigger the workflow. The data includes customer details and line items.

- **Nodes Involved:**  
  - Webhook  
  - Sticky Note (Pinned Test Data)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point receiving JSON invoice data (customer info, line items).  
    - Configuration: Path uniquely identifying the webhook; no authentication or additional options configured.  
    - Inputs: External HTTP request with invoice payload.  
    - Outputs: Passes JSON data downstream.  
    - Edge Cases: Missing or malformed JSON could cause failures downstream; no explicit validation here.

  - **Sticky Note (Pinned Test Data)**  
    - Type: Sticky Note  
    - Role: Provides sample test data simulating the webhook payload.  
    - Content: Contains example customer, address, line items, payment QR data, and customer ID.  
    - Inputs/Outputs: None, informational only.

#### 2.2 Unique Invoice ID Generation & Validation

- **Overview:**  
Generates a random invoice ID and verifies against a Google Sheet to avoid duplicates before proceeding.

- **Nodes Involved:**  
  - Generate Invoice ID (Code)  
  - Check if ID Already Exists (Google Sheets)  
  - If Does not Exist (If)  
  - Sticky Note1 (block description)

- **Node Details:**

  - **Generate Invoice ID**  
    - Type: Code  
    - Role: Generates an 8-character alphanumeric invoice number prefixed with "INVOICE-".  
    - Configuration: JavaScript function producing a random string from uppercase letters and digits.  
    - Key Expressions: Returns JSON with `order_number`.  
    - Inputs: From Webhook.  
    - Outputs: Provides generated invoice ID downstream.  
    - Edge Cases: Random collision possible but mitigated by next node.

  - **Check if ID Already Exists**  
    - Type: Google Sheets  
    - Role: Queries a specific Google Sheet (Sheet2) to check if the generated invoice ID already exists (to avoid duplicates).  
    - Configuration: Uses OAuth2 credentials for Google Sheets, document ID and sheet name configured.  
    - Inputs: Invoice ID from previous node.  
    - Outputs: Passes row data or empty result if no match.  
    - Edge Cases: API errors, rate limits, or incorrect sheet setup could cause failures.

  - **If Does not Exist**  
    - Type: If  
    - Role: Checks if the invoice ID was found in the sheet.  
    - Configuration: Condition tests if `row_number` is empty (no existing entry).  
    - Inputs: Output from Check ID node.  
    - Outputs:  
      - True: Proceed to generate PDF.  
      - False: Loop back to Generate Invoice ID to retry.  
    - Edge Cases: Infinite loops if sheet access fails or no unique ID can be generated.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Describes the purpose of generating invoice number and checking duplicates.

#### 2.3 PDF Invoice Generation & Encryption

- **Overview:**  
Generates a PDF invoice document populated with invoice data and encrypts the PDF using the customer's ID as password.

- **Nodes Involved:**  
  - Generate a PDF document (PDF Generator API)  
  - Encrypt PDF document (PDF Generator API)  
  - Sticky Note2 (block description)  
  - Sticky Note4 (PDF Generator API explanation)

- **Node Details:**

  - **Generate a PDF document**  
    - Type: PDF Generator API Node  
    - Role: Creates a PDF invoice from a template, filling in dynamic data fields received through the webhook.  
    - Configuration:  
      - Uses a saved template ID (496040).  
      - Data includes invoice number, customer name, company, address, fees, payment QR code, and line items serialized as JSON string.  
      - Output is a URL to the generated PDF.  
    - Inputs: Triggered after confirming unique invoice ID.  
    - Outputs: URL to the generated PDF file.  
    - Edge Cases: Network/API errors, incorrect template ID or data formatting could cause generation failure.

  - **Encrypt PDF document**  
    - Type: PDF Generator API Node  
    - Role: Encrypts the generated PDF file to secure it before sending.  
    - Configuration:  
      - Uses the PDF URL from the previous node.  
      - Operation: encrypt with password set to the `customer_id` from webhook data.  
      - Output format: file (binary).  
    - Inputs: URL of generated PDF.  
    - Outputs: Encrypted PDF file and filename.  
    - Edge Cases: Encryption failure, invalid URL, or missing password may cause errors.

  - **Sticky Note2**  
    - Describes the generation and encryption process of the PDF document.

  - **Sticky Note4**  
    - Explains the PDF Generator API node’s functionality and importance of encryption.

#### 2.4 File Upload & Email Sending

- **Overview:**  
Uploads the encrypted PDF to a specific Google Drive folder and then emails it as an attachment to the customer.

- **Nodes Involved:**  
  - Upload file (Google Drive)  
  - Send a message + file (Gmail)  
  - Sticky Note3 (block description)

- **Node Details:**

  - **Upload file**  
    - Type: Google Drive  
    - Role: Uploads the encrypted PDF file to a specified folder in Google Drive.  
    - Configuration:  
      - Filename set to the generated invoice ID.  
      - Folder ID targets a specific folder ("n8n_folder").  
      - OAuth2 credentials for Google Drive used.  
      - Input binary data comes from the encrypted PDF node.  
    - Inputs: Encrypted PDF file binary.  
    - Outputs: Metadata about the uploaded file.  
    - Edge Cases: Upload failures due to credential expiration, folder permission issues, or file corruption.

  - **Send a message + file**  
    - Type: Gmail  
    - Role: Sends an email to the customer with the encrypted PDF invoice attached.  
    - Configuration:  
      - Recipient email from webhook data.  
      - Email subject includes the invoice number.  
      - Body includes customer name, invoice number, and instructions about encryption password (customer_id).  
      - Attachment: the encrypted PDF binary.  
      - Uses Gmail OAuth2 credentials.  
    - Inputs: File from Google Drive upload node (binary attachment).  
    - Outputs: Email send confirmation.  
    - Edge Cases: Email send failures, invalid email, attachment size limits.

  - **Sticky Note3**  
    - Describes the upload and email sending process.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                                  | Input Node(s)            | Output Node(s)                     | Sticky Note                                                                                     |
|-------------------------|--------------------------------|-------------------------------------------------|--------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Webhook                 | Webhook                        | Receive incoming invoice data                    | -                        | Generate Invoice ID               | Pinned test data simulates the webhook input.                                                  |
| Generate Invoice ID      | Code                          | Generate unique invoice number                    | Webhook                  | Check if ID Already Exists        | Generate a random INVOICE number and check for duplicities.                                    |
| Check if ID Already Exists | Google Sheets                | Check invoice ID uniqueness in Google Sheets     | Generate Invoice ID       | If Does not Exist                 | Generate a random INVOICE number and check for duplicities.                                    |
| If Does not Exist        | If                            | Branch based on ID existence                       | Check if ID Already Exists | Generate a PDF document / Generate Invoice ID | Generate a random INVOICE number and check for duplicities.                                    |
| Generate a PDF document  | PDF Generator API              | Create PDF invoice document                        | If Does not Exist         | Encrypt PDF document              | Generate a document and Encrypt. PDF Generator API dynamically fills invoice template and encrypts PDF. |
| Encrypt PDF document     | PDF Generator API              | Encrypt PDF with customer password                 | Generate a PDF document   | Upload file                      | Generate a document and Encrypt. PDF Generator API dynamically fills invoice template and encrypts PDF. |
| Upload file             | Google Drive                   | Upload encrypted PDF to Google Drive               | Encrypt PDF document      | Send a message + file            | Upload the file and send the invoice to the customer.                                         |
| Send a message + file    | Gmail                         | Email encrypted invoice to customer                | Upload file              | -                                | Upload the file and send the invoice to the customer.                                         |
| Sticky Note             | Sticky Note                   | Informational / comments                            | -                        | -                                | Pinned TEST data simulates the test payload (JSON) from the Webhook.                          |
| Sticky Note1            | Sticky Note                   | Informational / comments                            | -                        | -                                | Generate a random INVOICE number and check for duplicities.                                    |
| Sticky Note2            | Sticky Note                   | Informational / comments                            | -                        | -                                | Generate a document and Encrypt.                                                               |
| Sticky Note3            | Sticky Note                   | Informational / comments                            | -                        | -                                | Upload the file and send the invoice to the customer.                                         |
| Sticky Note4            | Sticky Note                   | Informational / comments                            | -                        | -                                | PDF Generator API node explanation.                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Name: `Webhook`  
   - Path: unique string (e.g., a UUID)  
   - No authentication required.  
   - This node receives JSON payload including customer info, line items, email, customer_id, etc.

2. **Add a Code node**  
   - Name: `Generate Invoice ID`  
   - Connect input from `Webhook`.  
   - Paste JavaScript code to generate an 8-character alphanumeric invoice number prefixed with "INVOICE-".  
   - Output JSON property: `order_number`.

3. **Add a Google Sheets node**  
   - Name: `Check if ID Already Exists`  
   - Connect from `Generate Invoice ID`.  
   - Configure credentials with Google Sheets OAuth2.  
   - Set Document ID to your target invoice sheet.  
   - Set Sheet Name (e.g., "Sheet2").  
   - Configure search to query the sheet for the generated invoice ID to check duplicates.

4. **Add an If node**  
   - Name: `If Does not Exist`  
   - Connect from `Check if ID Already Exists`.  
   - Condition: Check if the output row number is empty (no match found).  
   - True branch: continue to PDF generation.  
   - False branch: loop back to `Generate Invoice ID` to try a new ID.

5. **Add a PDF Generator API node**  
   - Name: `Generate a PDF document`  
   - Connect from the True output of `If Does not Exist`.  
   - Configure credentials for PDF Generator API.  
   - Set Template ID to your invoice template ID.  
   - Set Data input as JSON including:  
     - `order_number` from `Generate Invoice ID` node  
     - Customer details from `Webhook`  
     - Line items serialized as JSON string  
   - Output as URL to the generated PDF file.

6. **Add another PDF Generator API node**  
   - Name: `Encrypt PDF document`  
   - Connect from `Generate a PDF document`.  
   - Configure credentials for PDF Generator API.  
   - Operation: Encrypt PDF using resource "pdfServices".  
   - Input file URL from previous node output.  
   - Set `ownerPassword` to the `customer_id` from the webhook data.  
   - Output as binary file.

7. **Add a Google Drive node**  
   - Name: `Upload file`  
   - Connect from `Encrypt PDF document`.  
   - Configure Google Drive OAuth2 credentials.  
   - Set Drive ID (e.g., "My Drive").  
   - Set Folder ID to your target folder where invoices are stored.  
   - Filename: Use the invoice ID from `Generate Invoice ID`.  
   - Input binary data is the encrypted PDF file.

8. **Add a Gmail node**  
   - Name: `Send a message + file`  
   - Connect from `Upload file`.  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient email from the webhook data.  
   - Subject: include invoice number.  
   - Message body: personalized with customer name, invoice number, and password info.  
   - Attach the encrypted PDF file binary from the previous node.

9. **Link the False branch of the If node** back to `Generate Invoice ID` to retry generating a unique invoice ID if a duplicate is found.

10. **Add sticky notes** throughout the workflow to annotate blocks for clarity as per logical grouping.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| PDF Generator API dynamically generates and encrypts PDFs from templates; template used here is “n8n Invoice Example (ID: 496040)”.                        | https://pdfgeneratorapi.com                                                                                   |
| Google Sheets is used to ensure invoice ID uniqueness by checking for existing entries in a specific sheet.                                                 | Google Sheets OAuth2 API                                                                                      |
| Google Drive folder must exist and be accessible by the OAuth2 credential used, to upload encrypted invoices.                                               | https://drive.google.com                                                                                       |
| Email sent via Gmail uses OAuth2; ensure the Gmail account has correct permissions to send emails with attachments.                                         | Gmail OAuth2 API                                                                                              |
| Invoice encryption password is set to the customer's unique ID to secure the document.                                                                      |                                                                                                             |
| The workflow’s input is expected in a JSON format matching the pinned test data example for correct operation.                                              |                                                                                                             |
| Be cautious of infinite loops if invoice ID uniqueness check repeatedly fails; consider adding a retry limit or fallback.                                  |                                                                                                             |

---

**Disclaimer:** The provided document is an analysis of an n8n workflow automating invoice generation and delivery; all data and operations comply with content policies and handle only legal and public information.