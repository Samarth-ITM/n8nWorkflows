Workshop Certificate Generator with Google Drive, Gmail & QR Verification

https://n8nworkflows.xyz/workflows/workshop-certificate-generator-with-google-drive--gmail---qr-verification-10979


# Workshop Certificate Generator with Google Drive, Gmail & QR Verification

### 1. Workflow Overview

This workflow automates the issuance of pre-registered workshop certificates by integrating multiple services: it receives participant registration data via a webhook, validates required fields, verifies the participant's email, generates a unique certificate ID with a QR code, creates a styled HTML certificate, converts it to an image, stores it on Google Drive, emails the certificate to the participant, logs the registration in Google Sheets, and notifies organizers on Slack. Finally, it returns a JSON confirmation response to the webhook.

**Logical Blocks:**

- **1.1 Input Reception & Validation:** Accepts registration data via webhook, checks required fields, and verifies email validity.
- **1.2 Certificate Generation:** Generates unique certificate ID and QR code, prepares the HTML certificate.
- **1.3 Certificate Rendering & Storage:** Converts HTML certificate to image, downloads the image, and uploads it to Google Drive.
- **1.4 Email Delivery & Logging:** Sends the certificate via email, logs registration in Google Sheets, and notifies organizers via Slack.
- **1.5 Response Handling:** Returns success or error responses to the webhook based on validation and processing status.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

**Overview:**  
Receives registration form submissions, validates that essential fields (name, email, event) are present, and verifies the email address using the VerifiEmail service.

**Nodes Involved:**  
- Webhook Registration  
- Validate Required Fields  
- Validation Error Response  
- Verifi Email  
- Check Email Valid  
- Email Verification Error  

**Node Details:**

- **Webhook Registration**  
  - *Type:* Webhook  
  - *Role:* Entry point. Receives POST requests at path `/workshop-registration`.  
  - *Config:* HTTP method POST, response mode set to respond via a node.  
  - *Inputs:* External HTTP request with JSON body containing registration fields (name, email, event, date, time, venue, designation, organization).  
  - *Outputs:* Passes data to "Validate Required Fields".  
  - *Failures:* Missing required POST data, invalid request format.

- **Validate Required Fields**  
  - *Type:* If  
  - *Role:* Checks that `name`, `email`, and `event` fields in the webhook body are non-empty strings.  
  - *Config:* Three conditions combined with AND; all must be non-empty strings.  
  - *Inputs:* Output from Webhook Registration.  
  - *Outputs:*  
    - True: proceeds to "Verifi Email" node.  
    - False: routes to "Validation Error Response".  
  - *Failures:* Expression errors if expected JSON path is missing.

- **Validation Error Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Returns JSON indicating missing required fields with HTTP response.  
  - *Config:* JSON body with `success: false`, `error`, and `message`.  
  - *Inputs:* False branch of "Validate Required Fields".  
  - *Outputs:* Ends workflow for invalid input.  

- **Verifi Email**  
  - *Type:* VerifiEmail node  
  - *Role:* Sends the provided email for validation using VerifiEmail API.  
  - *Config:* Email parameter extracted from webhook body.  
  - *Credentials:* Requires VerifiEmail API key configured.  
  - *Inputs:* True branch of "Validate Required Fields".  
  - *Outputs:* Passes email validation result to "Check Email Valid".  
  - *Failures:* API authentication errors, network timeouts.

- **Check Email Valid**  
  - *Type:* If  
  - *Role:* Checks if the email verification returned `deliverable: true`.  
  - *Config:* Condition checks the boolean field `valid` equals "deliverable".  
  - *Inputs:* Output from "Verifi Email".  
  - *Outputs:*  
    - True: proceeds to certificate generation.  
    - False: routes to "Email Verification Error".  
  - *Failures:* Expression errors if verification response lacks expected properties.

- **Email Verification Error**  
  - *Type:* Respond to Webhook  
  - *Role:* Returns JSON indicating email verification failed.  
  - *Config:* JSON body with `success: false`, `error`, and `message`.  
  - *Inputs:* False branch of "Check Email Valid".  
  - *Outputs:* Ends workflow for invalid email.

---

#### 2.2 Certificate Generation

**Overview:**  
Generates a unique certificate ID and verification QR code URL, then prepares a detailed HTML certificate template populated with registration data and generated information.

**Nodes Involved:**  
- Generate Certificate ID & QR  
- Prepare Certificate HTML  

**Node Details:**

- **Generate Certificate ID & QR**  
  - *Type:* Code (JavaScript)  
  - *Role:* Creates a unique certificate ID based on timestamp and random string, constructs a verification URL, and generates a QR code URL. Also formats the current date for display.  
  - *Config:* Custom JavaScript code generating `certificateId`, `verifyUrl`, `qrCodeUrl`, `timestamp`, and `formattedDate`, merged into the item JSON.  
  - *Inputs:* Output from "Check Email Valid" (true branch).  
  - *Outputs:* Passes enriched JSON to "Prepare Certificate HTML".  
  - *Failures:* Script runtime errors, date/time formatting issues.

- **Prepare Certificate HTML**  
  - *Type:* Set  
  - *Role:* Builds an HTML certificate document string with embedded CSS styling, dynamically injecting registration fields (name, organization, event, date, venue, etc.) and generated QR code/certificate ID.  
  - *Config:* Large multiline HTML string using mustache expressions referencing data from "Webhook Registration" and "Generate Certificate ID & QR". Labelled as `certificateHtml`.  
  - *Inputs:* Output from "Generate Certificate ID & QR".  
  - *Outputs:* Passes HTML content to "HTML/CSS to Image" node.  
  - *Failures:* Expression evaluation errors, malformed HTML.

---

#### 2.3 Certificate Rendering & Storage

**Overview:**  
Converts the prepared HTML certificate to a PNG image, downloads the binary image, and uploads it to a specified Google Drive folder with metadata for easy retrieval.

**Nodes Involved:**  
- HTML/CSS to Image  
- Download Binary File  
- Upload to Google Drive  

**Node Details:**

- **HTML/CSS to Image**  
  - *Type:* HTML/CSS to Image  
  - *Role:* Converts the HTML content (`certificateHtml`) to an image URL using an external API.  
  - *Config:* Uses the `certificateHtml` string as input HTML content.  
  - *Credentials:* Requires Htmlcsstoimg API key.  
  - *Inputs:* Output from "Prepare Certificate HTML".  
  - *Outputs:* Provides image URL for next step.  
  - *Failures:* API quota exceeded, invalid HTML content, network errors.

- **Download Binary File**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the PNG image binary from the URL generated by the previous node.  
  - *Config:* URL dynamically set to the `image_url` field returned by "HTML/CSS to Image".  
  - *Inputs:* Output from "HTML/CSS to Image".  
  - *Outputs:* Binary data passed to Google Drive upload.  
  - *Failures:* Download failures, broken URLs.

- **Upload to Google Drive**  
  - *Type:* Google Drive  
  - *Role:* Uploads the downloaded PNG certificate file to a specified Google Drive folder, setting file metadata with attendee and event info for easy lookup.  
  - *Config:*  
    - File name constructed as `{name}_{certificateId}.png`.  
    - Drive and folder IDs configured (replace placeholders).  
    - App properties set for `certificateId`, `event`, and `attendee`.  
  - *Credentials:* Google Drive OAuth2 account required.  
  - *Inputs:* Binary file from "Download Binary File".  
  - *Outputs:* Passes Google Drive file metadata to email node.  
  - *Failures:* OAuth token expiration, permission issues, quota limits.

---

#### 2.4 Email Delivery & Logging

**Overview:**  
Sends an HTML email to the participant including event details, the certificate preview, QR code, and download link; logs the registration and certificate info in Google Sheets; and notifies the organizers on Slack.

**Nodes Involved:**  
- Send Certificate Email  
- Log to Google Sheets  
- Notify Slack Channel  

**Node Details:**

- **Send Certificate Email**  
  - *Type:* Gmail  
  - *Role:* Sends a rich HTML email confirming registration with embedded event info, QR code, and certificate image link.  
  - *Config:*  
    - Recipient email from verified email node.  
    - Subject includes event name.  
    - Uses HTML message template referencing workflow nodes for dynamic content (name, event, date, QR code, certificate image).  
  - *Credentials:* Gmail OAuth2 account required.  
  - *Inputs:* Output from "Upload to Google Drive".  
  - *Outputs:* Passes data to logging node.  
  - *Failures:* Email send limits, OAuth errors.

- **Log to Google Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Appends a new row to a Google Sheet with participant and certificate details for record-keeping.  
  - *Config:*  
    - Sheet and document IDs placeholders must be replaced.  
    - Columns include name, email, event, venue, status ("Pre-Issued"), certificate ID, certificate URL, Google Drive link, registration date, etc.  
  - *Credentials:* Google Sheets OAuth2 account required.  
  - *Inputs:* Output from "Send Certificate Email".  
  - *Outputs:* Passes data to Slack notification.  
  - *Failures:* Permission denied, API quota exceeded.

- **Notify Slack Channel**  
  - *Type:* Slack  
  - *Role:* Sends a formatted Slack message to a configured channel notifying organizers of the new registration and providing key details and certificate links.  
  - *Config:* Slack channel ID and API token must be configured.  
  - *Inputs:* Output from "Log to Google Sheets".  
  - *Outputs:* Passes data to success response node.  
  - *Failures:* Slack API rate limits, invalid channel ID.

---

#### 2.5 Response Handling

**Overview:**  
Returns appropriate JSON responses to the webhook caller depending on the workflow outcome, confirming success or reporting validation or email verification errors.

**Nodes Involved:**  
- Success Response  
- Validation Error Response (covered in 2.1)  
- Email Verification Error (covered in 2.1)  

**Node Details:**

- **Success Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Returns a JSON object with success status, confirmation message, generated certificate ID, and verification URL.  
  - *Config:* Dynamic JSON referencing generated data.  
  - *Inputs:* Output from "Notify Slack Channel".  
  - *Outputs:* Finalizes the webhook response.  
  - *Failures:* None expected unless response generation fails.

- **Validation Error Response** and **Email Verification Error**  
  - As described in 2.1, handle input validation or email verification failure cases, returning HTTP JSON error messages and halting workflow.

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                                    | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                     |
|----------------------------|-----------------------|---------------------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------|
| Webhook Registration       | Webhook               | Entry point, receives registration data           | -                          | Validate Required Fields     | ## How it Works This workflow automates issuing pre-registration workshop certificates...                      |
| Validate Required Fields    | If                    | Validates required fields (name, email, event)    | Webhook Registration       | Verifi Email / Validation Error Response | ## Validation Webhook receives form data...                                                                   |
| Validation Error Response   | Respond to Webhook    | Returns error for missing required fields         | Validate Required Fields (false) | -                           | ## Validation Webhook receives form data...                                                                   |
| Verifi Email               | VerifiEmail           | Verifies email deliverability                      | Validate Required Fields (true) | Check Email Valid            | ## Validation Webhook receives form data...                                                                   |
| Check Email Valid           | If                    | Checks if email verified as deliverable            | Verifi Email               | Generate Certificate ID & QR / Email Verification Error | ## Validation Webhook receives form data...                                                                   |
| Email Verification Error    | Respond to Webhook    | Returns error for invalid email                    | Check Email Valid (false)  | -                           | ## Validation Webhook receives form data...                                                                   |
| Generate Certificate ID & QR| Code                  | Generates unique certificate ID and QR code       | Check Email Valid (true)   | Prepare Certificate HTML     | ## Certificate Generation Generates unique Certificate ID + QR code, inserts attendee/event details           |
| Prepare Certificate HTML    | Set                   | Builds HTML certificate template                   | Generate Certificate ID & QR | HTML/CSS to Image          | ## Certificate Generation Generates unique Certificate ID + QR code, inserts attendee/event details           |
| HTML/CSS to Image           | HTML/CSS to Image     | Converts HTML certificate to PNG image             | Prepare Certificate HTML   | Download Binary File         | ## Storage Certificate PNG is downloaded and uploaded to Google Drive...                                       |
| Download Binary File        | HTTP Request          | Downloads PNG image binary                          | HTML/CSS to Image          | Upload to Google Drive       | ## Storage Certificate PNG is downloaded and uploaded to Google Drive...                                       |
| Upload to Google Drive      | Google Drive          | Uploads certificate PNG and metadata               | Download Binary File       | Send Certificate Email       | ## Storage Certificate PNG is downloaded and uploaded to Google Drive...                                       |
| Send Certificate Email      | Gmail                 | Sends confirmation email with certificate         | Upload to Google Drive     | Log to Google Sheets         | ## Email Delivery Sends participant an HTML email including event info, QR code, and the certificate image/download link. |
| Log to Google Sheets        | Google Sheets         | Logs registration and certificate details         | Send Certificate Email     | Notify Slack Channel         | ## Logging & Alerts Registration + certificate details are logged in Google Sheets...                          |
| Notify Slack Channel        | Slack                 | Sends Slack notification to organizers             | Log to Google Sheets       | Success Response             | ## Logging & Alerts Registration + certificate details are logged in Google Sheets...                          |
| Success Response            | Respond to Webhook    | Returns success JSON response to webhook caller   | Notify Slack Channel       | -                           | ## Response Returns JSON with success message, certificateId, and verification URL.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Registration Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `workshop-registration`  
   - Response Mode: Response Node  

2. **Add Validate Required Fields Node (If)**  
   - Conditions: All must be true:  
     - `$json.body.name` is not empty  
     - `$json.body.email` is not empty  
     - `$json.body.event` is not empty  
   - Connect `Webhook Registration` output to this node’s input.

3. **Add Validation Error Response Node**  
   - Type: Respond to Webhook  
   - Response Body: JSON with `success: false`, error message about missing required fields.  
   - Connect false output of Validate Required Fields to this node.

4. **Add Verifi Email Node**  
   - Type: VerifiEmail  
   - Parameter: Email = `{{$json.body.email}}`  
   - Configure VerifiEmail API credentials.  
   - Connect true output of Validate Required Fields to this node.

5. **Add Check Email Valid Node (If)**  
   - Condition: `$json.valid == 'deliverable'` (boolean true)  
   - Connect output of Verifi Email to this node.

6. **Add Email Verification Error Node**  
   - Type: Respond to Webhook  
   - Response Body: JSON indicating invalid email address and verification failure message.  
   - Connect false output of Check Email Valid to this node.

7. **Add Generate Certificate ID & QR Node (Code)**  
   - Paste provided JavaScript code to generate unique certificate ID, verification URL, QR code URL, and formatted date.  
   - Connect true output of Check Email Valid to this node.

8. **Add Prepare Certificate HTML Node (Set)**  
   - Add new string field `certificateHtml` with the provided full HTML template.  
   - Use mustache expressions to inject data from `"Webhook Registration"` and `"Generate Certificate ID & QR"` nodes.  
   - Connect output of Generate Certificate ID & QR to this node.

9. **Add HTML/CSS to Image Node**  
   - Input: Use `certificateHtml` field from previous node.  
   - Configure Htmlcsstoimg API credentials.  
   - Connect output of Prepare Certificate HTML to this node.

10. **Add Download Binary File Node (HTTP Request)**  
    - URL: Use the `image_url` property from HTML/CSS to Image node output.  
    - Connect output of HTML/CSS to Image to this node.

11. **Add Upload to Google Drive Node**  
    - File Name: `={{ $('Webhook Registration').item.json.body.name }}_{{ $('Generate Certificate ID & QR').item.json.certificateId }}.png`  
    - Drive and Folder IDs: Set to your Google Drive and folder.  
    - Set App Properties with keys `certificateId`, `event`, `attendee` and corresponding values.  
    - Configure Google Drive OAuth2 credentials.  
    - Connect output of Download Binary File to this node.

12. **Add Send Certificate Email Node (Gmail)**  
    - Recipient: `={{ $('Check Email Valid').item.json.email }}`  
    - Subject: `=✅ Your Workshop Certificate - {{ $('Webhook Registration').item.json.body.event }}`  
    - Message: Paste provided HTML email template, referencing necessary dynamic fields.  
    - Configure Gmail OAuth2 credentials.  
    - Connect output of Upload to Google Drive to this node.

13. **Add Log to Google Sheets Node**  
    - Operation: Append  
    - Document ID and Sheet Name: Set your Google Sheets document and sheet.  
    - Columns: Map fields (name, email, event, venue, status, drive link, certificate ID, URL, registration date, etc.) as per node config.  
    - Configure Google Sheets OAuth2 credentials.  
    - Connect output of Send Certificate Email to this node.

14. **Add Notify Slack Channel Node**  
    - Channel ID: Your Slack channel ID  
    - Text: Use provided template with dynamic content placeholders.  
    - Configure Slack API credentials.  
    - Connect output of Log to Google Sheets to this node.

15. **Add Success Response Node (Respond to Webhook)**  
    - Response Body: JSON with `success: true`, confirmation message, `certificateId`, and `verifyUrl`.  
    - Connect output of Notify Slack Channel to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow automates issuing pre-registration workshop certificates with validation, verification, generation, storage, emailing, logging, and alerts.  | Workflow description and use case.                                                                          |
| Setup requires adding API credentials for VerifiEmail, Htmlcsstoimg, Gmail OAuth2, Google Sheets OAuth2, Google Drive OAuth2, and Slack API.             | Credential configuration step.                                                                              |
| Replace placeholders for Google Drive folder, Google Sheets document/sheet, Slack channel, and API keys with your actual resources before publishing.  | Setup instructions.                                                                                          |
| The certificate HTML template uses Google Fonts and inline CSS for a professional design; customize branding as needed.                                | Customization tip.                                                                                           |
| Slack notification includes direct links to certificate and Google Drive for quick organizer access.                                                  | Organizer alert enhancement.                                                                                 |
| Email contains detailed event information with embedded QR code and download button to improve user experience.                                        | User communication best practice.                                                                            |
| Returned JSON from webhook facilitates integration with external systems or front-end confirmation messages.                                          | Integration consideration.                                                                                    |

---

**Disclaimer:**  
The text above is derived solely from an n8n automation workflow. It complies with all content policies and contains no illegal or protected material. All processed data is legal and publicly accessible.