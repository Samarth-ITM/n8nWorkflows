Generate Personalized Promo Cards with Email Verification, Gmail & Google Sheets

https://n8nworkflows.xyz/workflows/generate-personalized-promo-cards-with-email-verification--gmail---google-sheets-9032


# Generate Personalized Promo Cards with Email Verification, Gmail & Google Sheets

### 1. Workflow Overview

This workflow automates the generation and distribution of personalized promotional discount cards to users who sign up via an API webhook. It ensures data sanitization and email verification to prevent fraud and maintain list quality, creates a visually appealing promo card image, sends a customized email with the promo card attached, and logs all successful distributions to Google Sheets. Invalid email attempts are handled with error logging and admin notifications.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Data Sanitization:** Receive user signup data via webhook, clean and normalize it, and prepare for validation.
- **1.2 Email Verification:** Validate the user email address using a dedicated verification API.
- **1.3 Validation Gateway:** Decide the processing path based on email validity.
- **1.4 Promo Card Generation and Email Delivery:** Generate a personalized promo card image, send it via Gmail to the user, and log the distribution.
- **1.5 Error Handling and Notification:** For invalid emails, prepare error data, send alert emails to admins, and log errors.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Sanitization

**Overview:**  
This block receives incoming promo signup data from an external POST webhook, sanitizes and normalizes the data, sets defaults for missing fields, and timestamps the request for tracking.

**Nodes Involved:**  
- WEBHOOK  
- Edit Fields  

**Node Details:**

- **WEBHOOK**  
  - Type: Webhook  
  - Role: Entry point for the workflow; receives POST requests with promo signup data.  
  - Configuration:  
    - Path: `/promo-signup`  
    - HTTP Method: POST  
    - Response: Static JSON confirmation `{ "status": "success", "message": "Promo card being generated!" }`  
  - Inputs: External HTTP POST request with JSON body containing fields like `name`, `email`, `promo_code`, `discount_value`.  
  - Outputs: Passes JSON data downstream.  
  - Edge Cases: Missing or malformed requests; webhook errors.

- **Edit Fields**  
  - Type: Set Node  
  - Role: Data sanitation and normalization.  
  - Configuration:  
    - Sets `name` with fallback to `"Valued Customer"` if missing.  
    - Copies `email` as is.  
    - Sets `promo_code` default `"WELCOME10"` if missing.  
    - Sets `discount_value` default `"10%"` if missing.  
    - Sets `discount_type` default `"percentage"`.  
    - Adds current timestamp in ISO format.  
  - Expressions: Uses `$json.body` to access incoming data, with fallback defaults.  
  - Inputs: From WEBHOOK node.  
  - Outputs: Structured and enriched JSON data for validation step.  
  - Edge Cases: Missing fields handled gracefully by defaults; timestamp always added.

---

#### 2.2 Email Verification

**Overview:**  
This block validates the sanitized email address to ensure it is syntactically correct, exists, is not disposable, and is deliverable, preventing fraudulent or invalid promo distribution.

**Nodes Involved:**  
- Validate Email  

**Node Details:**

- **Validate Email**  
  - Type: VerifiEmail (third-party verification API)  
  - Role: Verify the authenticity and validity of the user email.  
  - Configuration:  
    - Email input expression: `={{ $json.email }}`  
    - Credentials: Requires VerifiEmail API key.  
  - Inputs: From Edit Fields node with sanitized data.  
  - Outputs: JSON with email validation results, including a boolean `valid` field.  
  - Edge Cases: API failures (timeouts, invalid API key), emails flagged as disposable or invalid, network errors.

---

#### 2.3 Validation Gateway

**Overview:**  
This decision block routes workflow execution based on the email validation result. Valid emails proceed to promo card generation, invalid emails trigger error handling.

**Nodes Involved:**  
- Validation Gateway  

**Node Details:**

- **Validation Gateway**  
  - Type: IF Node  
  - Role: Branching based on email validity.  
  - Configuration:  
    - Condition: Check if `$json.valid === true`  
  - Inputs: From Validate Email node.  
  - Outputs:  
    - TRUE output: Valid emails → Generate Promo Card Image  
    - FALSE output: Invalid emails → Set Error Data  
  - Edge Cases: Missing `valid` field; ensure strict boolean comparison.

---

#### 2.4 Promo Card Generation and Email Delivery

**Overview:**  
This block generates a personalized promo card image using HTML and CSS, sends a customized promotional email with the image attachment to the valid user, and logs the successful distribution to Google Sheets.

**Nodes Involved:**  
- Generate Promo Card Image  
- Success Path (Gmail)  
- Log Promo Distribution (Google Sheets)  

**Node Details:**

- **Generate Promo Card Image**  
  - Type: HTML/CSS to Image node (HTMLCSSToImage API)  
  - Role: Create a visually appealing PNG promo card from HTML template.  
  - Configuration:  
    - HTML includes placeholders for `name`, `promo_code`, `discount_value` using expressions like `{{ $('Edit Fields').item.json.name }}`.  
    - QR code generated dynamically linking to checkout URL with promo code.  
    - Style: Gradient background, discount circle, QR code, footer with expiry date.  
    - Output format: PNG image.  
    - Credentials: Requires HTMLCSSToImage API key.  
  - Inputs: From Validation Gateway TRUE output (validated user data).  
  - Outputs: Image URL or base64 image data passed downstream.  
  - Edge Cases: API quota limits, malformed HTML, image generation errors.

- **Success Path (Gmail node)**  
  - Type: Gmail  
  - Role: Send personalized HTML email with promo card image embedded.  
  - Configuration:  
    - Recipient: User email from `Edit Fields`.  
    - Subject: Includes discount value dynamically.  
    - Message: Rich HTML content with a warm welcome, promo code display, features, CTA button, and embedded promo card image using `{{ $json.image_url }}`.  
    - Credentials: Gmail OAuth2 API configured.  
  - Inputs: From Generate Promo Card Image node with image data.  
  - Outputs: Email delivery status.  
  - Edge Cases: Gmail API limits, OAuth token expiration, invalid email addresses (should be rare here), HTML rendering issues.

- **Log Promo Distribution (Google Sheets)**  
  - Type: Google Sheets  
  - Role: Append a new row logging the promo distribution event.  
  - Configuration:  
    - Document ID and Sheet name configured.  
    - Columns: Timestamp, Name, Email, Status ("Sent Successfully"), Promo Code, Discount Value.  
    - Credentials: Google Sheets OAuth2 API configured.  
  - Inputs: From Success Path node after email sent.  
  - Outputs: Confirmation of logging.  
  - Edge Cases: API limits, permission errors, sheet access issues.

---

#### 2.5 Error Handling and Notification

**Overview:**  
This block handles invalid email cases by preparing error details, sending an alert email to administrators, and preventing promo distribution.

**Nodes Involved:**  
- Set Error Data  
- Error Path (Gmail node)  

**Node Details:**

- **Set Error Data**  
  - Type: Set Node  
  - Role: Create structured error information for invalid email attempts.  
  - Configuration:  
    - Sets `error_type` = "Invalid Email"  
    - Copies `email` from Edit Fields  
    - Adds current timestamp  
  - Inputs: From Validation Gateway FALSE output.  
  - Outputs: Error object passed to notification node.  
  - Edge Cases: Missing email data; fallback not defined.

- **Error Path (Gmail node)**  
  - Type: Gmail  
  - Role: Notify system admin about invalid promo card generation attempt.  
  - Configuration:  
    - Recipient: Admin email (placeholder `YOUR_ADMIN_EMAIL_HERE`).  
    - Subject: "⚠️ Failed Promo Card Generation"  
    - Message: Text email including failed email address.  
    - Credentials: Gmail OAuth2 API configured.  
  - Inputs: From Set Error Data node.  
  - Outputs: Email delivery status for alert.  
  - Edge Cases: Admin email misconfiguration, Gmail API issues.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                           | Input Node(s)        | Output Node(s)             | Sticky Note                                                                                                      |
|-------------------------|--------------------------------|------------------------------------------|----------------------|----------------------------|------------------------------------------------------------------------------------------------------------------|
| WEBHOOK                 | Webhook                       | Receive promo signup POST request        | -                    | Edit Fields                | ## PROMO SIGNUP TRIGGER - API: Promo code signup webhook - Method: POST - Response: 200 OK with confirmation      |
| Edit Fields             | Set                           | Sanitize and set default input fields    | WEBHOOK              | Validate Email             | ## DATA SANITIZATION - Trim, defaults, timestamp, normalize discount format                                     |
| Validate Email          | VerifiEmail API               | Validate email address                    | Edit Fields          | Validation Gateway         | ## EMAIL VALIDATION - Syntax, domain, mailbox, disposable detection                                             |
| Validation Gateway      | IF                            | Route workflow based on email validity   | Validate Email       | Generate Promo Card Image (TRUE), Set Error Data (FALSE) | ## VALIDATION GATEWAY - Route valid vs invalid emails                                                          |
| Generate Promo Card Image | HTML/CSS to Image             | Generate personalized promo card image   | Validation Gateway   | Success Path               | ## PROMO CARD GENERATION - HTML to PNG, QR code, gradient design                                               |
| Success Path            | Gmail                         | Send promo email with promo card attached| Generate Promo Card Image | Log Promo Distribution    | ## CUSTOMER EMAIL DELIVERY - Personalized email with promo card and instructions                               |
| Log Promo Distribution  | Google Sheets                 | Log successful promo email distribution  | Success Path         | -                          | ## ANALYTICS TRACKING - Log promo details for reporting                                                        |
| Set Error Data          | Set                           | Prepare error data for invalid emails    | Validation Gateway   | Error Path                 | ## ERROR LOGGING - Capture invalid email attempts                                                             |
| Error Path              | Gmail                         | Notify admin of invalid promo attempt    | Set Error Data       | -                          | ## ADMIN NOTIFICATION - Alert team of invalid attempts                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Set path as `promo-signup`  
   - HTTP Method: POST  
   - Response Data: Static JSON `{"status": "success", "message": "Promo card being generated!"}`

2. **Create Set Node "Edit Fields":**  
   - Connect from Webhook node.  
   - Assign fields:  
     - `name`: expression `{{$json.body.name || 'Valued Customer'}}`  
     - `email`: expression `{{$json.body.email}}`  
     - `promo_code`: expression `{{$json.body.promo_code || 'WELCOME10'}}`  
     - `discount_value`: expression `{{$json.body.discount_value || '10%'}}`  
     - `discount_type`: `"percentage"` (default)  
     - `timestamp`: `{{ new Date().toISOString() }}`

3. **Create Email Validation Node "Validate Email":**  
   - Connect from "Edit Fields".  
   - Use VerifiEmail node or equivalent email validation node.  
   - Configure with API key credential.  
   - Set input email expression: `{{$json.email}}`.

4. **Create IF Node "Validation Gateway":**  
   - Connect from "Validate Email".  
   - Condition: `$json.valid === true` (boolean)  
   - TRUE branch leads to promo card generation.  
   - FALSE branch leads to error handling.

5. **Create HTML to Image Node "Generate Promo Card Image":**  
   - Connect from TRUE output of "Validation Gateway".  
   - Use HTMLCSSToImage API node.  
   - Provide HTML template with CSS for promo card design.  
   - Use expressions to insert fields like `name`, `promo_code`, `discount_value`.  
   - Include QR code image URL with embedded promo code link.  
   - Configure credentials with HTMLCSSToImage API key.

6. **Create Gmail Node "Success Path":**  
   - Connect from "Generate Promo Card Image".  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient as `{{$('Edit Fields').item.json.email}}`.  
   - Subject: `🎉 Your Exclusive {{$('Edit Fields').item.json.discount_value}} Discount is Ready!`  
   - Message: Rich HTML body referencing user name, promo code, and embed promo card image URL `{{$json.image_url}}`.

7. **Create Google Sheets Node "Log Promo Distribution":**  
   - Connect from "Success Path".  
   - Configure Google Sheets OAuth2 credentials.  
   - Set document ID and sheet name (e.g., `gid=0`).  
   - Append row with columns: Timestamp, Name, Email, Status ("Sent Successfully"), Promo Code, Discount Value.

8. **Create Set Node "Set Error Data":**  
   - Connect from FALSE output of "Validation Gateway".  
   - Set fields:  
     - `error_type`: "Invalid Email"  
     - `email`: `{{$('Edit Fields').item.json.email}}`  
     - `timestamp`: current ISO timestamp

9. **Create Gmail Node "Error Path":**  
   - Connect from "Set Error Data".  
   - Configure Gmail OAuth2 credentials.  
   - Recipient: Admin email (replace placeholder).  
   - Subject: "⚠️ Failed Promo Card Generation"  
   - Message (plain text): Include failed email address.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Verifi Email API key required for email validation. Get API key: https://verifi.email | Email Verification Setup |
| HTMLCSSToImage API key required for generating promo card images. Get API key: https://htmlcsstoimg.com | Promo Card Image Generation |
| Gmail API OAuth2 credentials needed for sending emails. Google Cloud Console: https://console.cloud.google.com/ | Email Delivery Setup |
| Google Sheets API credentials needed for logging. Enable Sheets API and create Service Account in Google Cloud Console. | Analytics Logging Setup |
| Test webhook URL format example: https://your-n8n-instance.com/webhook/promo-signup | Workflow Trigger Testing |
| Processing time per request approximately 30-60 seconds depending on API response times. | Performance Expectation |
| Monitor API usage limits and email delivery metrics regularly to maintain high success rate. | Operational Monitoring |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow. All processes comply with current content policies and involve only legal and public data.