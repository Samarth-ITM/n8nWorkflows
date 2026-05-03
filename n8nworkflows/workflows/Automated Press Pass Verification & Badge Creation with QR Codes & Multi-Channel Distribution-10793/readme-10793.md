Automated Press Pass Verification & Badge Creation with QR Codes & Multi-Channel Distribution

https://n8nworkflows.xyz/workflows/automated-press-pass-verification---badge-creation-with-qr-codes---multi-channel-distribution-10793


# Automated Press Pass Verification & Badge Creation with QR Codes & Multi-Channel Distribution

### 1. Workflow Overview

This workflow automates the verification and issuance of press passes for media event journalists. It processes incoming applications submitted via a webhook, validates essential data, verifies email domains against approved media outlets, generates unique press IDs and QR codes, creates branded digital press badges, logs data to Google Sheets, sends confirmation emails with badges, notifies event organizers via Slack, and finally responds to the webhook caller with the outcome.

The workflow is logically structured into five main blocks:

- **1.1 Input Reception & Validation**: Receives journalist application data via webhook and validates mandatory fields.
- **1.2 Email Domain Verification**: Uses the VerifiEmail API to confirm that the journalist’s email domain is verified and trusted.
- **1.3 Press Pass & Badge Generation**: Generates a unique press ID, QR code, and creates a professional digital badge image.
- **1.4 Distribution & Logging**: Sends the badge via email, notifies organizers on Slack, logs the data to Google Sheets.
- **1.5 Final Response**: Sends a JSON success response back to the webhook caller with key press pass information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block handles incoming journalist press pass applications via a webhook and ensures all required fields (`name`, `email`, `photo_url`) are present before proceeding.

- **Nodes Involved:**  
  - Webhook Trigger  
  - Validate Required Fields (IF)  
  - Stop: Incomplete Data (StopAndError)

- **Node Details:**

  - **Webhook Trigger**  
    - Type: Webhook  
    - Role: Entry point that listens for HTTP POST requests at path `/press-application`.  
    - Configuration: Responds via Response Node mode to allow later response customization.  
    - Inputs: External HTTP POST data with application JSON payload.  
    - Outputs: Passes the JSON body downstream.  
    - Edge Cases: Missing request body, invalid JSON, HTTP errors.  

  - **Validate Required Fields**  
    - Type: IF Condition  
    - Role: Checks that `name`, `email`, and `photo_url` fields in the webhook JSON body are non-empty.  
    - Key Expressions:  
      - `={{ $json.body.name }}`  
      - `={{ $json.body.email }}`  
      - `={{ $json.body.photo_url }}`  
    - Inputs: Webhook Trigger output.  
    - Outputs:  
      - `true` branch: Proceed to email verification.  
      - `false` branch: Stop workflow with error.  
    - Edge Cases: Empty strings, missing keys, malformed JSON.  

  - **Stop: Incomplete Data**  
    - Type: Stop and Error  
    - Role: Halts workflow with error message "Data is Incomplete" if validation fails.  
    - Inputs: From `Validate Required Fields` false condition.  
    - Outputs: None (workflow terminates).  
    - Edge Cases: Execution stops here if required data missing.

---

#### 2.2 Email Domain Verification

- **Overview:**  
  This block verifies the journalist’s email domain using VerifiEmail API, confirming MX records and validation against approved media organizations.

- **Nodes Involved:**  
  - Verifi Email (VerifiEmail API node)  
  - Check Domain Verified (IF)  
  - Stop: Domain Not Verified (StopAndError)

- **Node Details:**

  - **Verifi Email**  
    - Type: VerifiEmail API  
    - Role: Validates the email address domain, MX records, and confirms domain legitimacy.  
    - Configuration: Uses credential for VerifiEmail account.  
    - Key Expression: `={{ $json.body.email }}` to pass the email from webhook.  
    - Inputs: From `Validate Required Fields` passing `true`.  
    - Outputs: JSON with domain verification status.  
    - Edge Cases: API authentication errors, network timeouts, invalid email formats.  

  - **Check Domain Verified**  
    - Type: IF Condition  
    - Role: Checks if the domain verification boolean flag `valid` is true.  
    - Key Expression: `={{ $json.valid }}`  
    - Inputs: Output from Verifi Email node.  
    - Outputs:  
      - `true` branch: Proceed to press ID generation.  
      - `false` branch: Stop workflow with domain error.  
    - Edge Cases: False negatives, missing `valid` field.

  - **Stop: Domain Not Verified**  
    - Type: Stop and Error  
    - Role: Terminates workflow if email domain is not verified, with error "Email Domain not Verified".  
    - Inputs: From `Check Domain Verified` false branch.  
    - Outputs: None (workflow halts).  
    - Edge Cases: None; stops on failure.

---

#### 2.3 Press Pass & Badge Generation

- **Overview:**  
  Generates a unique press pass ID, constructs a verification URL and dynamic QR code, and creates a branded digital badge image with journalist photo and credentials.

- **Nodes Involved:**  
  - Generate Press ID & QR (Code node)  
  - HTML/CSS to Image (HTML to Image converter node)

- **Node Details:**

  - **Generate Press ID & QR**  
    - Type: Code (JavaScript)  
    - Role: Creates a unique press ID using timestamp and random string, constructs verification URL, and generates a QR code URL using an external QR code API. Adds issue and expiry dates.  
    - Key Logic:  
      - Generates `press_id` like "PRESS-XXXXXX-TIMESTAMP".  
      - Constructs verification URL: `https://your-event-portal.com/verify?id=press_id`.  
      - Creates QR code image URL with `api.qrserver.com`.  
    - Inputs: JSON including webhook body and verification status.  
    - Outputs: Enriched JSON with press ID, QR code URL, issued and valid dates.  
    - Edge Cases: API rate limits for QR code generation, time zone inconsistencies, malformed input data.

  - **HTML/CSS to Image**  
    - Type: HTML/CSS to Image conversion  
    - Role: Generates a visually styled digital badge PNG image using embedded HTML and CSS, showing journalist photo, name, media outlet, press ID, QR code, and event branding.  
    - Configuration:  
      - CSS imports Google Fonts (Poppins).  
      - Badge layout styled with gradients, shadows, overlays, and responsive design.  
      - Dynamic placeholders populated via expressions from webhook and generated data.  
    - Inputs: Output from Generate Press ID & QR node.  
    - Outputs: JSON with `image_url` for the generated badge image.  
    - Edge Cases: API quota limits, HTML rendering errors, image generation timeouts.

---

#### 2.4 Distribution & Logging

- **Overview:**  
  Sends the generated press pass email with badge attachment to the journalist, notifies event organizers on Slack, and logs all relevant data into a Google Sheets spreadsheet for record-keeping.

- **Nodes Involved:**  
  - Send Press Pass Email (Gmail node)  
  - Notify Organizers (Slack node)  
  - Log to Google Sheets (Google Sheets node)

- **Node Details:**

  - **Send Press Pass Email**  
    - Type: Gmail (OAuth2)  
    - Role: Sends a rich HTML email to the journalist with personalized press pass details, instructions, and a download button for the digital badge.  
    - Key Configuration:  
      - Recipient: `={{ $('Generate Press ID & QR').item.json.body.email }}`  
      - Subject: `"✅ Your Verified Press Pass - {{ event_name }}"`  
      - Message: Styled HTML embedding dynamic data with inline CSS and event branding.  
    - Inputs: Badge image URL and press pass data.  
    - Outputs: Confirmation of email sent.  
    - Credentials: Gmail OAuth2 account required.  
    - Edge Cases: Email sending failures, OAuth token expiry, spam filtering.  

  - **Notify Organizers (Slack)**  
    - Type: Slack API  
    - Role: Sends a formatted Slack message to a specified channel with new press pass issuance details including journalist info, press ID, badge link, and verification URL.  
    - Configuration:  
      - Channel: Configured Slack channel ID.  
      - Message: Template with markdown formatting and dynamic data placeholders.  
    - Inputs: From Send Press Pass Email node.  
    - Credentials: Slack API token.  
    - Edge Cases: Slack API rate limits, authentication errors, invalid channel ID.  

  - **Log to Google Sheets**  
    - Type: Google Sheets (OAuth2)  
    - Role: Appends a new row to a configured Google Sheet with all relevant press pass application and issuance data for tracking.  
    - Configuration:  
      - Maps columns such as Name, Email, Phone, Press ID, Event Name, Issue and Validity dates, QR and badge URLs, verification status, and webhook execution mode.  
      - Target Sheet and Document IDs must be configured.  
    - Inputs: Data from previous nodes.  
    - Credentials: Google Sheets OAuth2 account.  
    - Edge Cases: API quota limits, invalid sheet ID, data mapping errors.

---

#### 2.5 Final Response

- **Overview:**  
  Responds to the initial webhook call with a JSON object confirming successful press pass creation and including key press pass details for immediate client feedback.

- **Nodes Involved:**  
  - Respond to Webhook (RespondToWebhook node)

- **Node Details:**

  - **Respond to Webhook**  
    - Type: RespondToWebhook  
    - Role: Sends a JSON response to the HTTP webhook caller with success status and press pass data (press ID, verification URL, badge URL, issue and valid dates).  
    - Configuration: Uses expressions to pull data from workflow JSON context.  
    - Inputs: From Logging or Slack notification nodes to guarantee all steps completed.  
    - Outputs: HTTP 200 response with JSON payload.  
    - Edge Cases: Response errors if node receives no input, network interruptions.

---

### 3. Summary Table

| Node Name                | Node Type              | Functional Role                                  | Input Node(s)              | Output Node(s)                         | Sticky Note                                                                                                                                                                                                                          |
|--------------------------|------------------------|-------------------------------------------------|----------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook Trigger          | Webhook                | Receives journalist application POST requests   | -                          | Validate Required Fields              | ## Data Reception & Validation. Webhook receives journalist applications and validates required fields (name, email, photo_url) before processing.                                                                                 |
| Validate Required Fields  | IF Condition           | Validates mandatory fields in incoming data      | Webhook Trigger            | Verifi Email (true), Stop: Incomplete Data (false) | ## Data Reception & Validation. Webhook receives journalist applications and validates required fields (name, email, photo_url) before processing.                                                                                 |
| Stop: Incomplete Data     | StopAndError           | Stops workflow if required fields are missing    | Validate Required Fields   | -                                    | ## Data Reception & Validation. Webhook receives journalist applications and validates required fields (name, email, photo_url) before processing.                                                                                 |
| Verifi Email             | VerifiEmail API        | Validates email domain against approved outlets  | Validate Required Fields   | Check Domain Verified                 | ## Email Verification. VerifiEmail API validates email domain, checks MX records, and confirms journalist works for approved media outlet.                                                                                        |
| Check Domain Verified     | IF Condition           | Checks if email domain is verified                | Verifi Email               | Generate Press ID & QR (true), Stop: Domain Not Verified (false) | ## Email Verification. VerifiEmail API validates email domain, checks MX records, and confirms journalist works for approved media outlet.                                                                                        |
| Stop: Domain Not Verified | StopAndError           | Stops workflow if email domain not verified       | Check Domain Verified      | -                                    | ## Email Verification. VerifiEmail API validates email domain, checks MX records, and confirms journalist works for approved media outlet.                                                                                        |
| Generate Press ID & QR    | Code                   | Generates unique press ID, verification URL, QR | Check Domain Verified      | HTML/CSS to Image                    | ## Badge Creation. Generates unique press ID, QR code, and professional badge image with journalist photo, credentials, and event branding.                                                                                        |
| HTML/CSS to Image         | HTML/CSS to Image      | Creates branded digital press badge image         | Generate Press ID & QR     | Send Press Pass Email                | ## Badge Creation. Generates unique press ID, QR code, and professional badge image with journalist photo, credentials, and event branding.                                                                                        |
| Send Press Pass Email     | Gmail (OAuth2)         | Sends confirmation email with badge to journalist | HTML/CSS to Image          | Notify Organizers (Slack), Log to Google Sheets | ## Distribution & Logging. Sends badge via email, notifies organizers on Slack, logs to Google Sheets, then responds to webhook with success.                                                                                      |
| Notify Organizers (Slack) | Slack API              | Notifies event organizers of new press pass       | Send Press Pass Email      | Respond to Webhook                  | ## Distribution & Logging. Sends badge via email, notifies organizers on Slack, logs to Google Sheets, then responds to webhook with success.                                                                                      |
| Log to Google Sheets      | Google Sheets (OAuth2) | Logs press pass data for record-keeping            | Send Press Pass Email      | Respond to Webhook                  | ## Distribution & Logging. Sends badge via email, notifies organizers on Slack, logs to Google Sheets, then responds to webhook with success.                                                                                      |
| Respond to Webhook        | RespondToWebhook       | Sends final success response to webhook caller    | Notify Organizers (Slack), Log to Google Sheets | -                          | ## Distribution & Logging. Sends badge via email, notifies organizers on Slack, logs to Google Sheets, then responds to webhook with success.                                                                                      |
| Sticky Note               | Sticky Note            | Workflow overview and setup instructions          | -                          | -                                    | ## How it works\nAutomated press pass generation system for media events. When journalists submit applications via webhook, the workflow validates their data, verifies email domains using VerifiEmail API against approved media organizations, generates unique press IDs with QR codes, creates professional branded badges using HTMLCSSToImage, stores records in Google Sheets, sends email confirmations with badge attachments, and notifies event organizers via Slack—all in under 10 seconds.\n\n## Setup steps\n1. **Activate workflow** and copy webhook URL to embed in your event website form\n2. **Connect credentials**: VerifiEmail API, HTMLCSSToImage, Gmail OAuth, Slack OAuth, Google Sheets API\n3. **Setup Google Sheets**: Add column headers:- Timestamp,Press ID,Name,Email,Phone,Media Outlet,Verification Status,Event Name,Issued Date,Valid Until,Badge Image URL,QR Code URL,Verification URL,Photo URL,Webhook Execution Mode\n4. **Customize branding**: Edit HTML/CSS in badge generation node to match your event colors and logo\n5. **Test thoroughly**: Send sample POST requests with all required fields before going live |
| Sticky Note1              | Sticky Note            | Data reception & validation summary                | -                          | -                                    | ## Data Reception & Validation\nWebhook receives journalist applications and validates required fields (name, email, photo_url) before processing.                                                                                 |
| Sticky Note2              | Sticky Note            | Email verification summary                          | -                          | -                                    | ## Email Verification\nVerifiEmail API validates email domain, checks MX records, and confirms journalist works for approved media outlet.                                                                                        |
| Sticky Note3              | Sticky Note            | Badge creation summary                              | -                          | -                                    | ## Badge Creation\nGenerates unique press ID, QR code, and professional badge image with journalist photo, credentials, and event branding.                                                                                        |
| Sticky Note4              | Sticky Note            | Distribution & logging summary                      | -                          | -                                    | ## Distribution & Logging\nSends badge via email, notifies organizers on Slack, logs to Google Sheets, then responds to webhook with success.                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `press-application`  
   - Response Mode: Response Node  
   - Purpose: Entry point for receiving journalist press pass application data.

2. **Add IF Node for Field Validation ("Validate Required Fields")**  
   - Checks non-empty: `body.name`, `body.email`, `body.photo_url`  
   - Conditions: All must be not empty (string not empty).  
   - Connect Webhook Trigger main output to this IF node.

3. **Add Stop Error Node ("Stop: Incomplete Data")**  
   - Error message: "Data is Incomplete"  
   - Connect this node to the `false` output of Validate Required Fields.

4. **Add VerifiEmail Node ("Verifi Email")**  
   - Set `email` parameter to `={{ $json.body.email }}`  
   - Connect `true` output of Validate Required Fields to this node.  
   - Configure with VerifiEmail API credentials.

5. **Add IF Node ("Check Domain Verified")**  
   - Condition: `={{ $json.valid }}` is boolean true  
   - Connect main output of Verifi Email node.  
   - `true` branch connects to next block; `false` branch to Stop node.

6. **Add Stop Error Node ("Stop: Domain Not Verified")**  
   - Error message: "Email Domain not Verified"  
   - Connect from `false` output of Check Domain Verified.

7. **Add Code Node ("Generate Press ID & QR")**  
   - Paste JavaScript code that:  
     - Generates unique `press_id` (e.g., "PRESS-XXXXXX-TIMESTAMP")  
     - Builds verification URL using base event portal URL and press ID  
     - Generates QR code URL via `api.qrserver.com` with verification URL data  
     - Adds issued date (ISO string), issued date formatted, and valid until date (+30 days)  
   - Connect `true` output of Check Domain Verified to this node.

8. **Add HTML/CSS to Image Node ("HTML/CSS to Image")**  
   - CSS: Import Google Fonts Poppins, set viewport width=400, height=600.  
   - HTML: Build badge layout with styles including gradient background, photo, name, media outlet, press ID, QR code image, event name, and validity date. Use expressions for dynamic data such as photo URL and press ID.  
   - Connect output of Generate Press ID & QR node.

9. **Add Gmail Node ("Send Press Pass Email")**  
   - Recipient: `={{ $('Generate Press ID & QR').item.json.body.email }}`  
   - Subject: `=✅ Your Verified Press Pass - {{ $('Webhook Trigger').item.json.body.event_name }}`  
   - Message: Use the provided comprehensive HTML email template with inline CSS, dynamic placeholders for press pass details, instructions, and download button linked to badge image URL.  
   - Credential: Gmail OAuth2 configured.  
   - Connect output of HTML/CSS to Image node.

10. **Add Slack Node ("Notify Organizers (Slack)")**  
    - Channel: Set to your Slack channel ID.  
    - Message: Compose formatted notification including journalist name, media outlet, email, press ID, badge image link, verification URL, issued and valid dates.  
    - Credential: Slack API token.  
    - Connect output of Gmail node.

11. **Add Google Sheets Node ("Log to Google Sheets")**  
    - Operation: Append  
    - Map columns: Name, Email, Phone, Press ID, Photo URL, Timestamp (current ISO), Event Name, Issued Date, Valid Until, QR Code URL, Media Outlet, Badge Image URL, Verification URL, Verification Status ("YES"), Webhook Execution Mode.  
    - Configure with your Google Sheets document ID and sheet name.  
    - Credential: Google Sheets OAuth2.  
    - Connect output of Gmail node (parallel with Slack node).

12. **Add Respond to Webhook Node ("Respond to Webhook")**  
    - Respond with JSON object containing:  
      - `success: true`  
      - `message: "Press pass generated successfully"`  
      - `press_id`, `verification_url`, `badge_url`, `issued_at`, `valid_until` from previous data  
    - Connect outputs of both Slack and Google Sheets nodes to this node.

13. **Add Stop Nodes for error handling** as described in steps 3 and 6.

14. **Add Sticky Notes** for documentation at appropriate locations describing workflow purpose, data validation, email verification, badge creation, and distribution & logging.

15. **Activate the workflow** and test with sample POST requests containing all required fields.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Automated press pass generation system for media events. When journalists submit applications via webhook, the workflow validates their data, verifies email domains using VerifiEmail API against approved media organizations, generates unique press IDs with QR codes, creates professional branded badges using HTMLCSSToImage, stores records in Google Sheets, sends email confirmations with badge attachments, and notifies event organizers via Slack—all in under 10 seconds.                             | Sticky Note overview attached to workflow start. |
| Setup steps include activating the workflow, copying webhook URL for embedding in event websites, connecting all required credentials (VerifiEmail API, HTMLCSSToImage, Gmail OAuth2, Slack OAuth, Google Sheets API), preparing Google Sheets with specified headers, customizing badge branding HTML/CSS, and thorough testing before going live.                                                                                                                                            | Sticky Note setup instructions.               |
| The badge generation uses Google Fonts (Poppins) and a modern gradient design, ensuring a professional and consistent look across all issued press passes.                                                                                                                                                                                                                                                                                                                                                   | Badge Creation Sticky Note.                    |
| The email template is a fully styled HTML email with instructions for journalists on saving the email, presenting QR code at event entry, and badge non-transferability.                                                                                                                                                                                                                                                                                                                                   | Email node HTML content.                        |
| VerifiEmail API usage requires valid API credentials and may incur usage limits; ensure proper monitoring and error handling for API failures.                                                                                                                                                                                                                                                                                                                                                               | Email verification block note.                  |
| Slack notifications provide instant alerts to event media teams with direct links to badge and verification, improving event security and organization.                                                                                                                                                                                                                                                                                                                                                      | Slack node usage note.                          |
| Google Sheets logging keeps a permanent record of all issued passes with timestamps and verification status, useful for audits and reporting.                                                                                                                                                                                                                                                                                                                                                               | Google Sheets node usage note.                  |

---

**Disclaimer:** The source text and workflow content come exclusively from an automated n8n workflow, adhering strictly to content policies and containing no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.