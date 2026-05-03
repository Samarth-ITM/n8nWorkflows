Patient Pre-registration System with Email Verification & QR Health Cards using Google Drive

https://n8nworkflows.xyz/workflows/patient-pre-registration-system-with-email-verification---qr-health-cards-using-google-drive-10771


# Patient Pre-registration System with Email Verification & QR Health Cards using Google Drive

---

### 1. Workflow Overview

This workflow automates the **patient pre-registration and check-in process** for a healthcare clinic. It enables patients to submit their appointment and personal details via a webhook, then performs data validation, email verification, and generates a custom digital health card including a QR code for quick check-in. The health card is transformed into a professional PNG image, uploaded to Google Drive, emailed to the patient, and clinic staff are notified via Slack. All patient records are logged in Google Sheets for administrative tracking.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Validation:** Receives patient data via webhook, validates required fields, cleans and formats inputs, and generates a unique patient ID and appointment date/time formats.
- **1.2 Email Verification:** Uses the VerifiEmail API to verify the patient's email address. Stops the workflow on failure.
- **1.3 Health Card Generation:** Creates a verification URL and QR code, builds a styled HTML health card with patient info, converts it into a PNG image.
- **1.4 Google Drive Storage:** Uploads the PNG health card to a designated Google Drive folder with a patient-specific filename and gets a shareable link.
- **1.5 Delivery & Notifications:** Sends the health card email to the patient, notifies the clinic reception team via Slack with patient details, and logs the patient record in Google Sheets. Finally, responds to the webhook confirming success.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

- **Overview:**  
  This block receives patient submission data via a POST webhook, validates required fields, cleans and standardizes input data, generates a unique patient ID, and formats the appointment date/time for display.

- **Nodes Involved:**  
  - Webhook Trigger  
  - Validate Input Data  

- **Node Details:**

  1. **Webhook Trigger**  
     - *Type:* Webhook  
     - *Role:* Entry point receiving patient data via HTTP POST at path `/patient-checkin`.  
     - *Config:* Listens for POST requests, uses response node mode to send response later.  
     - *Input/Output:* No input; outputs raw patient data JSON from the request body.  
     - *Edge Cases:* Invalid or missing request body may cause downstream errors.

  2. **Validate Input Data**  
     - *Type:* Code (JavaScript)  
     - *Role:* Validates required fields `name`, `email`, `appointment_date`, and `appointment_time`.  
     - *Config:*  
       - Throws an error if any required field is missing or empty.  
       - Trims and formats inputs, defaults optional fields: `phone`, `symptoms`, `age`, `gender`.  
       - Generates a unique `patient_id` using timestamp and random string.  
       - Builds ISO datetime `appointment_datetime` and two display formats (`appointment_display` and `appointment_short`) localized to Indian English with 12hr time.  
     - *Expressions Used:* Standard JavaScript with `$input.item.json.body` to extract webhook payload.  
     - *Input:* Raw webhook JSON.  
     - *Output:* Cleaned and enriched patient data for next steps.  
     - *Edge Cases:* Malformed data, missing required fields, date/time format errors.

#### 1.2 Email Verification

- **Overview:**  
  Verifies if the patient’s email is valid and deliverable using the VerifiEmail API. If email is invalid, workflow stops with error.

- **Nodes Involved:**  
  - Verifi Email  
  - Check Email Valid  
  - Stop and Error  

- **Node Details:**

  1. **Verifi Email**  
     - *Type:* VerifiEmail API node  
     - *Role:* Sends the patient email for validation.  
     - *Config:* Uses expression `={{ $json.email }}` from validated patient data.  
     - *Credentials:* Requires VerifiEmail API key configured in n8n credentials.  
     - *Input:* Cleaned patient data with email.  
     - *Output:* API response with email validity info.  
     - *Edge Cases:* API downtime, invalid API keys, network errors.

  2. **Check Email Valid**  
     - *Type:* If node  
     - *Role:* Determines next step based on email verification result.  
     - *Config:* Checks if `valid` field from VerifiEmail API equals `true`.  
     - *Input:* VerifiEmail API output.  
     - *Output:*  
       - True branch: proceed to generate health card.  
       - False branch: stops workflow with error.  
     - *Edge Cases:* API response format changes, missing `valid` field.

  3. **Stop and Error**  
     - *Type:* Stop and Error node  
     - *Role:* Stops workflow execution and returns error "Invalid Email" to webhook caller.  
     - *Input:* False branch from email validity check.  
     - *Output:* Workflow halted with error.  
     - *Edge Cases:* None specifically; serves as controlled failure point.

#### 1.3 Health Card Generation

- **Overview:**  
  Generates a unique verification URL and QR code, builds a professional HTML health card embedding patient data and the QR code, converts the HTML card to a PNG image using a third-party API.

- **Nodes Involved:**  
  - Generate QR Code  
  - Build Health Card HTML  
  - HTML/CSS to Image  
  - Download PNG Binary  

- **Node Details:**

  1. **Generate QR Code**  
     - *Type:* Code node  
     - *Role:* Constructs a verification URL with patient ID, generates QR code image URL using free QR Server API.  
     - *Config:* Hardcoded base URL `https://clinic-verify.mediajade.com/patient/` plus patient ID from validated data.  
     - *Output:* Patient data enriched with `verification_url` and `qr_code_url`.  
     - *Edge Cases:* QR API availability, URL correctness.

  2. **Build Health Card HTML**  
     - *Type:* Code node  
     - *Role:* Builds a responsive, styled HTML document as a health card.  
     - *Config:*  
       - Uses inline CSS with gradients, grid layouts, hover effects.  
       - Includes patient details, symptoms, appointment info, QR code image, and instructions.  
       - Formats timestamp display.  
     - *Input:* Patient data with QR code URL.  
     - *Output:* Patient data plus `html_content` string for rendering.  
     - *Edge Cases:* Large symptom text might overflow, HTML rendering issues.

  3. **HTML/CSS to Image**  
     - *Type:* HTML/CSS to Image API node  
     - *Role:* Converts the generated HTML health card into a PNG image.  
     - *Config:* Passes `html_content` to HTMLCSSToImg API.  
     - *Credentials:* Requires HTMLCSSToImg API key.  
     - *Input:* HTML content string.  
     - *Output:* Image URL or binary file reference (depending on API).  
     - *Edge Cases:* API failures, timeout, malformed HTML.

  4. **Download PNG Binary**  
     - *Type:* HTTP Request node  
     - *Role:* Downloads the PNG image binary from the URL provided by the previous node.  
     - *Config:* Uses URL from previous node’s output `image_url`.  
     - *Input:* URL of generated PNG image.  
     - *Output:* Binary PNG file ready for upload.  
     - *Edge Cases:* Network issues, HTTP errors, file size limits.

#### 1.4 Google Drive Storage

- **Overview:**  
  Uploads the generated PNG health card image to a Google Drive folder named "Patients record", naming the file with the patient ID for easy reference.

- **Nodes Involved:**  
  - Upload PNG to Drive  

- **Node Details:**

  1. **Upload PNG to Drive**  
     - *Type:* Google Drive node  
     - *Role:* Uploads the PNG binary file to Google Drive folder specified by folder ID.  
     - *Config:*  
       - File named as `PAT-<timestamp-random>_health_card.png` dynamically from patient ID.  
       - Target folder is "Patients record" (folder ID must be set).  
     - *Credentials:* Google Drive OAuth2 credentials.  
     - *Input:* PNG binary data from prior download node.  
     - *Output:* Google Drive file metadata including shareable link.  
     - *Edge Cases:* Permission errors, quota limits, invalid folder ID.

#### 1.5 Delivery & Notifications

- **Overview:**  
  In parallel, sends the health card email to the patient, notifies the clinic reception team via Slack with patient details and file links, logs the patient record in Google Sheets, and finally responds to the webhook with success confirmation.

- **Nodes Involved:**  
  - Email Health Card to Patient  
  - Notify Reception Team  
  - Log to Patient Database  
  - Respond to Webhook  

- **Node Details:**

  1. **Email Health Card to Patient**  
     - *Type:* Gmail node  
     - *Role:* Sends an email to the patient with appointment details, embedded message HTML, and links to the health card PNG in Google Drive.  
     - *Config:*  
       - Recipient email from patient data.  
       - Subject includes appointment date/time.  
       - Message body is styled HTML with patient name, instructions, and Drive link button.  
     - *Credentials:* Gmail OAuth2 credentials.  
     - *Input:* Patient data and Drive link.  
     - *Output:* Email send confirmation.  
     - *Edge Cases:* Email delivery failures, invalid email addresses.

  2. **Notify Reception Team**  
     - *Type:* Slack node  
     - *Role:* Posts a formatted message to a configured Slack channel with new patient pre-check-in details and Drive file link.  
     - *Config:*  
       - Channel ID and Slack API credentials set.  
       - Message uses Markdown formatting with patient info and links.  
     - *Credentials:* Slack API OAuth2.  
     - *Input:* Patient data and Google Drive file link.  
     - *Output:* Slack message confirmation.  
     - *Edge Cases:* Slack API errors, permission issues.

  3. **Log to Patient Database**  
     - *Type:* Google Sheets node  
     - *Role:* Appends or updates patient record in a Google Sheet with all relevant fields including verification status, timestamps, and file links.  
     - *Config:*  
       - Sheet has columns: Timestamp, Patient ID, Name, Email, Phone, Email Verified, Appointment, Symptoms, PNG Link, Email Sent, Status.  
       - Matches rows on Patient ID to update existing records if any.  
     - *Credentials:* Google Sheets OAuth2.  
     - *Input:* Patient data and Drive link.  
     - *Output:* Sheet update confirmation.  
     - *Edge Cases:* Sheet access errors, mismatched columns.

  4. **Respond to Webhook**  
     - *Type:* Respond to Webhook node  
     - *Role:* Sends a JSON response to the original webhook caller indicating success, patient ID, and email verification status.  
     - *Config:* Static JSON response with dynamic fields from patient data.  
     - *Input:* Any one of the final nodes in delivery branch (configured as converging).  
     - *Output:* HTTP response to client.  
     - *Edge Cases:* Response failures if webhook connection dropped.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                          | Input Node(s)          | Output Node(s)                         | Sticky Note                                                                                                      |
|-------------------------|-------------------------|----------------------------------------|-----------------------|--------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook Trigger         | Webhook                 | Receives patient form data via POST    | None                  | Validate Input Data                  | ## 📥 Section 1: Data Input & Validation                                                                        |
| Validate Input Data     | Code                    | Validates and cleans input data        | Webhook Trigger       | Verifi Email                        | ## 📥 Section 1: Data Input & Validation                                                                        |
| Verifi Email            | VerifiEmail API         | Verifies patient email address         | Validate Input Data   | Check Email Valid                   | ## 🔍 Section 2: Email Verification                                                                              |
| Check Email Valid       | If                      | Routes flow based on email validity    | Verifi Email          | Generate QR Code, Stop and Error    | ## 🔍 Section 2: Email Verification                                                                              |
| Stop and Error          | Stop and Error           | Stops workflow on invalid email        | Check Email Valid (false) | None                              | ## 🔍 Section 2: Email Verification                                                                              |
| Generate QR Code        | Code                    | Generates verification URL and QR code | Check Email Valid (true) | Build Health Card HTML             | ## 🎨 Section 3: Health Card Generation                                                                          |
| Build Health Card HTML  | Code                    | Builds styled patient health card HTML | Generate QR Code      | HTML/CSS to Image                   | ## 🎨 Section 3: Health Card Generation                                                                          |
| HTML/CSS to Image       | HTML/CSS to Image API   | Converts HTML health card to PNG image  | Build Health Card HTML | Download PNG Binary                 | ## 🎨 Section 3: Health Card Generation                                                                          |
| Download PNG Binary     | HTTP Request            | Downloads PNG image binary              | HTML/CSS to Image     | Upload PNG to Drive                 | ## 📁 Section 4: Google Drive Storage                                                                           |
| Upload PNG to Drive     | Google Drive            | Uploads PNG to Google Drive folder      | Download PNG Binary   | Log to Patient Database, Notify Reception Team, Email Health Card to Patient | ## 📁 Section 4: Google Drive Storage                                                                           |
| Log to Patient Database | Google Sheets           | Logs patient record in Google Sheets    | Upload PNG to Drive   | Respond to Webhook                 | ## 📧 Section 5: Delivery & Notifications                                                                        |
| Notify Reception Team   | Slack                   | Posts patient info notification to Slack | Upload PNG to Drive | Respond to Webhook                 | ## 📧 Section 5: Delivery & Notifications                                                                        |
| Email Health Card to Patient | Gmail               | Sends health card email to patient      | Upload PNG to Drive   | Respond to Webhook                 | ## 📧 Section 5: Delivery & Notifications                                                                        |
| Respond to Webhook      | Respond to Webhook      | Sends success JSON response to caller   | Notify Reception Team, Log to Patient Database, Email Health Card to Patient | None                   | ## 📧 Section 5: Delivery & Notifications                                                                        |
| Sticky Note - Main Overview | Sticky Note          | Workflow high-level description         | None                  | None                              | ## 🏥 Patient Pre-Check-in Health Card Workflow... (see full note)                                              |
| Sticky Note - Input     | Sticky Note             | Describes Input & Validation block      | None                  | None                              | ## 📥 Section 1: Data Input & Validation                                                                        |
| Sticky Note - Verification | Sticky Note           | Describes Email Verification block      | None                  | None                              | ## 🔍 Section 2: Email Verification                                                                              |
| Sticky Note - Generation | Sticky Note            | Describes Health Card Generation block  | None                  | None                              | ## 🎨 Section 3: Health Card Generation                                                                          |
| Sticky Note - Storage   | Sticky Note             | Describes Google Drive Storage block    | None                  | None                              | ## 📁 Section 4: Google Drive Storage                                                                           |
| Sticky Note - Delivery  | Sticky Note             | Describes Delivery & Notifications block | None                | None                              | ## 📧 Section 5: Delivery & Notifications                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `patient-checkin`  
   - Response Mode: Response Node  
   - No credentials needed.

2. **Add Code node named "Validate Input Data"**  
   - Connect input from Webhook Trigger.  
   - Paste JavaScript code to:  
     - Extract POST body fields.  
     - Validate required fields: `name`, `email`, `appointment_date`, `appointment_time`.  
     - Clean and format data, default optional fields.  
     - Generate unique `patient_id`.  
     - Format appointment datetime strings in Indian locale.  
   - Outputs cleaned patient data JSON.

3. **Add VerifiEmail node "Verifi Email"**  
   - Connect input from Validate Input Data.  
   - Set email parameter to `={{ $json.email }}`.  
   - Configure VerifiEmail API credentials.

4. **Add If node "Check Email Valid"**  
   - Connect input from Verifi Email.  
   - Condition: `$json.valid === true` (strict boolean check).  
   - True branch continues workflow; False branch stops.

5. **Add Stop and Error node "Stop and Error"**  
   - Connect from False output of Check Email Valid.  
   - Set error message: "Invalid Email".

6. **Add Code node "Generate QR Code"**  
   - Connect from True output of Check Email Valid.  
   - JavaScript code to:  
     - Construct verification URL `https://clinic-verify.mediajade.com/patient/${patient_id}`.  
     - Generate QR code URL using `https://api.qrserver.com/v1/create-qr-code/` with size 300x300.  
     - Return enriched patient data with these URLs.

7. **Add Code node "Build Health Card HTML"**  
   - Connect from Generate QR Code.  
   - Paste provided HTML and CSS code that creates a visually styled health card embedding patient info and QR code image.  
   - Outputs patient data including `html_content` string.

8. **Add HTML/CSS to Image node**  
   - Connect from Build Health Card HTML.  
   - Set HTML content parameter as `={{ $json.html_content }}`.  
   - Configure HTMLCSSToImg API credentials.

9. **Add HTTP Request node "Download PNG Binary"**  
   - Connect from HTML/CSS to Image.  
   - Set URL parameter to `={{ $json.image_url }}` (or appropriate output from previous node).  
   - Set response format to file (binary).  

10. **Add Google Drive node "Upload PNG to Drive"**  
    - Connect from Download PNG Binary.  
    - Configure Google Drive OAuth2 credentials.  
    - Set file name as `={{ $json.patient_id }}_health_card.png`.  
    - Set target folder to your Google Drive folder ID (e.g., "Patients record").  

11. **Add Gmail node "Email Health Card to Patient"**  
    - Connect from Upload PNG to Drive.  
    - Configure Gmail OAuth2 credentials.  
    - Set recipient as `={{ $('Build Health Card HTML').item.json.email }}`.  
    - Compose email subject with appointment date/time.  
    - Compose HTML message body embedding patient name, appointment details, Drive link, and instructions.

12. **Add Slack node "Notify Reception Team"**  
    - Connect from Upload PNG to Drive.  
    - Configure Slack API credentials.  
    - Set target channel ID (e.g., #clinic-reception).  
    - Compose message with patient name, ID, email, phone, appointment, symptoms, and Drive link.

13. **Add Google Sheets node "Log to Patient Database"**  
    - Connect from Upload PNG to Drive.  
    - Configure Google Sheets OAuth2 credentials.  
    - Set spreadsheet ID and sheet name.  
    - Map columns: Timestamp, Patient ID, Name, Email, Phone, Email Verified, Appointment, Symptoms, PNG Link, Email Sent, Status.  
    - Use "Append or Update" operation matching on Patient ID.

14. **Add Respond to Webhook node**  
    - Connect from Email Health Card to Patient, Notify Reception Team, and Log to Patient Database in parallel.  
    - Configure to respond with JSON:  
      ```json
      {
        "success": true,
        "message": "Pre-check-in completed successfully",
        "patient_id": "{{ $json.patient_id }}",
        "email_sent": true,
        "verification_status": "verified"
      }
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                     | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow completes the entire patient pre-check-in process in about 15-20 seconds with full automation from data input to email delivery, enabling efficient clinic operations.                                                                                              | Main workflow overview sticky note                                                                        |
| Google Drive folder "Patients record" must be created beforehand; folder ID is required in Google Drive node configuration.                                                                                                                                                     | Setup step 2 in sticky note                                                                               |
| Google Sheet must have columns: Timestamp, Patient ID, Name, Email, Phone, Email Verified, Appointment DateTime, Symptoms, Drive Link, Email Sent, Status for proper logging.                                                                                                   | Setup step 3 in sticky note                                                                               |
| Slack channel (e.g., #clinic-reception) must be created and Slack API credentials configured for notification delivery.                                                                                                                                                         | Setup step 4 in sticky note                                                                               |
| VerifiEmail API, HTMLCSSToImg API, Google Drive, Gmail, Slack, and Google Sheets OAuth2 credentials must be securely added in n8n before running workflow.                                                                                                                       | Setup step 1 in sticky note                                                                               |
| The verification URL in QR code generation is currently set as `https://clinic-verify.mediajade.com/patient/{patient_id}` — replace this domain with your clinic's actual verification URL if needed.                                                                             | Generate QR Code node comment                                                                             |
| The HTML health card uses inline CSS and responsive design to ensure good display on both desktop and mobile devices, and is styled with gradients and grid layouts for clarity and professionalism.                                                                               | Build Health Card HTML node comment                                                                       |
| The free QR code API used (`api.qrserver.com`) has usage limits and reliability considerations; consider hosting your own QR code generator or using a paid service for high volume.                                                                                             | Generate QR Code node comment                                                                             |
| The email sent to patients includes both PNG and PDF attachments (if PDF generation added externally), appointment details, and a direct Drive link for convenience.                                                                                                          | Email Health Card to Patient node comment                                                                 |
| The workflow uses the Indian English locale with 12-hour time format for all date/time displays to match the target audience style.                                                                                                                                             | Validate Input Data node comment                                                                           |

---

**Disclaimer:** The text provided is derived exclusively from a fully automated workflow created using n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.

---