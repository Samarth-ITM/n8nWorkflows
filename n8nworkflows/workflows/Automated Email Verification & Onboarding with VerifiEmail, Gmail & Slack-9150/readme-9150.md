Automated Email Verification & Onboarding with VerifiEmail, Gmail & Slack

https://n8nworkflows.xyz/workflows/automated-email-verification---onboarding-with-verifiemail--gmail---slack-9150


# Automated Email Verification & Onboarding with VerifiEmail, Gmail & Slack

---
### 1. Workflow Overview

This workflow automates the verification and onboarding process for new customer signups by validating their email addresses, sending personalized welcome or correction emails accordingly, logging verified users, and notifying the team via Slack. It targets SaaS platforms or online services that require reliable user email validation before granting access.

The workflow is logically divided into these key blocks:

- **1.1 Input Reception:** Receives new signup data via a webhook.
- **1.2 Data Sanitization:** Normalizes and validates incoming data structure.
- **1.3 Email Verification:** Uses VerifiEmail API to check email validity.
- **1.4 Validation Decision:** Branches flow based on email validity.
- **1.5 Welcome Email Preparation and Sending:** Constructs and sends a personalized welcome email for valid emails.
- **1.6 Invalid Email Handling:** Prepares a correction email and stops the workflow for invalid emails.
- **1.7 Logging and Notification:** Logs verified users to Google Sheets and sends Slack alerts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for new user signup data via HTTP POST requests to a webhook endpoint.
- **Nodes Involved:** Webhook
- **Node Details:**

  **Webhook**
  - Type: Webhook - entry point for external HTTP POST requests.
  - Configuration:
    - HTTP Method: POST
    - Path: `/new-signup`
    - Authentication: None configured (consider adding if public-facing)
    - Expects JSON body with `name` and `email` fields.
  - Inputs: External HTTP calls
  - Outputs: Passes received data to Data Sanitization node.
  - Edge cases:
    - Malformed requests or missing required fields cause failure downstream.
    - No rate limiting configured; susceptible to abuse if public.
  - Sticky Note: Explains configuration, required fields, and debugging tips.

#### 2.2 Data Sanitization

- **Overview:** Normalizes the input data by trimming whitespace, lowercasing emails, and verifying required fields.
- **Nodes Involved:** Data Sanitization
- **Node Details:**

  **Data Sanitization**
  - Type: Code node executing JavaScript.
  - Configuration:
    - Extracts `name` and `email` from various possible input formats.
    - Trims whitespace from `name`, lowercases and trims `email`.
    - Throws error if either field is missing.
    - Adds a timestamp of receipt.
    - Preserves original email as received.
  - Inputs: Webhook output.
  - Outputs: Cleaned JSON with `{name, email, original_email, received_at}`.
  - Edge cases:
    - Missing fields or unexpected input structure leads to explicit error.
  - Sticky Note: Details actions, common issues, and output example.

#### 2.3 Email Verification

- **Overview:** Validates the sanitized email address using the VerifiEmail API to check deliverability and legitimacy.
- **Nodes Involved:** Email Validation
- **Node Details:**

  **Email Validation**
  - Type: VerifiEmail node (third-party API integration).
  - Configuration:
    - Email input dynamically set from sanitized email.
    - Uses stored VerifiEmail API credentials.
  - Inputs: Data Sanitization output.
  - Outputs: JSON with validation results including boolean `valid` field.
  - Edge cases:
    - API failures cause workflow error (no fallback handling implemented).
    - Potential rate limit or quota issues with API.
  - Sticky Note: Explains validation features and credential link.

#### 2.4 Validation Decision

- **Overview:** Routes workflow based on whether email is valid or invalid.
- **Nodes Involved:** Validation Decision (IF node)
- **Node Details:**

  **Validation Decision**
  - Type: If node (conditional branching).
  - Configuration:
    - Condition: `$json.valid == true`
    - True branch: Proceed to personalize welcome email.
    - False branch: Prepare correction email.
  - Inputs: Email Validation output.
  - Outputs: Two branches.
  - Edge cases:
    - If `valid` field missing or unexpected, may default to false branch.
  - Sticky Note: Describes decision logic and monitoring recommendations.

#### 2.5 Welcome Email Preparation and Sending

- **Overview:** Creates a personalized HTML welcome email and sends it to the verified user.
- **Nodes Involved:** Personalize Welcome Email, Send Welcome Email
- **Node Details:**

  **Personalize Welcome Email**
  - Type: Code node.
  - Configuration:
    - Extracts name and email from Data Sanitization.
    - Derives first name by splitting full name.
    - Constructs a rich HTML email template with branding, CTAs, and footer.
    - Sets email subject dynamically with first name.
    - Includes validation score for optional logging.
    - Outputs JSON with email content and metadata.
  - Inputs: Validation Decision true branch.
  - Outputs: Data for sending email.
  - Edge cases:
    - Large HTML (~8KB) could be problematic if email clients have limits.
    - Template URLs and branding placeholders must be updated manually.
  - Sticky Note: Describes template details and customization points.

  **Send Welcome Email**
  - Type: Gmail node.
  - Configuration:
    - Sends to email address from JSON.
    - Email body and subject from personalization node.
    - Uses OAuth2 credentials for Gmail account.
  - Inputs: Personalize Welcome Email output.
  - Outputs: Success/failure of email sending.
  - Edge cases:
    - Gmail OAuth2 token expiration or quota limits can cause failures.
    - Email sending errors should be monitored.
  
#### 2.6 Invalid Email Handling

- **Overview:** Prepares an email to notify the user of an invalid email and stops the workflow.
- **Nodes Involved:** Prepare Correction Email, Stop and Error
- **Node Details:**

  **Prepare Correction Email**
  - Type: Code node.
  - Configuration:
    - Extracts user name and email.
    - Detects common typos in email domain and suggests corrections.
    - Creates an HTML email explaining the issue, including a "Try Again" CTA.
    - Sets subject line accordingly.
    - Outputs JSON with email content and metadata.
  - Inputs: Validation Decision false branch.
  - Outputs: Data for sending correction email (although no send node is present).
  - Edge cases:
    - Limited typo detection; some invalid emails may not get suggestions.
    - No actual email send implemented; workflow stops next.
  
  **Stop and Error**
  - Type: Stop and Error node.
  - Configuration:
    - Stops workflow with error message `"Invalid email address"`.
  - Inputs: Prepare Correction Email output.
  - Outputs: Workflow halts here.
  - Edge cases:
    - Stops workflow with no email sent or logging for invalid users; could be improved.
  - Sticky Note: Advises replacing with a send email node and logging for better UX.

#### 2.7 Logging and Notification

- **Overview:** Logs verified user data to Google Sheets and notifies the team on Slack.
- **Nodes Involved:** Log Valid Users, Team Notification
- **Node Details:**

  **Log Valid Users**
  - Type: Google Sheets node.
  - Configuration:
    - Operation: Append or update row matching on email to avoid duplicates.
    - Fields logged: Name, Email, Status, Verified At, Original Email, Validation Score.
    - Uses Google Sheets OAuth2 credentials.
    - Targets a specific sheet and document (names from cached credentials).
  - Inputs: Send Welcome Email output.
  - Outputs: Confirmation of logging.
  - Edge cases:
    - API quota or permission errors possible.
    - Matching relies on email uniqueness.
  - Sticky Note: Explains audit and analytics usage.

  **Team Notification**
  - Type: Slack node.
  - Configuration:
    - Sends a formatted message to Slack channel `#new-signup`.
    - Message includes name, email, time, and verification status.
    - Uses Slack OAuth2 credentials.
  - Inputs: Log Valid Users output.
  - Outputs: Slack message confirmation.
  - Edge cases:
    - Slack API limits or permission errors.
  - Sticky Note: Notes channel and audience, suggests digest if volume high.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                      | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                              |
|-------------------------|-----------------------------|------------------------------------|--------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook                 | Webhook                     | Receives signup data via POST      | -                        | Data Sanitization           | Explains webhook config, required fields, debugging tips, and rate limiting advice.                    |
| Data Sanitization       | Code                        | Cleans and validates input data    | Webhook                   | Email Validation            | Details sanitization actions, common errors, and output format example.                               |
| Email Validation        | VerifiEmail                 | Validates email via API             | Data Sanitization          | Validation Decision         | Describes validation checks, API credentials, and possible fallback needs.                            |
| Validation Decision     | If                          | Routes flow based on validity      | Email Validation           | Personalize Welcome Email, Prepare Correction Email | Explains decision logic and monitoring.                                                               |
| Personalize Welcome Email| Code                        | Builds personalized welcome email  | Validation Decision (true) | Send Welcome Email          | Details email template, customization points, and data fields used.                                   |
| Send Welcome Email      | Gmail                       | Sends welcome email to user        | Personalize Welcome Email  | Log Valid Users             | -                                                                                                      |
| Prepare Correction Email| Code                        | Builds correction email for invalid| Validation Decision (false)| Stop and Error             | Explains typo detection and email content; no actual send implemented.                               |
| Stop and Error          | Stop and Error              | Stops workflow with error          | Prepare Correction Email   | -                          | Advises replacing with send email and logging for better user experience.                             |
| Log Valid Users         | Google Sheets               | Logs verified users for auditing   | Send Welcome Email         | Team Notification           | Explains logging fields and usage for analytics.                                                     |
| Team Notification       | Slack                       | Sends Slack alert on new signup    | Log Valid Users            | -                          | Notes target channel, message content, and alert volume management advice.                            |
| Sticky Note             | Sticky Note                 | Workflow overview and stats        | -                         | -                          | Describes purpose, owner, status, and monitoring instructions.                                       |
| Sticky Note1            | Sticky Note                 | Webhook configuration details      | -                         | -                          | Covers webhook configuration and debugging tips.                                                     |
| Sticky Note2            | Sticky Note                 | Data sanitization details           | -                         | -                          | Describes sanitization logic and common errors.                                                      |
| Sticky Note3            | Sticky Note                 | Email validation explanation        | -                         | -                          | Details VerifiEmail API usage and fallback notes.                                                    |
| Sticky Note4            | Sticky Note                 | Validation decision logic           | -                         | -                          | Explains branching conditions and monitoring suggestions.                                           |
| Sticky Note5            | Sticky Note                 | Welcome email template info         | -                         | -                          | Notes template customization and data used.                                                         |
| Sticky Note6            | Sticky Note                 | Google Sheets logging info          | -                         | -                          | Explains logging operation and fields.                                                              |
| Sticky Note7            | Sticky Note                 | Slack notification overview         | -                         | -                          | Details Slack channel usage and team audience.                                                      |
| Sticky Note8            | Sticky Note                 | Stop and error explanation          | -                         | -                          | Advises improving error handling with user notification and logging.                                |
| Sticky Note9            | Sticky Note                 | Maintenance checklist               | -                         | -                          | Lists weekly/monthly maintenance tasks and quick fixes.                                            |
| Sticky Note10           | Sticky Note                 | Quick reference and testing tips    | -                         | -                          | Provides links, targets, test curl command, and owner info.                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook
   - HTTP Method: POST
   - Path: `new-signup`
   - Authentication: None or as required
   - Save and activate to listen for incoming signups.

2. **Add Data Sanitization Node**
   - Type: Code
   - JavaScript Code:
     ```js
     const items = $input.all();
     return items.map(item => {
       const data = item.json.body || item.json || {};
       const name = (data.name || '').trim();
       const email = (data.email || '').toLowerCase().trim();
       if (!name || !email) {
         throw new Error(`Missing required fields. Received: ${JSON.stringify(data)}`);
       }
       return {
         json: {
           name,
           email,
           original_email: data.email || email,
           received_at: new Date().toISOString()
         }
       };
     });
     ```
   - Connect Webhook output to this node input.

3. **Add Email Validation Node**
   - Type: VerifiEmail
   - Email: Expression - `{{$json.email}}`
   - Set credentials with valid VerifiEmail API key.
   - Connect Data Sanitization output here.

4. **Create Validation Decision (If) Node**
   - Condition: `$json.valid == true`
   - True Branch: For valid emails.
   - False Branch: For invalid emails.
   - Connect Email Validation output here.

5. **True Branch: Personalize Welcome Email**
   - Type: Code
   - Run once per item.
   - JavaScript Code:
     ```js
     const name = $('Data Sanitization').item.json.name;
     const email = $('Data Sanitization').item.json.email;
     const firstName = name.split(' ')[0];
     const validationScore = $json.quality_score || 'N/A';

     return {
       json: {
         name,
         firstName,
         email,
         subject: `Welcome to Our Platform, ${firstName}! 🎉`,
         emailBody: `<!DOCTYPE html>...custom HTML template with CTA URLs...`,
         timestamp: new Date().toISOString(),
         status: 'verified',
         validationScore
       }
     };
     ```
   - Connect Validation Decision true output here.

6. **Send Welcome Email**
   - Type: Gmail
   - Send To: Expression `{{$json.email}}`
   - Subject: Expression `{{$json.subject}}`
   - Message: Expression `{{$json.emailBody}}`
   - Set Gmail OAuth2 credentials with appropriate scopes.
   - Connect Personalize Welcome Email output here.

7. **Log Valid Users**
   - Type: Google Sheets
   - Operation: Append or update
   - Set sheet and document IDs for "Verified Users"
   - Map columns: Name, Email, Status, Verified At, Original Email, Validation Score
   - Connect Send Welcome Email output here.
   - Set Google Sheets OAuth2 credentials.

8. **Team Notification**
   - Type: Slack
   - Message Text:
     ```
     🎉 *New Verified Signup!*
     👤 *Name:* {{ $('Personalize Welcome Email').item.json.name }}
     📧 *Email:* {{ $('Personalize Welcome Email').item.json.email }}
     ⏰ *Time:* {{ $('Personalize Welcome Email').item.json.timestamp }}
     ✅ *Status:* Verified & Welcomed
     ```
   - Channel: `#new-signup` (select from workspace)
   - Use Slack OAuth2 credentials.
   - Connect Log Valid Users output here.

9. **False Branch: Prepare Correction Email**
   - Type: Code
   - JavaScript Code:
     ```js
     const name = $('Data Sanitization').item.json.name;
     const email = $('Data Sanitization').item.json.email;
     const firstName = name.split(' ')[0];
     const validationData = $json;
     const reason = validationData.deliverability || 'invalid';

     const commonTypos = {
       'gmial': 'gmail',
       'gmai': 'gmail',
       'yahooo': 'yahoo',
       'yaho': 'yahoo',
       'hotmial': 'hotmail',
       'outlok': 'outlook',
       'outloo': 'outlook'
     };

     let suggestion = '';
     for (const [typo, correct] of Object.entries(commonTypos)) {
       if (email.includes(typo)) {
         suggestion = email.replace(typo, correct);
         break;
       }
     }

     return {
       json: {
         name,
         firstName,
         email,
         suggestion,
         reason,
         subject: `Please verify your email address`,
         emailBody: `<!DOCTYPE html>...correction email HTML with suggestion and retry link...`,
         status: 'invalid',
         timestamp: new Date().toISOString()
       }
     };
     ```
   - Connect Validation Decision false output here.

10. **Stop and Error Node**
    - Type: Stop and Error
    - Error message: `"Invalid email address"`
    - Connect Prepare Correction Email output here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                               | Context or Link                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow purpose: Validate new signups and send personalized welcome emails with high accuracy and logging.                                                                                 | Sticky Note overview                              |
| Webhook endpoint: POST `/new-signup` with JSON `{ "name": "string", "email": "string" }`. Use "Listen for Test Event" for debugging.                                                       | Sticky Note1                                      |
| VerifiEmail API documentation and credential setup: https://verifi.email                                                                                                                    | Sticky Note3                                      |
| Suggested maintenance: Weekly checks on API usage, Slack alerts, sample signups; Monthly update of templates, archive data, check Gmail limits.                                            | Sticky Note9                                      |
| Testing example curl command: `curl -X POST [webhook] -H "Content-Type: application/json" -d '{"name":"Test","email":"test@gmail.com"}'`                                                     | Sticky Note10                                     |
| Slack notification channel: `#new-signup`, targeted for Sales, Marketing, Product teams. For high volumes, consider digest or summary messages.                                            | Sticky Note7                                      |
| Current stop on invalid email could be improved to send correction email and log invalid attempts for better user experience and tracking.                                                  | Sticky Note8                                      |
| Personalize welcome email template: Update company name, CTA URLs, brand colors, and footer copyright before deployment.                                                                     | Sticky Note5                                      |

---

**Disclaimer:**  
The above content is generated exclusively from the provided n8n workflow JSON. The workflow respects all current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---