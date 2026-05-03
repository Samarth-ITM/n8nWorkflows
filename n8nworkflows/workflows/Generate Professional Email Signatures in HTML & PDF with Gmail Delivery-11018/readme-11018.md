Generate Professional Email Signatures in HTML & PDF with Gmail Delivery

https://n8nworkflows.xyz/workflows/generate-professional-email-signatures-in-html---pdf-with-gmail-delivery-11018


# Generate Professional Email Signatures in HTML & PDF with Gmail Delivery

### 1. Workflow Overview

This workflow automates the creation and distribution of professional email signatures by generating both HTML and PDF formats and sending them via Gmail. It is designed for users who submit their personal and professional details through a webhook, which then processes the data into a visually appealing email signature, converts it to PDF, and delivers it by email.

**Target Use Cases:**  
- Businesses or individuals wanting dynamic, branded email signatures generated on demand.  
- Automated email signature generation as part of onboarding or marketing workflows.  
- Systems requiring high-quality PDF signatures for consistency and easy sharing.

**Logical Blocks:**  
- **1.1 Input Reception and Validation:** Accepts user data via webhook and extracts relevant inputs.  
- **1.2 HTML Signature Creation:** Builds a premium HTML email signature using extracted data.  
- **1.3 PDF Generation:** Converts the HTML signature into a high-resolution PDF file via an external API.  
- **1.4 PDF Retrieval:** Downloads the generated PDF binary data for further processing.  
- **1.5 Email Delivery:** Sends the PDF as an email attachment using Gmail with a preview link and confirmation message.  
- **1.6 Success Response:** Returns a JSON response confirming successful email delivery and provides the PDF URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

- **Overview:**  
Receives incoming POST requests containing user details for the signature. Extracts and normalizes data fields for consistent processing.

- **Nodes Involved:**  
  - Webhook - Signature Request  
  - Extract Inputs

- **Node Details:**

  - **Webhook - Signature Request**  
    - Type: Webhook (HTTP POST listener)  
    - Role: Entry point for external requests with JSON payload containing user info.  
    - Configuration: Listens on path `/email-signature` with POST method; response mode set to responseNode to allow later response customization.  
    - Inputs: External HTTP requests.  
    - Outputs: JSON body with user data.  
    - Edge Cases: Missing required fields, invalid JSON, unauthorized access (no explicit token validation shown, but recommended).  
    - Version: 2.

  - **Extract Inputs**  
    - Type: Code (JavaScript)  
    - Role: Normalizes and extracts specific user fields from webhook JSON.  
    - Configuration: Extracts `name`, `designation`, `email`, `phone`, `linkedin`, `instagram`, `website` from the webhook body into a clean JSON structure.  
    - Expressions: Uses `$json.body` to access incoming data.  
    - Inputs: JSON from webhook node.  
    - Outputs: Clean JSON with defined fields for downstream nodes.  
    - Edge Cases: Missing keys could result in undefined values; no validation or sanitization present.  
    - Version: 2.

#### 1.2 HTML Signature Creation

- **Overview:**  
Generates a styled, premium HTML email signature including user details and social links formatted with icons and a two-column layout.

- **Nodes Involved:**  
  - Build HTML Signature

- **Node Details:**

  - **Build HTML Signature**  
    - Type: Set  
    - Role: Constructs the complete HTML string for the email signature using template literals with variables.  
    - Configuration: Assigns a field `signature_html` containing an inline HTML template styled with CSS and dynamic placeholders for user data such as name and social links.  
    - Expressions: Injects variables like `{{$json.name}}`, `{{$json.email}}` from the input JSON.  
    - Inputs: Normalized user data from Extract Inputs.  
    - Outputs: JSON with `signature_html` string.  
    - Edge Cases: Missing user data will result in empty placeholders; no fallback text included.  
    - Version: 3.4.

#### 1.3 PDF Generation

- **Overview:**  
Converts the generated HTML signature into a PDF document via an external HTML to PDF conversion API.

- **Nodes Involved:**  
  - HTML to PDF

- **Node Details:**

  - **HTML to PDF**  
    - Type: HTML CSS to PDF  
    - Role: Calls a third-party API to transform the HTML content into a hosted PDF URL.  
    - Configuration: Passes the `signature_html` content as input HTML.  
    - Credentials: Requires an API key/credential for the HTML to PDF service (e.g., pdfmunk.com).  
    - Inputs: JSON containing `signature_html` from Build HTML Signature.  
    - Outputs: JSON containing a `pdf_url` field with the hosted PDF file location.  
    - Edge Cases: API errors, invalid HTML, credential/authentication failures, service downtime.  
    - Version: 1.

#### 1.4 PDF Retrieval

- **Overview:**  
Downloads the PDF file from the URL provided by the conversion API to obtain binary data for email attachment.

- **Nodes Involved:**  
  - Download binary data

- **Node Details:**

  - **Download binary data**  
    - Type: HTTP Request  
    - Role: Fetches the PDF file as binary data from the `pdf_url`.  
    - Configuration: URL set dynamically from `{{$json.pdf_url}}`.  
    - Inputs: JSON with `pdf_url` from HTML to PDF.  
    - Outputs: Binary data representing the PDF file.  
    - Edge Cases: Network errors, invalid URL, timeouts, file not found.  
    - Version: 4.3.

#### 1.5 Email Delivery

- **Overview:**  
Emails the generated PDF signature to the user, including an HTML email body with a preview link and attachment.

- **Nodes Involved:**  
  - Send Email via Gmail

- **Node Details:**

  - **Send Email via Gmail**  
    - Type: Gmail  
    - Role: Sends an email with the PDF attached to the user's email address.  
    - Configuration:  
      - `sendTo` set dynamically from extracted user email.  
      - `subject` fixed as “Your New Email Signature PDF”.  
      - `message` contains personalized greeting, confirmation, and preview link to the PDF.  
      - Attachment uses binary data from previous node.  
    - Credentials: Gmail OAuth2 credentials required for authenticated sending.  
    - Inputs: Binary PDF data and JSON with user email and name.  
    - Outputs: Sent email confirmation data.  
    - Edge Cases: Authentication failure, quota exceeded, invalid email address, attachment size limits.  
    - Version: 2.1.

#### 1.6 Success Response

- **Overview:**  
Returns a JSON response to the webhook caller confirming success and providing the PDF URL.

- **Nodes Involved:**  
  - Success Response

- **Node Details:**

  - **Success Response**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 200 JSON response with status, message, recipient email, and PDF URL.  
    - Configuration: Sets headers for JSON content type; body dynamically includes email and pdf_url from previous nodes.  
    - Inputs: JSON from Send Email via Gmail and HTML to PDF nodes.  
    - Outputs: HTTP response to original webhook caller.  
    - Edge Cases: Response generation failure, missing data references.  
    - Version: 1.1.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                                                                               |
|-------------------------|---------------------------|---------------------------------------|-------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook - Signature Request | Webhook                   | Receives user data via POST            | -                       | Extract Inputs          | # **Input & Extraction** Webhook Intake: Receives JSON (name, role, email, phone, links); Validates required fields; Secure via custom admin token                                       |
| Extract Inputs          | Code                      | Extracts and normalizes user inputs    | Webhook - Signature Request | Build HTML Signature    | # **Input & Extraction** Extract Inputs: Normalizes fields; Creates clean variables for HTML                                                                                              |
| Build HTML Signature    | Set                       | Builds premium HTML signature           | Extract Inputs           | HTML to PDF              | # **HTML Signature Builder** Build Modern Signature: Premium layout (icons + sections); Injects dynamic data safely                                                                       |
| HTML to PDF             | HTML CSS to PDF            | Converts HTML signature to PDF          | Build HTML Signature     | Download binary data     | # **HTML Signature Builder** HTML → PDF: Handles fonts + icon SVGs; Outputs secure hosted `pdf_url`                                                                                         |
| Download binary data    | HTTP Request               | Downloads PDF binary from URL           | HTML to PDF              | Send Email via Gmail     | # **Delivery** Download PDF File: Fetches `pdf_url`; Stores binary (property: data)                                                                                                        |
| Send Email via Gmail    | Gmail                      | Sends email with PDF attachment         | Download binary data     | Success Response         | # **Delivery** Email Delivery: Attaches generated PDF; Includes preview + signature details                                                                                                |
| Success Response        | Respond to Webhook         | Returns JSON success response           | Send Email via Gmail     | -                       | # **Delivery** Return Response: Sends JSON success message                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: "Webhook - Signature Request"  
   - HTTP Method: POST  
   - Path: `email-signature`  
   - Response Mode: Response Node  
   - Purpose: Receive JSON payload with user details.

2. **Add Code Node to Extract Inputs**  
   - Name: "Extract Inputs"  
   - Type: Code (JavaScript)  
   - Paste code to extract fields from `$json.body`:  
     ```javascript
     const b = $json.body;
     return [{
       json: {
         name: b.name,
         designation: b.designation,
         email: b.email,
         phone: b.phone,
         linkedin: b.linkedin,
         instagram: b.instagram,
         website: b.website
       }
     }];
     ```  
   - Connect Webhook → Extract Inputs.

3. **Add Set Node to Build HTML Signature**  
   - Name: "Build HTML Signature"  
   - Type: Set  
   - Add a string field `signature_html` with the full HTML template, injecting dynamic variables using expressions like `{{$json.name}}`. Use the provided multi-line HTML string with inline CSS for layout, icons, and links.  
   - Connect Extract Inputs → Build HTML Signature.

4. **Add HTML to PDF Node**  
   - Name: "HTML to PDF"  
   - Type: HTML CSS to PDF  
   - Set `html_content` parameter to `{{$json.signature_html}}`  
   - Configure credentials with your HTML to PDF API account (e.g., pdfmunk.com).  
   - Connect Build HTML Signature → HTML to PDF.

5. **Add HTTP Request Node to Download PDF**  
   - Name: "Download binary data"  
   - Type: HTTP Request  
   - Set URL to `{{$json.pdf_url}}` dynamically from previous node.  
   - Enable binary data download.  
   - Connect HTML to PDF → Download binary data.

6. **Add Gmail Node to Send Email**  
   - Name: "Send Email via Gmail"  
   - Type: Gmail  
   - Set "Send To" to `{{$('Extract Inputs').item.json.email}}`  
   - Subject: `"Your New Email Signature PDF"`  
   - Message (HTML):  
     ```
     Hi {{$('Extract Inputs').item.json.name}}, <br><br>
     Your email signature has been generated successfully. <br>
     You can download it from the attachment below.<br><br>
     Preview: <br>
     <a href="{{$json.pdf_url}}" style="color:#2563eb;">View Signature PDF</a><br><br>
     Thanks,<br>
     n8n Automation Bot
     ```  
   - Attach the binary PDF data from "Download binary data" node's binary property.  
   - Use Gmail OAuth2 credentials for sending.  
   - Connect Download binary data → Send Email via Gmail.

7. **Add Respond to Webhook Node**  
   - Name: "Success Response"  
   - Type: Respond to Webhook  
   - Set response code to 200  
   - Set Content-Type header to `application/json`  
   - Response Body:  
     ```json
     {
       "status": "success",
       "message": "Email signature PDF sent successfully",
       "sent_to": "{{ $('Extract Inputs').item.json.email }}",
       "pdf_url": "{{ $('HTML to PDF').item.json.pdf_url }}"
     }
     ```  
   - Connect Send Email via Gmail → Success Response.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                      |
|---------------------------------------------------------------------------------------------------------------|------------------------------------|
| Setup Checklist: Add Gmail OAuth2 credentials, verify HTML to PDF API key at https://pdfmunk.com, test webhook with Postman. | Sticky Note1 (Setup Instructions)  |
| How the workflow operates: User inputs → extraction → HTML signature creation → PDF conversion → PDF download → Gmail email → JSON response. | Sticky Note (Workflow Overview)    |
| Input & Extraction: Webhook intake expects JSON with required fields; extraction node normalizes inputs.       | Sticky Note2 (Input & Extraction)  |
| HTML Signature Builder: Creates a modern signature with icons and layout; HTML to PDF node handles fonts and outputs a hosted PDF URL. | Sticky Note3 (HTML Signature Builder) |
| Delivery: Downloads PDF binary, sends email with attachment and preview, returns JSON success confirmation.    | Sticky Note4 (Delivery)             |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or offensive material. All processed data is legal and public.