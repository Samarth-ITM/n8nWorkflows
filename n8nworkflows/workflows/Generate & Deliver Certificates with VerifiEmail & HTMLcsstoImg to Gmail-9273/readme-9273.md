Generate & Deliver Certificates with VerifiEmail & HTMLcsstoImg to Gmail

https://n8nworkflows.xyz/workflows/generate---deliver-certificates-with-verifiemail---htmlcsstoimg-to-gmail-9273


# Generate & Deliver Certificates with VerifiEmail & HTMLcsstoImg to Gmail

### 1. Workflow Overview

This workflow automates the generation and delivery of personalized course completion certificates via email, incorporating email validation, certificate creation, image conversion, email sending, and logging. It is designed for educational platforms or training providers who want to automate certificate issuance upon course completion.

Logical blocks:

- **1.1 Input Reception:** Receives certificate request data via a webhook.
- **1.2 Email Validation:** Validates the recipient's email address using VerifiEmail API.
- **1.3 Validation Check:** Ensures all required fields and email validation succeed.
- **1.4 Data Combination:** Merges webhook data with validation results into a structured object.
- **1.5 HTML Certificate Generation:** Creates a styled HTML certificate using a JavaScript code node.
- **1.6 Image Conversion:** Converts the HTML certificate to a PNG image via HTMLcsstoImg API.
- **1.7 Email Delivery:** Sends the certificate image and details to the recipient via Gmail OAuth2.
- **1.8 Logging:** Records certificate issuance details in Google Sheets for auditing.
- **1.9 Error Handling:** Catches workflow errors, formats error info, and optionally sends Slack alerts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives POST requests at a webhook containing certificate information.
- **Nodes Involved:**  
  - Certificate Request Webhook (Webhook Node)  
  - Webhook Info (Sticky Note)
- **Node Details:**

  - **Certificate Request Webhook**  
    - Type: Webhook  
    - Role: Entry point triggering workflow on HTTP POST  
    - Configuration: Path `"certificate-generator"`, method POST, response mode "lastNode"  
    - Inputs: External HTTP requests with JSON body  
    - Outputs: JSON with fields like name, course, date, email, optional instructor, duration, certificateId  
    - Edge cases: Missing or malformed JSON, invalid HTTP method  
    - Sticky notes describe expected JSON schema and webhook URL.
  
  - **Webhook Info** (Sticky Note)  
    - Contains detailed instructions and example JSON for the webhook input format.

#### 1.2 Email Validation

- **Overview:** Validates recipient email using VerifiEmail API for format correctness, MX records, disposable email detection, and spoof detection.
- **Nodes Involved:**  
  - Verifi Email (VerifiEmail Node)  
  - Email Validation Info (Sticky Note)
- **Node Details:**

  - **Verifi Email**  
    - Type: VerifiEmail Node  
    - Role: Calls VerifiEmail API to validate the email address from webhook data  
    - Configuration: Uses email from webhook `{{$json.body.email}}`  
    - Credentials: Requires configured VerifiEmail API key  
    - Outputs: Validation result including boolean `valid`, provider info, MX record validity, disposable flag  
    - Edge cases: API failure, invalid or disposable emails, quota exceeded  
    - Next step depends on validation success.
  
  - **Email Validation Info** (Sticky Note)  
    - Explains validation checks performed and output fields.

#### 1.3 Validation Check

- **Overview:** Checks that required fields (name, course, date) are present and email validation passed; halts workflow if not.
- **Nodes Involved:**  
  - If (IF Node)  
  - Validation Logic (Sticky Note)  
  - Stop and Error (Stop Node)
- **Node Details:**

  - **If**  
    - Type: IF Node  
    - Role: Conditional gate to allow workflow continuation only if all conditions pass  
    - Conditions:  
      - name not empty  
      - course not empty  
      - date not empty  
      - email validation `valid` is true  
    - Inputs: webhook data and VerifiEmail output  
    - Outputs:  
      - True: continue data processing  
      - False: trigger Stop and Error  
    - Edge cases: expression errors if input data missing or malformed.
  
  - **Validation Logic** (Sticky Note)  
    - Details the exact conditions checked in the IF node.
  
  - **Stop and Error**  
    - Type: Stop Node  
    - Role: Terminates workflow with error message if validation fails  
    - Configuration: Custom error message "Missing required fields: name, course, date, or valid email"  
    - Outputs: Stops workflow execution  
    - Edge cases: None (standard stop behavior)

#### 1.4 Data Combination

- **Overview:** Combines original webhook data and email validation results into a single structured JSON object for downstream processing.
- **Nodes Involved:**  
  - Code in JavaScript (Code Node)  
  - Data Combination Info (Sticky Note)
- **Node Details:**

  - **Code in JavaScript**  
    - Type: Code Node (JavaScript)  
    - Role: Merges webhook body fields and VerifiEmail validation data into one object  
    - Logic: Extracts webhook body (`$('Certificate Request Webhook').first().json.body`) and email validation data from previous node; generates default instructor and duration if missing  
    - Outputs: Combined JSON including certificateId (empty string if missing), emailValidation subobject with detailed validation info  
    - Edge cases: Missing webhook data, unexpected data format causing runtime errors  
    - Input: IF node true branch output  
    - Output: To HTML generation node.
  
  - **Data Combination Info** (Sticky Note)  
    - Explains input sources and output structure, including auto-generation of certificateId if missing.

#### 1.5 HTML Certificate Generation

- **Overview:** Generates a professional, styled HTML certificate with embedded CSS and dynamic content for the recipient.
- **Nodes Involved:**  
  - Generate HTML Certificate (Code Node)  
  - HTML Generation Info (Sticky Note)
- **Node Details:**

  - **Generate HTML Certificate**  
    - Type: Code Node (JavaScript)  
    - Role: Creates full HTML document string with inline CSS and dynamic placeholders replaced by input data  
    - Features:  
      - Purple gradient background  
      - Playfair Display and Montserrat fonts via Google Fonts  
      - Certificate data: recipient name, course, completion date (formatted), instructor (default "Program Director"), duration (optional), certificateId (auto-generated if missing)  
      - Includes decorative gold badge and styles for signature, date, and footer  
    - Outputs: JSON with original data plus generated certificateId, formattedDate, and full HTML string `html`  
    - Edge cases: Date parsing errors, missing fields, special characters in names requiring escaping  
    - Input: Combined data from previous node  
    - Output: HTML/CSS to Image node.
  
  - **HTML Generation Info** (Sticky Note)  
    - Describes design features, dynamic content, and output.

#### 1.6 Image Conversion

- **Overview:** Converts the generated HTML certificate into a high-quality PNG image hosted on HTMLcsstoImg servers.
- **Nodes Involved:**  
  - HTML/CSS to Image (HTMLcsstoimage Node)  
  - Image Conversion Info (Sticky Note)
- **Node Details:**

  - **HTML/CSS to Image**  
    - Type: HTMLcsstoimage Node  
    - Role: Sends HTML content to third-party API to return PNG image URL  
    - Input: HTML content from previous node via expression `{{$json.html}}`  
    - Credentials: Requires HTMLcsstoImg User ID and API Key  
    - Outputs: JSON with `image_url` (permanent PNG URL) and `image_id`  
    - Edge cases: API rate limits, invalid HTML causing rendering failure, network issues  
    - Output feeds email delivery node.
  
  - **Image Conversion Info** (Sticky Note)  
    - Details API usage, output format, and credential requirements.

#### 1.7 Email Delivery

- **Overview:** Sends a formatted congratulatory email with certificate link to the recipient using Gmail OAuth2 credentials.
- **Nodes Involved:**  
  - Send Certificate Email (Gmail Node)  
  - Email Delivery Info (Sticky Note)
- **Node Details:**

  - **Send Certificate Email**  
    - Type: Gmail Node  
    - Role: Sends the email with HTML body to recipient  
    - Configuration:  
      - To: recipient email from earlier nodes `={{ $('Code in JavaScript').item.json.email }}`  
      - Subject: "🎓 Congratulations! Your Certificate of Completion"  
      - Message: Rich HTML template referencing certificate details, formatted date, download button linking to PNG image URL from HTML/CSS to Image node, and call-to-action to share on LinkedIn  
    - Credentials: Gmail OAuth2 (requires OAuth2 setup with Google)  
    - Edge cases: OAuth token expiration, incorrect recipient email, Gmail API quota limits  
    - Output: Logs node input.
  
  - **Email Delivery Info** (Sticky Note)  
    - Explains email content structure, variables used, and credential requirements.

#### 1.8 Logging

- **Overview:** Appends or updates a row in a Google Sheets spreadsheet to log certificate issuance details for auditing and analytics.
- **Nodes Involved:**  
  - Log to Google Sheets (Google Sheets Node)  
  - Database Logging Info (Sticky Note)
- **Node Details:**

  - **Log to Google Sheets**  
    - Type: Google Sheets Node  
    - Role: Appends or updates certificate info in "Certificates Log" spreadsheet, "Sheet1"  
    - Configuration:  
      - Columns: Certificate ID, Recipient Name, Course, Email, Completion Date, Generated At (current timestamp), Certificate URL, Status ("Sent"), Instructor, Duration  
      - Matching Column: Certificate ID (for update vs append)  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Spreadsheet access denied, quota exceeded, row matching issues  
    - Output: Ends main workflow chain.
  
  - **Database Logging Info** (Sticky Note)  
    - Details spreadsheet structure, logging fields, and purpose.

#### 1.9 Error Handling

- **Overview:** Captures workflow errors, formats error details, and optionally sends Slack notifications to alert the admin team.
- **Nodes Involved:**  
  - On Workflow Error (Error Trigger Node)  
  - Format Error Details (Code Node)  
  - Send Slack Alert (Slack Node, disabled)  
  - Error Handling Info (Sticky Note)
- **Node Details:**

  - **On Workflow Error**  
    - Type: Error Trigger Node  
    - Role: Automatically triggers if any workflow error occurs  
    - Outputs: Passes error details to formatting node  
    - Edge cases: Captures all errors including node failures and execution exceptions.
  
  - **Format Error Details**  
    - Type: Code Node  
    - Role: Extracts error message, failed node name, timestamp, workflow and execution IDs, and user input data (name, email, course) for reporting  
    - Outputs: Formatted JSON for alerting
    
  - **Send Slack Alert**  
    - Type: Slack Node (disabled by default)  
    - Role: Sends formatted error notification to Slack channel with details including error, user info, execution id  
    - Credentials: Slack API token (required if enabled)  
    - Edge cases: Slack API downtime or invalid credentials  
    - Note: Disabled; user may enable by configuring Slack credentials.
  
  - **Error Handling Info** (Sticky Note)  
    - Explains error flow, optional Slack alerts, and recommendation to add error response node.

---

### 3. Summary Table

| Node Name               | Node Type                | Functional Role                          | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                   |
|-------------------------|--------------------------|----------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Credentials Setup Guide  | Sticky Note              | Credential Setup Instructions          |                             |                             | Explains all required credentials setup for VerifiEmail, HTMLcsstoImg, Gmail OAuth2, Google Sheets OAuth2                       |
| Webhook Info            | Sticky Note              | Webhook Request Format Reference       |                             |                             | Describes expected JSON input for webhook and webhook URL                                                                     |
| Certificate Request Webhook | Webhook Node            | Receives certificate request           |                             | Verifi Email                |                                                                                                                              |
| Email Validation Info    | Sticky Note              | Email Validation Details               |                             |                             | Describes VerifiEmail checks and output fields                                                                                |
| Verifi Email            | VerifiEmail Node          | Validate recipient email                | Certificate Request Webhook | If                          | Credential required: VerifiEmail API                                                                                          |
| Validation Logic        | Sticky Note              | Validation criteria                     |                             |                             | Lists conditions checked in IF node                                                                                           |
| If                      | IF Node                  | Checks required fields and email valid | Verifi Email                | Code in JavaScript, Stop and Error |                                                                                                                              |
| Stop and Error          | Stop Node                | Stops workflow on validation failure   | If (False branch)           |                             | Custom error message: "Missing required fields: name, course, date, or valid email"                                           |
| Data Combination Info   | Sticky Note              | Merging webhook and email validation data |                             |                             | Describes merged JSON structure                                                                                                |
| Code in JavaScript      | Code Node                | Combines webhook and email validation data | If (True branch)            | Generate HTML Certificate    |                                                                                                                              |
| HTML Generation Info    | Sticky Note              | HTML certificate design details        |                             |                             | Describes design and dynamic content                                                                                          |
| Generate HTML Certificate | Code Node                | Generates full HTML certificate         | Code in JavaScript          | HTML/CSS to Image            |                                                                                                                              |
| Image Conversion Info   | Sticky Note              | HTML to PNG image conversion details    |                             |                             | Describes HTMLcsstoImg API usage and output                                                                                   |
| HTML/CSS to Image       | HTMLcsstoimage Node       | Converts HTML to PNG image               | Generate HTML Certificate   | Send Certificate Email       | Credential required: HTMLcsstoImg API                                                                                         |
| Email Delivery Info     | Sticky Note              | Email content and delivery details      |                             |                             | Describes Gmail email content and credential requirements                                                                     |
| Send Certificate Email  | Gmail Node                | Sends certificate email to recipient    | HTML/CSS to Image           | Log to Google Sheets         | Credential required: Gmail OAuth2                                                                                             |
| Database Logging Info   | Sticky Note              | Google Sheets logging details           |                             |                             | Describes spreadsheet columns and logging purpose                                                                             |
| Log to Google Sheets    | Google Sheets Node        | Logs certificate issuance data          | Send Certificate Email      |                             | Credential required: Google Sheets OAuth2                                                                                     |
| Success Response Info   | Sticky Note              | Recommends adding webhook response node |                             |                             | Advises adding "Respond to Webhook" node to confirm success to API caller                                                    |
| Error Handling Info     | Sticky Note              | Describes error catching and alerts     |                             |                             | Explains error flow, Slack alert (disabled), and response recommendation                                                    |
| On Workflow Error       | Error Trigger Node        | Triggers on workflow errors             |                             | Format Error Details         |                                                                                                                              |
| Format Error Details    | Code Node                | Formats error details                    | On Workflow Error           | Send Slack Alert             |                                                                                                                              |
| Send Slack Alert        | Slack Node (disabled)    | Sends error notifications via Slack     | Format Error Details        |                             | Optional: enable Slack integration for error alerts                                                                          |
| Workflow Overview       | Sticky Note              | Summarizes entire workflow structure    |                             |                             |                                                                                                                              |
| Testing Instructions    | Sticky Note              | Describes testing process and endpoint  |                             |                             |                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Certificate Request Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `certificate-generator`  
   - Response Mode: Last Node  
   - No credentials needed  
   - This node triggers on receiving JSON with fields: name, course, date, email, optional instructor, duration, certificateId.

2. **Add VerifiEmail Node: "Verifi Email"**  
   - Type: VerifiEmail Node  
   - Email: Expression → `{{$json.body.email}}` from webhook  
   - Credentials: Configure VerifiEmail API key (sign up at https://verifi.email)  
   - Connect webhook output to this node.

3. **Add IF Node: "If"**  
   - Type: IF Node (version 2)  
   - Conditions (AND logic):  
     - Expression: `={{ $('Certificate Request Webhook').item.json.body.name }}` not empty  
     - Expression: `={{ $('Certificate Request Webhook').item.json.body.course }}` not empty  
     - Expression: `={{ $('Certificate Request Webhook').item.json.body.date }}` not empty  
     - Expression: `={{ $json.valid }}` equals true (from VerifiEmail)  
   - Connect VerifiEmail output to IF node input.

4. **Add Stop Node: "Stop and Error"**  
   - Type: Stop Node  
   - Error Message: `"Missing required fields: name, course, date, or valid email"`  
   - Connect IF node "False" output to this node.

5. **Add Code Node: "Code in JavaScript"**  
   - Type: Code Node (JavaScript)  
   - Code: Merge webhook body and email validation results into one JSON object.  
   - Use the provided script combining `$input.item.json` and webhook data.  
   - Connect IF node "True" output to this node.

6. **Add Code Node: "Generate HTML Certificate"**  
   - Type: Code Node (JavaScript)  
   - Purpose: Generate full HTML certificate with inline CSS and dynamic content.  
   - Use the provided HTML template with Playfair Display and Montserrat fonts, purple gradient background, gold badge, and dynamic certificate data.  
   - Connect "Code in JavaScript" output to this node.

7. **Add HTMLcsstoimage Node: "HTML/CSS to Image"**  
   - Type: HTMLcsstoimage Node  
   - HTML Content: Expression `={{ $json.html }}` from previous node  
   - Credentials: Configure HTMLcsstoImg User ID and API Key (https://htmlcsstoimg.com)  
   - Connect "Generate HTML Certificate" output to this node.

8. **Add Gmail Node: "Send Certificate Email"**  
   - Type: Gmail Node  
   - To: Expression `={{ $('Code in JavaScript').item.json.email }}`  
   - Subject: `"🎓 Congratulations! Your Certificate of Completion"`  
   - Message: Use provided HTML email template referencing certificate name, course, formatted date, and download link (`{{$json.image_url}}`)  
   - Credentials: Setup Gmail OAuth2 with Google OAuth flow and necessary scopes  
   - Connect "HTML/CSS to Image" output to this node.

9. **Add Google Sheets Node: "Log to Google Sheets"**  
   - Type: Google Sheets Node  
   - Operation: Append or Update  
   - Spreadsheet: Create "Certificates Log" with columns: Certificate ID, Recipient Name, Course, Email, Completion Date, Generated At, Certificate URL, Status, Instructor, Duration  
   - Match Column: Certificate ID  
   - Map columns using expressions from "Generate HTML Certificate" and "HTML/CSS to Image" nodes, plus current timestamp (`{{ $now.toISO() }}`)  
   - Credentials: Setup Google Sheets OAuth2 with access to your spreadsheet  
   - Connect "Send Certificate Email" output to this node.

10. **Optional: Add Respond to Webhook Node** (Recommended)  
    - After "Log to Google Sheets"  
    - Response Code: 200  
    - Response Body: JSON confirming success with certificateId, recipient email, certificate URL, and generated timestamp  
    - This node completes the webhook response cycle for API consumers.

11. **Add Error Handling Flow:**  
    - Add Error Trigger Node: "On Workflow Error"  
    - Add Code Node: "Format Error Details" to extract and format error info  
    - Optional Slack Node: "Send Slack Alert" (disabled by default) with Slack credentials and alert message template  
    - Recommend adding a Respond to Webhook node after error trigger to return 500 status with error details.

12. **Add Sticky Notes** for guidance at various points to instruct users on credentials, input format, validation logic, etc.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                          |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| Credential setup instructions for VerifiEmail, HTMLcsstoImg, Gmail OAuth2, and Google Sheets OAuth2                                | See "Credentials Setup Guide" sticky note                 |
| Webhook URL and sample JSON payload for testing with cURL or Postman                                                               | See "Webhook Info" and "Testing Instructions" sticky notes |
| Visual and functional design details of the HTML certificate                                                                        | See "HTML Generation Info" sticky note                     |
| Email content includes professional branding, styled with CSS, and encourages sharing on LinkedIn                                   | See "Email Delivery Info" sticky note                       |
| Google Sheets logging provides audit and analytics capabilities with configured columns and matching logic                          | See "Database Logging Info" sticky note                    |
| Recommended addition: Respond to Webhook node to confirm success to external API consumers                                           | See "Success Response Info" sticky note                    |
| Error handling flow includes detailed error data formatting and optional Slack notifications                                         | See "Error Handling Info" sticky note                       |
| Testing instructions include example cURL command and common validation failure scenarios                                            | See "Testing Instructions" sticky note                      |
| Fonts used in certificate and email: Playfair Display and Montserrat via Google Fonts                                                | Integrated in HTML and email CSS                            |
| Workflow designed with n8n version supporting HTMLcsstoimage, VerifiEmail, Gmail OAuth2, and Google Sheets OAuth2 nodes             | Ensure n8n instance supports these node types and versions |

---

_Disclaimer: The provided text originates exclusively from an automated workflow constructed with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible._