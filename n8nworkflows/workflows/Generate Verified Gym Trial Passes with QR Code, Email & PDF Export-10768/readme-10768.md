Generate Verified Gym Trial Passes with QR Code, Email & PDF Export

https://n8nworkflows.xyz/workflows/generate-verified-gym-trial-passes-with-qr-code--email---pdf-export-10768


# Generate Verified Gym Trial Passes with QR Code, Email & PDF Export

### 1. Workflow Overview

This workflow automates the creation and distribution of verified gym trial passes with QR codes, photo ID, and PDF export. It is designed for gym operators who want to securely onboard new trial members by verifying their email addresses, generating unique passes, and delivering professional digital passes by email. The workflow handles input reception, email verification, pass generation with QR code, HTML pass design, conversion to image and PDF, email delivery, logging to Google Sheets, and error handling.

Logical blocks:

- **1.1 Input & Validation:** Receives trial signup data via webhook, verifies email authenticity, branches based on validation result.
- **1.2 Pass Generation:** Generates unique pass details including pass ID, timestamps, and formats dates.
- **1.3 QR Code Generation & HTML Pass Design:** Creates a QR code for the pass and builds a styled HTML pass including member photo and validity dates.
- **1.4 Image & PDF Conversion:** Converts the HTML pass to an image and then to a PDF document.
- **1.5 Email Delivery, Logging & Response:** Sends the PDF pass via email, logs the signup in Google Sheets, and returns a JSON success response.
- **1.6 Error Handling:** Returns a structured error response if the email verification fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Validation

**Overview:**  
Handles incoming HTTP POST requests with signup data, validates the email address using the VerifiEmail service, and routes the workflow depending on validation success.

**Nodes Involved:**  
- Webhook  
- Verifi Email  
- IF Email Valid  
- Send Error Response  

**Node Details:**

- **Webhook**  
  - *Type:* Webhook  
  - *Role:* Entry point for receiving gym trial signup data (expects JSON with fields like name, email, photo_url, start_date, valid_till) via POST at path `/gym-trial`.  
  - *Config:* HTTP method POST, response mode set to respond through a later node.  
  - *Input/Output:* No input; outputs request body JSON.  
  - *Edge cases:* Missing required fields, malformed JSON, unauthorized POSTs.  
  - *Sticky Note:* Describes overall workflow intake and testing via Postman.

- **Verifi Email**  
  - *Type:* VerifiEmail node (third-party email verification)  
  - *Role:* Validates the email from webhook data to confirm authenticity and detect disposable or invalid emails.  
  - *Config:* Email parameter set dynamically from webhook JSON `{{ $json.body.email }}`. Requires VerifiEmail API credentials.  
  - *Input:* Webhook output JSON.  
  - *Output:* JSON including `valid` boolean indicating email validity.  
  - *Failure:* API key missing, rate limiting, network errors, invalid email format.  
  - *Sticky Note:* Highlights input and validation logic.

- **IF Email Valid**  
  - *Type:* Conditional (IF) node  
  - *Role:* Branches workflow based on `{{$json.valid}}` boolean from VerifiEmail node.  
  - *Config:* Condition checks if `valid` is true (strict boolean).  
  - *Input:* Verifi Email output.  
  - *Outputs:* True branch for valid emails; false branch for invalid.  
  - *Edge cases:* Missing or malformed `valid` field, expression errors.

- **Send Error Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Returns HTTP 400 error with JSON explaining invalid or disposable email detected.  
  - *Config:* Response code 400, JSON body includes status "error", message, email verified false, and email provided.  
  - *Input:* IF Email Valid false branch.  
  - *Edge cases:* Fail to respond if webhook expired or network issues.  
  - *Sticky Note:* Section on error handling.

---

#### 1.2 Pass Generation

**Overview:**  
Generates a unique pass ID, timestamps, and formats date fields, combining original webhook data with verification results.

**Nodes Involved:**  
- Generate Pass Details

**Node Details:**

- **Generate Pass Details**  
  - *Type:* Code (JavaScript)  
  - *Role:* Synthesizes data for the pass: generates unique pass ID (`GYM-<timestamp>`), ISO issued date, formats start and valid-till dates, and merges original webhook fields with email verification status.  
  - *Config:* Retrieves original webhook JSON using expression `$('Webhook').first().json.body` because the Verifi Email node returns only verification results.  
  - *Input:* IF Email Valid true branch JSON (verification data).  
  - *Output:* Pass details JSON with fields: name, email, photo_url, start_date, valid_till, pass_id, issued_at, start_date_formatted, valid_till_formatted, email_verified.  
  - *Edge cases:* Invalid/missing dates, timezone issues, code execution errors, missing or malformed original webhook data.  
  - *Sticky Note:* Describes pass generation logic.

---

#### 1.3 QR Code Generation & HTML Pass Design

**Overview:**  
Uses the unique pass ID to generate a QR code image URL, then builds a styled HTML pass embedding member photo, validity dates, and QR code.

**Nodes Involved:**  
- Generate QR Code  
- Build HTML Pass

**Node Details:**

- **Generate QR Code**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves a QR code image from external service `api.qrserver.com` encoding the pass_id.  
  - *Config:* URL constructed dynamically as `https://api.qrserver.com/v1/create-qr-code/?data={{ $json.pass_id }}&size=200x200`. Response expected as a file stream/image.  
  - *Input:* Generate Pass Details output JSON.  
  - *Output:* Binary response containing QR code image file.  
  - *Edge cases:* API downtime, network failure, invalid URLs, large data causing QR failure.  
  - *Sticky Note:* Part of pass generation block.

- **Build HTML Pass**  
  - *Type:* Set node  
  - *Role:* Creates a complete HTML document for the gym trial pass using inline CSS styles and dynamic content placeholders.  
  - *Config:* Assigns `html_content` string with embedded HTML template referencing data from both the Webhook and generated pass details (e.g. photo_url, name, formatted dates, pass_id). The QR code is embedded as an image URL from the QR code generator.  
  - *Input:* Generate QR Code output.  
  - *Output:* JSON with `html_content` string field.  
  - *Edge cases:* Template syntax issues, missing variables, image URL broken, CSS rendering inconsistencies.  
  - *Sticky Note:* Notes on design and export block.

---

#### 1.4 Image & PDF Conversion

**Overview:**  
Converts the generated HTML pass to an image (PNG) and then to a PDF document, preparing the digital pass for email attachment.

**Nodes Involved:**  
- HTML/CSS to Image  
- HTML to PDF  
- Digital Pass (HTTP Request)

**Node Details:**

- **HTML/CSS to Image**  
  - *Type:* HTML/CSS to Image node  
  - *Role:* Converts the `html_content` string to a PNG image file using an external API.  
  - *Config:* Input HTML from `Build HTML Pass`. Requires HTMLCSSToImage API credentials.  
  - *Input:* Build HTML Pass output JSON.  
  - *Output:* Binary PNG image.  
  - *Edge cases:* API errors, malformed HTML, network issues, rate limiting.

- **HTML to PDF**  
  - *Type:* HTML to PDF node  
  - *Role:* Converts the same HTML content into a PDF file for official pass distribution.  
  - *Config:* Uses `html_content` from Build HTML Pass node. Requires HTML to PDF service credentials.  
  - *Input:* HTML/CSS to Image output (though only HTML content is referenced).  
  - *Output:* PDF file binary, with `pdf_url` property generated.  
  - *Edge cases:* Conversion failures, API downtime, malformed HTML.

- **Digital Pass**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the generated PDF file via the URL provided by the HTML to PDF node to attach in the email.  
  - *Config:* URL set dynamically from `{{ $json.pdf_url }}`.  
  - *Input:* HTML to PDF node output.  
  - *Output:* Binary PDF file.  
  - *Edge cases:* Broken URLs, network timeouts, file not found errors.

---

#### 1.5 Email Delivery, Logging & Response

**Overview:**  
Sends the PDF pass to the verified user's email, logs pass issuance details into a Google Sheets spreadsheet, and returns a success JSON response.

**Nodes Involved:**  
- Send Email with Pass  
- Log to Google Sheets  
- Send Success Response  

**Node Details:**

- **Send Email with Pass**  
  - *Type:* Gmail node (OAuth2)  
  - *Role:* Sends a rich HTML email to the user with pass details, attaching the PDF pass file.  
  - *Config:*  
    - Recipient: Email from original webhook data.  
    - Subject: "Your Verified Gym Trial Pass - Welcome! 🏋️‍♀️"  
    - Message: HTML formatted welcome message with pass details (pass ID, validity, verification status).  
    - Attachment: PDF binary from Digital Pass node.  
    - Requires Gmail OAuth2 credentials.  
  - *Input:* Digital Pass node output (PDF), original webhook data for email.  
  - *Output:* Email send result JSON.  
  - *Edge cases:* Auth errors, attachment size limits, invalid recipient address, SMTP failures.  
  - *Sticky Note:* Delivery, logging & response block.

- **Log to Google Sheets**  
  - *Type:* Google Sheets node (OAuth2)  
  - *Role:* Appends or updates a row in a Google Sheet to record the trial pass issuance and user info.  
  - *Config:*  
    - Document and sheet IDs must be set (placeholders in JSON).  
    - Columns: Name, Email, Status ("Active"), Pass ID, Issued At, Start Date, Valid Till, Email Verified ("true").  
    - Matching column: Pass ID (to update existing or append new).  
    - Requires Google Sheets OAuth2 credentials.  
  - *Input:* Send Email with Pass output and original webhook data.  
  - *Output:* Google Sheets operation result.  
  - *Edge cases:* Credential expiry, quota limits, sheet structure mismatch.

- **Send Success Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Returns HTTP 200 success response JSON with detailed pass info and flags indicating success of email and logging.  
  - *Config:* Response body dynamically populates with logged data and timestamps.  
  - *Input:* Log to Google Sheets output.  
  - *Output:* JSON HTTP response to original webhook caller.  
  - *Edge cases:* Webhook timeout, partial failures in earlier steps not reflected here.

---

#### 1.6 Error Handling

**Overview:**  
Handles the workflow path when email verification fails, returning an error response with a standardized message.

**Nodes Involved:**  
- Send Error Response (already detailed in 1.1)

---

### 3. Summary Table

| Node Name            | Node Type                      | Functional Role                                | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                  |
|----------------------|--------------------------------|------------------------------------------------|---------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Webhook              | Webhook                        | Receives signup POST request                    | —                   | Verifi Email             | Describes workflow intake, testing via Postman                                               |
| Verifi Email         | VerifiEmail                   | Validates email authenticity                     | Webhook             | IF Email Valid           | Input & Validation block                                                                     |
| IF Email Valid       | IF Condition                  | Branches on email validity                        | Verifi Email        | Generate Pass Details, Send Error Response | Input & Validation block; Error Handling block                                               |
| Send Error Response  | Respond to Webhook            | Returns HTTP 400 with error JSON                  | IF Email Valid (false) | —                        | Error Handling block                                                                         |
| Generate Pass Details| Code                         | Generates unique pass ID, timestamps, formats dates | IF Email Valid (true) | Generate QR Code          | Pass Generation block                                                                        |
| Generate QR Code     | HTTP Request                 | Gets QR code image for pass ID                    | Generate Pass Details | Build HTML Pass           | Pass Generation block                                                                        |
| Build HTML Pass      | Set                          | Builds styled HTML pass with member data and QR  | Generate QR Code     | HTML/CSS to Image         | Design & Export block                                                                        |
| HTML/CSS to Image    | HTML/CSS to Image            | Converts HTML pass to PNG image                    | Build HTML Pass      | HTML to PDF               | Design & Export block                                                                        |
| HTML to PDF          | HTML to PDF                  | Converts HTML pass to PDF document                 | HTML/CSS to Image    | Digital Pass              | Design & Export block                                                                        |
| Digital Pass         | HTTP Request                 | Downloads PDF for email attachment                  | HTML to PDF          | Send Email with Pass      | Design & Export block                                                                        |
| Send Email with Pass | Gmail                        | Sends email with PDF pass attachment               | Digital Pass        | Log to Google Sheets      | Delivery, Logging & Response block                                                           |
| Log to Google Sheets | Google Sheets                | Logs trial pass issuance in spreadsheet            | Send Email with Pass | Send Success Response     | Delivery, Logging & Response block                                                           |
| Send Success Response| Respond to Webhook            | Returns success JSON response                        | Log to Google Sheets | —                        | Delivery, Logging & Response block                                                           |
| Sticky Note          | Sticky Note                  | Notes on workflow overview and blocks              | —                   | —                        | See content for Verified Gym Trial Pass Automation and block descriptions                    |
| Sticky Note21        | Sticky Note                  | Notes on Pass Generation block                      | —                   | —                        | See content for Pass Generation                                                             |
| Sticky Note22        | Sticky Note                  | Notes on Design & Export block                       | —                   | —                        | See content for Design & Export                                                             |
| Sticky Note16        | Sticky Note                  | Notes on Input & Validation block                    | —                   | —                        | See content for Input & Validation                                                          |
| Sticky Note19        | Sticky Note                  | Notes on Delivery, Logging & Response block          | —                   | —                        | See content for Delivery, Logging & Response                                                |
| Section: Error Handling | Sticky Note                | Notes on error handling for invalid emails          | —                   | —                        | See content for Error Handling                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node:**  
   - Type: Webhook  
   - Set HTTP Method to POST  
   - Path: `gym-trial`  
   - Response Mode: `responseNode`

2. **Add Verifi Email node:**  
   - Type: VerifiEmail  
   - Set `email` parameter to expression: `{{$json.body.email}}`  
   - Configure VerifiEmail API credentials with your API key.

3. **Add IF Condition node (IF Email Valid):**  
   - Condition: Check if `{{$json.valid}}` is boolean true  
   - Input: Verifi Email node output  
   - True branch: proceed to pass generation  
   - False branch: send error response

4. **Add Respond to Webhook node (Send Error Response):**  
   - Response Code: 400  
   - Respond With: JSON  
   - Response Body:  
     ```json
     {
       "status": "error",
       "message": "Invalid or disposable email address. Please use a valid email to register.",
       "email_verified": false,
       "email_provided": "{{ $json.email }}"
     }
     ```  
   - Connect from IF Email Valid false branch.

5. **Add Code node (Generate Pass Details):**  
   - Paste the provided JavaScript code to:  
     - Retrieve original webhook data  
     - Generate unique pass ID: `"GYM-" + Date.now()`  
     - Format dates to readable strings  
     - Combine all into output JSON  
   - Connect from IF Email Valid true branch.

6. **Add HTTP Request node (Generate QR Code):**  
   - Method: GET  
   - URL: `https://api.qrserver.com/v1/create-qr-code/?data={{ $json.pass_id }}&size=200x200`  
   - Response Format: File (binary)  
   - Connect from Generate Pass Details.

7. **Add Set node (Build HTML Pass):**  
   - Add a string field `html_content` with the full HTML template embedding:  
     - Member photo URL from `$('Webhook').item.json.body.photo_url`  
     - Member name from `$('Webhook').item.json.body.name`  
     - Formatted start and valid till dates from current JSON  
     - Pass ID and QR code image URL  
   - Connect from Generate QR Code.

8. **Add HTML/CSS to Image node:**  
   - Input: `html_content` from Build HTML Pass  
   - Configure API credentials (htmlcsstoimg.com)  
   - Connect from Build HTML Pass.

9. **Add HTML to PDF node:**  
   - Input: `html_content` from Build HTML Pass  
   - Configure API credentials (pdfmunk.com or equivalent)  
   - Connect from HTML/CSS to Image.

10. **Add HTTP Request node (Digital Pass):**  
    - Method: GET  
    - URL: `{{ $json.pdf_url }}` from HTML to PDF output  
    - Connect from HTML to PDF.

11. **Add Gmail node (Send Email with Pass):**  
    - Send To: `{{ $('Webhook').item.json.body.email }}`  
    - Subject: `"Your Verified Gym Trial Pass - Welcome! 🏋️‍♀️"`  
    - Message: Use provided rich HTML message with pass details and instructions  
    - Attachments: Attach PDF binary from Digital Pass node  
    - Configure Gmail OAuth2 credentials  
    - Connect from Digital Pass.

12. **Add Google Sheets node (Log to Google Sheets):**  
    - Operation: Append or Update  
    - Sheet Name and Document ID: Your Google Sheet details  
    - Mapping columns: Name, Email, Status ("Active"), Pass ID, Issued At, Start Date, Valid Till, Email Verified ("true")  
    - Matching Column: Pass ID  
    - Configure Google Sheets OAuth2 credentials  
    - Connect from Send Email with Pass.

13. **Add Respond to Webhook node (Send Success Response):**  
    - Respond With: JSON  
    - Response Body: JSON including status success, pass details, and flags for email sent and logging success  
    - Connect from Log to Google Sheets.

14. **Add connections as per above steps.**

15. **Add Sticky Notes** (optional) with content describing each block for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Verified Gym Trial Pass Automation automates gym trial pass creation with email verification, QR codes, and PDF export.        | Workflow overview sticky note at workflow start                                                    |
| Setup requires API keys for VerifiEmail (https://verifi.email), HTMLCSSToImage (https://htmlcsstoimg.com), HTML to PDF service.| Setup instructions in sticky notes                                                                |
| Gmail node requires OAuth2 credentials connected to a Gmail account for sending emails securely.                               | Gmail OAuth2 setup                                                                                |
| Google Sheets requires a sheet named "Gym Trial Passes 2025" with columns matching those in logging node for proper operation. | Google Sheets configuration note                                                                  |
| Test the webhook using Postman or similar tools by POSTing JSON with keys: name, email, photo_url, start_date, valid_till.      | Testing instructions in sticky notes                                                              |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.