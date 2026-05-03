Hackathon Participant Badge Generator with QR Code, PDF & Email Delivery

https://n8nworkflows.xyz/workflows/hackathon-participant-badge-generator-with-qr-code--pdf---email-delivery-11016


# Hackathon Participant Badge Generator with QR Code, PDF & Email Delivery

---

### 1. Workflow Overview

This workflow automates the generation and delivery of personalized participant badges for hackathon events. It accepts participant registration data via a webhook, validates the input (including email verification), generates a unique badge ID and QR code, renders an HTML badge as a PDF, sends the badge by email with an inline preview and PDF attachment, logs the participant data into Google Sheets, and returns a success response.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Validation:** Receive registration data via webhook, validate required fields and email format, verify email deliverability and disposability.
- **1.2 Badge ID & QR Code Generation:** Generate a unique badge ID, verification URL, issue date, and QR code link.
- **1.3 Badge Rendering & PDF Creation:** Render a styled HTML badge using participant data and convert it into a PDF.
- **1.4 Delivery & Logging:** Download the PDF, send it via Gmail with a polished HTML email, log participant details to Google Sheets, and send a confirmation response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block receives participant registration data via a secure webhook POST request, validates the presence and correctness of essential fields (name, email, event), and verifies the email's deliverability and disposability in real-time to prevent fake or junk registrations.

- **Nodes Involved:**  
  - Webhook Trigger  
  - Validate Input (Function)  
  - Verifi Email  
  - Email Valid? (IF)

- **Node Details:**

  - **Webhook Trigger**  
    - Type: Webhook (HTTP POST endpoint)  
    - Configuration: Listens on path `/hackathon-badge` with POST method.  
    - Inputs: External HTTP POST request containing JSON body with participant data.  
    - Outputs: Passes received data as JSON to next node.  
    - Failure Modes: Invalid HTTP method, missing body, network issues.  
    - Notes: Entry point of the workflow.

  - **Validate Input**  
    - Type: Function  
    - Role: Extracts the payload from webhook input and validates required fields: `name`, `email`, `event`.  
    - Configuration:  
      - Checks if required fields exist and are non-empty.  
      - Performs basic email format validation (must contain "@" and ".").  
      - Throws error with detailed messages if validation fails.  
    - Inputs: JSON from webhook node.  
    - Outputs: Cleaned JSON payload forwarded if valid.  
    - Failure Modes: Validation error triggers workflow failure with clear error messages.

  - **Verifi Email**  
    - Type: VerifiEmail node (third-party email verification service)  
    - Role: Checks email deliverability and whether the email is disposable or fake.  
    - Configuration: Uses verified API credentials from VerifiEmail service.  
    - Inputs: Email from the validated payload.  
    - Outputs: JSON containing verification result with a boolean field `valid`.  
    - Failure Modes: API authentication errors, service timeouts, invalid email input.

  - **Email Valid? (IF Node)**  
    - Type: IF (conditional branching)  
    - Role: Filters workflow continuation based on email validity.  
    - Configuration: Checks if `valid == true` from Verifi Email output.  
    - Inputs: Verification result JSON.  
    - Outputs:  
      - True branch: proceeds with badge generation.  
      - False branch: ends or optionally routes elsewhere (not implemented).  
    - Failure Modes: Expression evaluation errors, missing `valid` field.

---

#### 2.2 Badge ID & QR Code Generation

- **Overview:**  
  Generates a unique badge ID, verification URL linked to the badge, formatted issue date, and a QR code URL that encodes the verification link.

- **Nodes Involved:**  
  - Generate Badge ID & URL (Function)

- **Node Details:**

  - **Generate Badge ID & URL**  
    - Type: Function  
    - Role: Creates a unique identifier for the participant's badge and associated URLs.  
    - Configuration:  
      - Badge ID format: `"HACK-<Year>-<UnixTimestamp>-<6-char random uppercase>"`  
      - Verification URL: `https://yourdomain.com/verify-badge?id=<badgeId>`  
      - Issue date formatted as "Month day, year" (e.g., January 1, 2024).  
      - QR code generated via external API `https://api.qrserver.com/v1/create-qr-code/` embedding the verification URL.  
    - Inputs: Valid participant data JSON.  
    - Outputs: JSON extended with `badgeId`, `verifyUrl`, `issuedAt`, and `qrCodeUrl`.  
    - Failure Modes: Date/time errors unlikely, external QR code API usage is just URL generation (no external call), so minimal failure risk.

---

#### 2.3 Badge Rendering & PDF Creation

- **Overview:**  
  Converts participant data into a visually styled HTML badge and converts it to PDF format using a third-party API.

- **Nodes Involved:**  
  - HTML to PDF

- **Node Details:**

  - **HTML to PDF**  
    - Type: HTML/CSS to PDF conversion node (via pdfmunk.com API)  
    - Role: Renders a pre-defined HTML template into a PDF file URL.  
    - Configuration:  
      - HTML template includes event name, participant name, team and role (with fallbacks), issue date, badge ID, and embedded QR code image.  
      - Uses Google Fonts "Inter" and custom CSS for badge styling.  
      - Logo image embedded via URL (`https://yourdomain.com/logo-white.png`).  
      - Credentials: API key for the HTML to PDF service.  
    - Inputs: JSON from badge generation node.  
    - Outputs: JSON containing `pdf_url` for the rendered badge PDF.  
    - Failure Modes: API authentication errors, invalid HTML, network timeouts, PDF generation failures.

---

#### 2.4 Delivery & Logging

- **Overview:**  
  Downloads the generated PDF, sends an email with an inline HTML badge preview and PDF attachment via Gmail OAuth2, logs participant details into Google Sheets, and returns a success response to the webhook caller.

- **Nodes Involved:**  
  - Download PDF (HTTP Request)  
  - Send Badge via Gmail  
  - Log to Sheets (Google Sheets append)  
  - Success Response (Respond to Webhook)

- **Node Details:**

  - **Download PDF**  
    - Type: HTTP Request  
    - Role: Downloads the PDF file from the `pdf_url` returned by HTML to PDF node to attach to email.  
    - Configuration: GET request to `{{ $json.pdf_url }}`. Binary output expected for attachment.  
    - Inputs: JSON with `pdf_url`.  
    - Outputs: Binary data of PDF file.  
    - Failure Modes: Network errors, invalid URL, 404 or access denied.

  - **Send Badge via Gmail**  
    - Type: Gmail node (OAuth2 authenticated)  
    - Role: Sends an email with a polished HTML body previewing the badge and attaches the PDF.  
    - Configuration:  
      - Recipient: Participant email from validated input.  
      - Subject: "Your Official <Event> Badge".  
      - HTML body includes personalized greeting, event name, badge ID, verification link, and instructions to print or show QR at check-in.  
      - PDF attached as binary file from previous node.  
      - OAuth2 credentials for Gmail required.  
    - Inputs: Participant data JSON and binary PDF.  
    - Outputs: Email send status.  
    - Failure Modes: Gmail API authentication errors, quota limits, invalid email addresses, attachment size limits.

  - **Log to Sheets**  
    - Type: Google Sheets append operation (OAuth2)  
    - Role: Logs participant and badge details into a designated Google Sheet for record-keeping.  
    - Configuration:  
      - Appends a row with columns: Name, Role, Team, Email, Event, Status ("Sent"), PDF URL, Badge ID, Timestamp, Verify URL, Issued Date.  
      - Document and sheet specified by IDs.  
      - OAuth2 credentials for Google Sheets required.  
    - Inputs: Combined data from previous nodes.  
    - Outputs: Append status.  
    - Failure Modes: Invalid credentials, API quota exceeded, invalid sheet ID or permissions.

  - **Success Response**  
    - Type: Respond to Webhook  
    - Role: Returns a JSON response signaling successful workflow execution back to the webhook caller.  
    - Configuration: Default success response with no body modifications.  
    - Inputs: Response from logging step.  
    - Outputs: HTTP response to client.  
    - Failure Modes: None typical unless network or webhook misconfiguration.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                                  | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                          |
|---------------------------|------------------------|-------------------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook Trigger           | Webhook                | Receives POST registration data                  | —                           | Validate Input                  | # Hackathon Badge Workflow: Fully automated process overview                                       |
| Validate Input            | Function               | Validates required fields and email format       | Webhook Trigger             | Verifi Email                   | # Validation Group — Validation & Safety: checks input fields and email format                     |
| Verifi Email              | VerifiEmail            | Verifies email deliverability and disposability | Validate Input              | Email Valid?                   | # Validation Group — Validation & Safety: real-time email deliverability check                     |
| Email Valid?              | IF                     | Filters valid emails to continue workflow        | Verifi Email                | Generate Badge ID & URL (true)  | # Validation Group — Validation & Safety: conditional branching based on email validity            |
| Generate Badge ID & URL   | Function               | Creates unique badge ID, verification URL, QR   | Email Valid? (true branch)  | HTML to PDF                    | # Badge Creation Group: unique ID, QR code, issue date generation                                  |
| HTML to PDF               | HTML → PDF Conversion  | Converts HTML badge to PDF                        | Generate Badge ID & URL     | Download PDF                   | # Badge Creation Group: renders styled badge HTML to PDF                                           |
| Download PDF              | HTTP Request           | Downloads PDF file for email attachment          | HTML to PDF                 | Send Badge via Gmail           | # Delivery & Logging Group: fetches PDF for email attachment                                       |
| Send Badge via Gmail      | Gmail                  | Sends personalized email with badge attached     | Download PDF                | Log to Sheets                  | # Delivery & Logging Group: sends inline email + PDF attachment                                    |
| Log to Sheets             | Google Sheets          | Logs participant and badge data                   | Send Badge via Gmail        | Success Response               | # Delivery & Logging Group: logs details to Google Sheets                                          |
| Success Response          | Respond to Webhook     | Sends success confirmation to webhook caller     | Log to Sheets               | —                             | # Delivery & Logging Group: final response confirming workflow completion                           |
| Sticky Note               | Sticky Note            | Credential requirements                           | —                           | —                             | Credentials required: VerifiEmail, HTML→PDF service (pdfmunk.com), Gmail OAuth2, Google Sheets OAuth2 |
| Sticky Note1              | Sticky Note            | Workflow overview summary                         | —                           | —                             | Workflow summary explaining full badge generation and delivery process                            |
| Sticky Note2              | Sticky Note            | Validation block explanation                      | —                           | —                             | Validation & safety details: input validation, email verification, conditional branching          |
| Sticky Note3              | Sticky Note            | Badge creation block explanation                  | —                           | —                             | Badge creation details: unique ID, QR code, PDF generation                                       |
| Sticky Note4              | Sticky Note            | Delivery and logging block explanation            | —                           | —                             | Delivery and logging details: PDF download, email, Google Sheets logging, success response        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `hackathon-badge`  
   - Purpose: Entry point to receive JSON registration data.

2. **Create Function Node "Validate Input"**  
   - Connect from Webhook Trigger  
   - Code:  
     - Extract `body` from webhook input.  
     - Assert presence of `name`, `email`, and `event` fields.  
     - Validate email format contains "@" and ".".  
     - Throw error if invalid.  
     - Return cleaned payload JSON.

3. **Create Verifi Email Node**  
   - Connect from Validate Input  
   - Configuration:  
     - Email expression: `={{ $json.email }}`  
     - Use VerifiEmail API credentials (register at verifi.email).  
   - Purpose: Verify email deliverability and check for disposable emails.

4. **Create IF Node "Email Valid?"**  
   - Connect from Verifi Email  
   - Condition: Boolean check if `{{$json.valid}}` is `true`  
   - True branch: proceed to badge generation  
   - False branch: end or handle invalid emails (optional).

5. **Create Function Node "Generate Badge ID & URL"**  
   - Connect from Email Valid? True output  
   - Code:  
     - Generate badge ID in format: `"HACK-<Year>-<UnixTimestamp>-<6 uppercase chars>"`  
     - Construct verification URL: `https://yourdomain.com/verify-badge?id=<badgeId>`  
     - Format issue date (e.g., January 1, 2024).  
     - Generate QR code URL using `https://api.qrserver.com/v1/create-qr-code/?size=300x300&format=png&data=<encoded verifyUrl>`.  
     - Return extended JSON with these fields.

6. **Create HTML to PDF Node**  
   - Connect from Generate Badge ID & URL  
   - Configuration:  
     - API Credentials for HTML to PDF service (e.g., pdfmunk.com)  
     - HTML template: include event name, participant name, team, role, issue date, badge ID, and QR code image URL.  
     - Use embedded CSS and Google Fonts for styling.  
   - Output: JSON with `pdf_url`.

7. **Create HTTP Request Node "Download PDF"**  
   - Connect from HTML to PDF  
   - Method: GET  
   - URL: `={{ $json.pdf_url }}`  
   - Purpose: Download PDF binary data for email attachment.

8. **Create Gmail Node "Send Badge via Gmail"**  
   - Connect from Download PDF  
   - Configuration:  
     - Recipient: `={{ $('Validate Input').item.json.email }}`  
     - Subject: `Your Official {{ $('Validate Input').item.json.event }} Badge`  
     - HTML Body: Personalized email with greeting, event info, badge ID, verify link, and instructions.  
     - Attachments: PDF binary from Download PDF node.  
     - Use Gmail OAuth2 credentials.

9. **Create Google Sheets Node "Log to Sheets"**  
   - Connect from Send Badge via Gmail  
   - Operation: Append row  
   - Document ID and Sheet Name: set to your Google Sheet document and sheet.  
   - Columns and Mappings:  
     - Name, Role, Team, Email, Event, Status ("Sent"), PDF URL, Badge ID, Timestamp (current), Verify URL, Issued Date.  
   - Use Google Sheets OAuth2 credentials.

10. **Create Respond to Webhook Node "Success Response"**  
    - Connect from Log to Sheets  
    - Configuration: Default success response to confirm workflow completion.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                             |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Credentials required for full operation: VerifiEmail (email verification), HTML→PDF (pdfmunk.com), Gmail OAuth2, Google Sheets OAuth2 | See Sticky Note on credentials in workflow                  |
| Workflow uses QR code generation via https://api.qrserver.com, no external calls for QR image needed | QR code URL generated by function node                      |
| HTML to PDF uses Google Fonts "Inter" for consistent and modern typography                           | https://fonts.google.com/specimen/Inter                     |
| Gmail OAuth2 must be configured with proper scopes to send emails with attachments                  | https://developers.google.com/identity/protocols/oauth2     |
| Google Sheets API requires OAuth2 credentials with write permissions for logging participant data   | https://developers.google.com/sheets/api/guides/authorizing |
| The verification URL placeholder domain `yourdomain.com` must be replaced with your actual domain    | URL used in badge ID generation and QR code                  |

---

**Disclaimer:**  
The provided workflow text is fully generated and configured using n8n automation platform. It complies with content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.

---