Newsletter Signup Flow with Email Verification API, Gmail & Google Sheets Tracking

https://n8nworkflows.xyz/workflows/newsletter-signup-flow-with-email-verification-api--gmail---google-sheets-tracking-10138


# Newsletter Signup Flow with Email Verification API, Gmail & Google Sheets Tracking

### 1. Workflow Overview

This workflow automates the process of handling newsletter signups by validating incoming subscriber data, verifying the submitted email address through an external API, and sending a personalized welcome email if valid. It also logs incomplete or invalid submissions into Google Sheets for tracking and analytics.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Initial Validation:** Receives signup data via a webhook and checks for required fields (name and email).
- **1.2 Email Verification:** Uses the Verifi Email API to validate the email’s authenticity and deliverability.
- **1.3 Conditional Routing:** Routes the flow based on email validity to either send a welcome email or handle invalid emails.
- **1.4 Email Generation and Sending:** Creates a branded, personalized HTML welcome email and sends it via Gmail.
- **1.5 Logging Invalid Data and Emails:** Logs incomplete submissions and invalid email details into dedicated Google Sheets tabs.
- **1.6 Master Log Aggregation:** Formats and consolidates all events (valid and invalid) into a master log sheet for comprehensive tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initial Validation

- **Overview:**  
  This block accepts new newsletter signup data via a webhook and verifies the presence of essential fields (name and email). If data is incomplete, the submission is logged separately.

- **Nodes Involved:**  
  - Webhook - Newsletter Signup  
  - Check Data Completeness (IF)  
  - Log Incomplete Submissions (Google Sheets)  
  - Sticky Notes: Webhook Info, Data Validation, Log Invalid

- **Node Details:**

  1. **Webhook - Newsletter Signup**  
     - *Type:* Webhook  
     - *Role:* Entry point receiving POST requests at path `/newsletter-signup`.  
     - *Configuration:* Accepts JSON payload with at least `name`, `email`, and optionally `source`. Responds with the last executed node’s output.  
     - *Input/Output:* Input via HTTP POST; output to Check Data Completeness.  
     - *Edge Cases:* Missing or malformed JSON payload; rejected by Check Data Completeness node if required fields missing.

  2. **Check Data Completeness**  
     - *Type:* IF node  
     - *Role:* Validates that both `email` and `name` fields in the webhook body are non-empty strings.  
     - *Configuration:* Checks `{{ $json.body.email }}` and `{{ $json.body.name }}` are not empty.  
     - *Input/Output:* Input from webhook; true branch leads to email verification, false branch to logging incomplete submissions.  
     - *Edge Cases:* Empty strings or missing keys cause false branch.

  3. **Log Incomplete Submissions**  
     - *Type:* Google Sheets Append  
     - *Role:* Records incomplete submission details (`name`, `email`, "Incomplete data" reason, timestamp) in sheet `Invalid_Submissions`.  
     - *Configuration:* Uses OAuth2 Google Sheets credentials; target sheet and document IDs must be set.  
     - *Input/Output:* Input from false branch of Check Data Completeness.  
     - *Edge Cases:* API failures, credential errors, or missing sheet/document IDs.

  4. **Sticky Notes**  
     - Provide documentation and context for data expectations and validation logic.

---

#### 2.2 Email Verification

- **Overview:**  
  Verifies the email address using the Verifi Email API, checking format, domain, MX records, and disposable email status.

- **Nodes Involved:**  
  - Verifi Email (API node)  
  - Sticky Note - Email Verification

- **Node Details:**

  1. **Verifi Email**  
     - *Type:* Verifi Email API node  
     - *Role:* Calls external API with the email from webhook data to validate authenticity.  
     - *Configuration:* Passes `{{ $json.body.email }}` as the email parameter; credentials must be configured.  
     - *Input/Output:* Input from true branch of Check Data Completeness; output to Check Email Validity node.  
     - *Edge Cases:* API key invalid, rate limits, network failures, malformed email input.

  2. **Sticky Note - Email Verification**  
     - Explains the checks performed by the Verifi API and expected output fields.

---

#### 2.3 Conditional Routing Based on Email Validity

- **Overview:**  
  Routes the flow depending on whether the email is valid or invalid as per the API response.

- **Nodes Involved:**  
  - Check Email Validity (IF)  
  - Prepare Invalid Email Message (Code)  
  - Generate Welcome Email HTML (Code)  
  - Sticky Note - Conditional Logic, Invalid Handler

- **Node Details:**

  1. **Check Email Validity**  
     - *Type:* IF node  
     - *Role:* Checks if the `valid` field from Verifi API response is `true`.  
     - *Configuration:* Condition: `{{ $json.valid }} == true`  
     - *Input/Output:* Input from Verifi Email node; true branch to Generate Welcome Email HTML, false branch to Prepare Invalid Email Message.  
     - *Edge Cases:* Missing or malformed `valid` field; fallback to false branch.

  2. **Prepare Invalid Email Message**  
     - *Type:* Code node (JavaScript)  
     - *Role:* Parses Verifi API response to determine specific invalid email reasons (e.g., no MX record, disposable email) and prepares detailed logging information.  
     - *Key Logic:* Checks various flags (`validMxRecord`, `rfcCompliant`, `disposable`, `spoofFree`) and assigns reasons accordingly.  
     - *Input/Output:* Input from false branch of Check Email Validity; output to Log Invalid Emails.  
     - *Edge Cases:* Unexpected API response structure or missing details.

  3. **Generate Welcome Email HTML**  
     - *Type:* Code node (JavaScript)  
     - *Role:* Creates a personalized, branded HTML welcome email using subscriber's first name and email source, embedding confirmation and unsubscribe links.  
     - *Key Logic:* Pulls data directly from the Webhook node to avoid data loss, builds responsive HTML email with inline styles and dynamic content.  
     - *Input/Output:* Input from true branch of Check Email Validity; output to Send Welcome Email node.  
     - *Edge Cases:* Missing or malformed name/email; encoding issues in URLs.

  4. **Sticky Notes**  
     - Document conditional routing logic and invalid email handling purpose.

---

#### 2.4 Email Sending

- **Overview:**  
  Sends the generated welcome email to the validated subscriber using Gmail OAuth2 credentials.

- **Nodes Involved:**  
  - Send Welcome Email (Gmail)  
  - Sticky Note - Send Email

- **Node Details:**

  1. **Send Welcome Email**  
     - *Type:* Gmail node  
     - *Role:* Sends the personalized HTML email to the subscriber’s email address.  
     - *Configuration:* Uses Gmail OAuth2 credentials; recipient and content dynamically set from previous node outputs (`email`, `htmlEmail`, `subject`). Sender name configured as "Your Brand Team".  
     - *Input/Output:* Input from Generate Welcome Email HTML node; no further outputs.  
     - *Edge Cases:* OAuth token expiry, Gmail API limits, invalid email address format despite verification.

  2. **Sticky Note - Send Email**  
     - Explains email sending configuration and personalization.

---

#### 2.5 Logging Invalid Data and Emails

- **Overview:**  
  Logs incomplete submissions and invalid email verifications into separate sheets for audit and troubleshooting.

- **Nodes Involved:**  
  - Log Incomplete Submissions (Google Sheets) [from 2.1]  
  - Log Invalid Emails (Google Sheets)  
  - Sticky Note - Log Invalid  
  - Sticky Note - Log Invalid Emails

- **Node Details:**

  1. **Log Invalid Emails**  
     - *Type:* Google Sheets Append  
     - *Role:* Logs detailed invalid email data including reasons, technical explanations, domain info, and timestamps in sheet `Invalid_Emails`.  
     - *Configuration:* OAuth2 Google Sheets credentials; target sheet and document IDs required.  
     - *Input/Output:* Input from Prepare Invalid Email Message node.  
     - *Edge Cases:* API failures, missing sheet/document, malformed log data.

  2. **Sticky Notes**  
     - Describe the purpose and data fields logged for invalid data and emails.

---

#### 2.6 Master Log Aggregation

- **Overview:**  
  Consolidates all processed data—both valid and invalid—into a master Google Sheet for centralized tracking and analytics.

- **Nodes Involved:**  
  - Format Master Log Entry (Code)  
  - Log to Master Sheet (Google Sheets)  
  - Sticky Note - Format  
  - Sticky Note - Master Log

- **Node Details:**

  1. **Format Master Log Entry**  
     - *Type:* Code node (JavaScript)  
     - *Role:* Standardizes and enriches the data structure for the master log, including fields like verification result, email sent status, and source tracking.  
     - *Key Logic:* Determines if email was valid, sets human-readable verification result, and organizes all relevant metadata.  
     - *Input/Output:* Input from both Log Invalid Emails and Send Welcome Email branches; output to Log to Master Sheet.  
     - *Edge Cases:* Missing fields or inconsistent data could affect logging accuracy.

  2. **Log to Master Sheet**  
     - *Type:* Google Sheets Append  
     - *Role:* Appends the formatted log entry to the `Master_Log` sheet in the main tracking document.  
     - *Configuration:* OAuth2 Google Sheets credentials required; sheet and document IDs must be set.  
     - *Input/Output:* Input from Format Master Log Entry node.  
     - *Edge Cases:* Google API errors, permission issues.

  3. **Sticky Notes**  
     - Explain the purpose of the master log, fields included, and use cases for analytics.

---

#### 2.7 Credentials and Setup Notes

- **Overview:**  
  Lists required credentials and setup instructions necessary to run the workflow properly.

- **Nodes Involved:**  
  - Sticky Note - CREDENTIALS SETUP

- **Node Details:**

  1. **Sticky Note - CREDENTIALS SETUP**  
     - Details all credentials to configure: Google Sheets OAuth2, Gmail OAuth2, Verifi Email API.  
     - Specifies the need to replace placeholder sheet IDs with actual Google Sheets document and tab IDs.  
     - Advises on creating three sheets: Invalid_Submissions, Invalid_Emails, Master_Log.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                          | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                  |
|----------------------------|------------------------|----------------------------------------|---------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| Webhook - Newsletter Signup| Webhook                | Entry point for signup data             | HTTP POST request               | Check Data Completeness          | Expected Payload sample JSON                                                                |
| Sticky Note - Webhook Info  | Sticky Note            | Explains webhook input format           | None                           | None                           | Describes expected JSON payload                                                             |
| Check Data Completeness     | IF                     | Validates presence of email and name    | Webhook - Newsletter Signup     | Verifi Email (true), Log Incomplete Submissions (false) | Explains data validation checks                                                             |
| Log Incomplete Submissions  | Google Sheets          | Logs incomplete signup entries           | Check Data Completeness (false) | None                           | Logs incomplete data to Invalid_Submissions sheet                                            |
| Sticky Note - Data Validation| Sticky Note            | Describes data validation purpose        | None                           | None                           | Details validation logic                                                                     |
| Verifi Email                | Verifi Email API       | Validates email address via API          | Check Data Completeness (true)  | Check Email Validity             | Describes email verification checks                                                         |
| Sticky Note - Email Verification| Sticky Note        | Explains email verification process      | None                           | None                           | Details Verifi API checks                                                                   |
| Check Email Validity        | IF                     | Routes flow based on email validity      | Verifi Email                   | Generate Welcome Email HTML (true), Prepare Invalid Email Message (false) | Describes conditional routing                                                              |
| Prepare Invalid Email Message| Code                   | Parses invalid email reasons             | Check Email Validity (false)    | Log Invalid Emails              | Explains invalid email handling                                                             |
| Sticky Note - Invalid Handler| Sticky Note            | Describes invalid email handling         | None                           | None                           | Explains reasons for invalid email logging                                                  |
| Log Invalid Emails          | Google Sheets          | Logs invalid emails details               | Prepare Invalid Email Message   | Format Master Log Entry          | Logs invalid emails to Invalid_Emails sheet                                                 |
| Sticky Note - Log Invalid Emails| Sticky Note          | Explains invalid emails logging          | None                           | None                           | Details fields logged for invalid emails                                                    |
| Generate Welcome Email HTML | Code                   | Creates personalized HTML welcome email  | Check Email Validity (true)     | Send Welcome Email              | Explains email content generation                                                           |
| Sticky Note - HTML Generation| Sticky Note            | Describes welcome email generation       | None                           | None                           | Details HTML email features                                                                 |
| Send Welcome Email          | Gmail                  | Sends welcome email to subscriber        | Generate Welcome Email HTML     | None                           | Explains email sending configuration                                                       |
| Sticky Note - Send Email    | Sticky Note            | Describes email sending configuration    | None                           | None                           | Details send email node setup                                                               |
| Format Master Log Entry     | Code                   | Standardizes data for master log          | Log Invalid Emails, Send Welcome Email | Log to Master Sheet             | Explains data formatting for master log                                                    |
| Sticky Note - Format        | Sticky Note            | Describes master log formatting           | None                           | None                           | Details formatting rationale                                                                |
| Log to Master Sheet         | Google Sheets          | Appends consolidated log to master sheet | Format Master Log Entry         | None                           | Logs all processed entries to Master_Log sheet                                              |
| Sticky Note - Master Log    | Sticky Note            | Explains master logging purpose           | None                           | None                           | Details analytics and reporting uses                                                       |
| Sticky Note - CREDENTIALS SETUP| Sticky Note          | Lists required credentials and setup steps| None                           | None                           | Describes all credentials needed and sheet setup                                           |
| Sticky Note - Workflow Summary| Sticky Note            | Summarizes entire workflow logic          | None                           | None                           | Overview of flow and error handling                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Path: `newsletter-signup`  
   - HTTP Method: POST  
   - Response Mode: Last Node  
   - Purpose: Receive signup JSON with fields `name`, `email`, `source`.

2. **Add IF Node "Check Data Completeness":**  
   - Condition: Check that `{{ $json.body.email }}` and `{{ $json.body.name }}` are not empty.  
   - True branch: Continue to email verification.  
   - False branch: Log incomplete submissions.

3. **Add Google Sheets Node "Log Incomplete Submissions":**  
   - Operation: Append  
   - Document ID & Sheet ID: Your Google Sheets document and `Invalid_Submissions` tab.  
   - Columns:  
     - name: `={{ $json.body.name || 'N/A' }}`  
     - email: `={{ $json.body.email || 'N/A' }}`  
     - reason: `"Incomplete data"`  
     - timestamp: `={{ $now.toISO() }}`  
   - Credentials: Google Sheets OAuth2.

4. **Add Verifi Email Node:**  
   - Email: `={{ $json.body.email }}`  
   - Credentials: Verifi Email API key.

5. **Add IF Node "Check Email Validity":**  
   - Condition: `{{ $json.valid }} == true`  
   - True branch: Generate welcome email.  
   - False branch: Handle invalid email.

6. **Add Code Node "Generate Welcome Email HTML":**  
   - JavaScript code generates a responsive HTML email using:  
     - Subscriber's first name from webhook data.  
     - Email and source for confirmation link.  
     - Well-styled branded email content with unsubscribe and preferences links.

7. **Add Gmail Node "Send Welcome Email":**  
   - To: `={{ $json.email }}`  
   - Subject: `={{ $json.subject }}`  
   - Message (HTML): `={{ $json.htmlEmail }}`  
   - Sender Name: "Your Brand Team"  
   - Credentials: Gmail OAuth2.

8. **Add Code Node "Prepare Invalid Email Message":**  
   - Parses Verifi API response to assign invalid reasons like:  
     - Invalid or non-existent email domain  
     - RFC compliance issues  
     - Disposable email detection  
     - Spoofing suspicion  
   - Outputs augmented JSON with detailed info for logging.

9. **Add Google Sheets Node "Log Invalid Emails":**  
   - Append operation to `Invalid_Emails` sheet.  
   - Columns include: `name`, `email`, `domain`, `reason`, `technical_reason`, `timestamp`.  
   - Credentials: Google Sheets OAuth2.

10. **Add Code Node "Format Master Log Entry":**  
    - Collects and standardizes fields from both valid and invalid flows:  
      - `verification_result` (Valid/Invalid)  
      - `email_sent` (Yes/No)  
      - Other verification metadata.

11. **Add Google Sheets Node "Log to Master Sheet":**  
    - Append to `Master_Log` sheet.  
    - Columns: `name`, `email`, `domain`, `source`, `timestamp`, `disposable`, `email_sent`, `verification_score`, `verification_result`.  
    - Credentials: Google Sheets OAuth2.

12. **Connect nodes according to the logical flow:**  
    - Webhook → Check Data Completeness  
    - Check Data Completeness (true) → Verifi Email → Check Email Validity  
    - Check Data Completeness (false) → Log Incomplete Submissions  
    - Check Email Validity (true) → Generate Welcome Email HTML → Send Welcome Email → Format Master Log Entry  
    - Check Email Validity (false) → Prepare Invalid Email Message → Log Invalid Emails → Format Master Log Entry  
    - Format Master Log Entry → Log to Master Sheet

13. **Add Sticky Notes:**  
    - At key points to document expected data, logic decisions, configuration notes, and credentials setup.

14. **Credential Setup:**  
    - Configure Google Sheets OAuth2 with appropriate scopes and access to your target spreadsheet.  
    - Configure Gmail OAuth2 for sending emails on behalf of your brand.  
    - Obtain Verifi Email API key from https://verifi.email and configure its credential.  
    - Replace all placeholder Google Sheets document and sheet IDs with your actual IDs.  
    - Create the Google Sheets spreadsheet with three tabs: `Invalid_Submissions`, `Invalid_Emails`, and `Master_Log`.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                       |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Verifi Email API key required: obtain from https://verifi.email                                      | Credential setup                                      |
| Gmail OAuth2 setup required for sending emails via Gmail API                                         | Credential setup                                      |
| Google Sheets OAuth2 required to log data to spreadsheets                                            | Credential setup                                      |
| Spreadsheet tabs to create: Invalid_Submissions, Invalid_Emails, Master_Log                          | Setup instructions                                   |
| Confirmation link and unsubscribe URLs are dynamically generated and embedded in the welcome email  | Email personalization and user experience            |
| The workflow includes retry mechanisms on API calls to handle transient errors                       | Reliability and error handling                         |
| The workflow logs all invalid and incomplete submissions separately for audit and troubleshooting    | Data quality monitoring                               |
| The master log provides a complete audit trail for analytics and reporting                           | Analytics and reporting                               |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.