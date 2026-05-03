Streamline Client Onboarding with PDF, Trello, Slack, Gmail & Airtable

https://n8nworkflows.xyz/workflows/streamline-client-onboarding-with-pdf--trello--slack--gmail---airtable-8930


# Streamline Client Onboarding with PDF, Trello, Slack, Gmail & Airtable

### 1. Workflow Overview

This **Client Onboarding Workflow** automates the onboarding process of new clients for agencies, SaaS startups, and B2B companies. It integrates multiple platforms (Google Sheets, Trello, Slack, Gmail, Airtable) to streamline client registration, validation, tier assignment, task creation, communication, and reporting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Email Validation:** Receives client onboarding requests via a webhook and validates client email addresses to filter legitimate entries.
- **1.2 Client Logging and Tier Assignment:** Logs validated client data into Google Sheets and assigns onboarding tiers and priorities based on the client’s plan.
- **1.3 Task Creation and Welcome PDF Generation:** Creates a Trello task card for the Customer Success Management (CSM) team and generates a personalized Welcome Pack PDF.
- **1.4 Team and Client Communication:** Sends Slack notifications to internal teams and personalized welcome emails with the attached PDF to clients.
- **1.5 Data Archiving:** Archives onboarding details in Airtable for record-keeping and future reference.
- **1.6 Weekly Reporting:** Periodically generates and emails a weekly summary report of new onboarded clients to management.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Email Validation

**Overview:**  
This block captures client onboarding form submissions via a webhook, validates the submitted email address to ensure authenticity, and blocks further processing for invalid emails.

**Nodes Involved:**  
- Client Onboarding Webhook  
- Email Validation  
- Email Validity Check

**Node Details:**

- **Client Onboarding Webhook**  
  - *Type:* Webhook  
  - *Role:* Entry point receiving POST requests with client onboarding data (name, email, company, plan).  
  - *Config:* Path set to `/client-onboarding`, HTTP method POST.  
  - *Inputs:* External HTTP POST request.  
  - *Outputs:* Emits the client data payload to Email Validation.  
  - *Edge Cases:* Missing or malformed payloads; webhook authentication not configured (potential security risk).  
  - *Version:* 2.1

- **Email Validation**  
  - *Type:* VerifiEmail node  
  - *Role:* Checks if the provided email is valid, reducing fake or erroneous signups.  
  - *Config:* Email field dynamically set from webhook payload (`$json.body.email`).  
  - *Credentials:* VerifiEmail API  
  - *Inputs:* Client data from webhook.  
  - *Outputs:* Emits validation result to Email Validity Check.  
  - *Edge Cases:* API rate limits, network failures, or invalid email formats.  
  - *Version:* 1

- **Email Validity Check**  
  - *Type:* If node  
  - *Role:* Conditional gate allowing only valid emails to proceed.  
  - *Config:* Checks if `$json.valid` equals true.  
  - *Inputs:* Validation result.  
  - *Outputs:* If true, passes client data to Log Client node; otherwise, stops workflow.  
  - *Edge Cases:* Expression errors if email validation response format changes.  
  - *Version:* 2.2

---

#### 1.2 Client Logging and Tier Assignment

**Overview:**  
Logs validated client information into Google Sheets and assigns onboarding tiers and priorities based on the client’s selected plan.

**Nodes Involved:**  
- Log Client  
- Assign Tier Logic

**Node Details:**

- **Log Client**  
  - *Type:* Google Sheets (append operation)  
  - *Role:* Records client details (name, plan, email, company, timestamp, status) into a Google Sheet for tracking.  
  - *Config:* Writes to Sheet1 in a specific Google Sheets document; columns mapped from webhook data; sets status as "Onboarding Started".  
  - *Credentials:* Google Sheets OAuth2  
  - *Inputs:* Validated client data from Email Validity Check.  
  - *Outputs:* Passes data to Assign Tier Logic.  
  - *Edge Cases:* Google Sheets API quota limits, permission errors, network issues.  
  - *Version:* 4.7

- **Assign Tier Logic**  
  - *Type:* Code (JavaScript)  
  - *Role:* Determines onboarding tier (Premium, Standard, Basic) and priority (High, Medium, Low) based on plan; generates onboarding ID and selects welcome email template.  
  - *Config:* Custom JS logic processing input items; uses plan field to assign tier and priority; generates unique onboarding_id with timestamp.  
  - *Inputs:* Client data from Google Sheets.  
  - *Outputs:* Augmented client data to Create Trello Task Card.  
  - *Edge Cases:* Missing or unexpected plan values; date-based ID generation collisions unlikely but possible.  
  - *Version:* 2

---

#### 1.3 Task Creation and Welcome PDF Generation

**Overview:**  
Creates a Trello task card for the onboarding process and generates a personalized Welcome Pack PDF for the client.

**Nodes Involved:**  
- Create Trello Task Card  
- Generate Welcome PDF

**Node Details:**

- **Create Trello Task Card**  
  - *Type:* Trello node (create card)  
  - *Role:* Creates a new Trello card with client onboarding tasks for the CSM team.  
  - *Config:* Card name includes client name and plan; description lists client details and checklist for onboarding steps; due date set 3 days from current date.  
  - *Credentials:* Trello API  
  - *Inputs:* Client data from Assign Tier Logic.  
  - *Outputs:* Passes to Generate Welcome PDF.  
  - *Edge Cases:* Trello API limits, invalid list ID, network errors.  
  - *Version:* 1

- **Generate Welcome PDF**  
  - *Type:* HTML/CSS to PDF conversion node  
  - *Role:* Creates a branded PDF welcome pack personalized with client details, plan, priority, and onboarding ID.  
  - *Config:* Inline HTML template with dynamic data interpolation; includes company logo, client name, plan, next steps, and priority.  
  - *Credentials:* Htmlcsstopdf API  
  - *Inputs:* Client data from Trello card creation.  
  - *Outputs:* PDF URL passed to Send Slack Notification.  
  - *Edge Cases:* PDF generation failures, malformed HTML or missing data, API errors.  
  - *Version:* 1

---

#### 1.4 Team and Client Communication

**Overview:**  
Sends internal Slack notifications about new onboarded clients and emails clients their personalized welcome pack with the PDF attached.

**Nodes Involved:**  
- Send Slack Notification  
- Download PDF File  
- Send Welcome Email

**Node Details:**

- **Send Slack Notification**  
  - *Type:* Slack message node  
  - *Role:* Notifies internal teams in a specified Slack channel about the new client onboarding with key details and statuses.  
  - *Config:* Message templated with client name, company, plan, tier, onboarding ID, and priority; channel ID provided; unfurl media disabled.  
  - *Credentials:* Slack OAuth2  
  - *Inputs:* PDF URL and client data from Generate Welcome PDF.  
  - *Outputs:* Passes PDF URL to Download PDF File.  
  - *Edge Cases:* Slack API rate limits, invalid channel ID, token expiration.  
  - *Version:* 2.3

- **Download PDF File**  
  - *Type:* HTTP Request node  
  - *Role:* Downloads the generated PDF file from the provided URL to attach in email.  
  - *Config:* GET request to PDF URL; response expected as file binary.  
  - *Inputs:* PDF URL from Slack Notification node.  
  - *Outputs:* Passes binary PDF to Send Welcome Email.  
  - *Edge Cases:* Broken URLs, download timeouts, network errors.  
  - *Version:* 4.2

- **Send Welcome Email**  
  - *Type:* Gmail node (send email)  
  - *Role:* Sends a personalized welcome email to the client with the Welcome Pack PDF attached.  
  - *Config:* Email recipient set dynamically from client email; subject and message body personalized with client and onboarding info; PDF attached as binary.  
  - *Credentials:* Gmail OAuth2  
  - *Inputs:* Client email and binary PDF file.  
  - *Outputs:* Passes workflow data to Archive Data.  
  - *Edge Cases:* Gmail API quotas, attachment size limits, invalid recipient emails.  
  - *Version:* 2.1

---

#### 1.5 Data Archiving

**Overview:**  
Archives all onboarding-related details in Airtable, marking onboarding status as completed and storing metadata for future reporting.

**Nodes Involved:**  
- Archive Data

**Node Details:**

- **Archive Data**  
  - *Type:* Airtable node (create record)  
  - *Role:* Creates a new record in Airtable with client info, assigned tier, priority, onboarding ID, status ("Completed"), and timestamp.  
  - *Config:* Maps fields explicitly from the client data JSON; uses Airtable OAuth2 credentials; typecasting enabled for field compatibility.  
  - *Inputs:* Client data from Send Welcome Email.  
  - *Outputs:* End of onboarding data pipeline.  
  - *Edge Cases:* Airtable API rate limiting, permission errors, field mapping mismatches.  
  - *Version:* 2.1

---

#### 1.6 Weekly Reporting

**Overview:**  
On a weekly schedule, collects onboarding data from Airtable, processes statistics such as new clients count and tier distribution, then emails a summary report to management.

**Nodes Involved:**  
- Schedule Weekly Summary  
- Weekly Report - Data Collection  
- Weekly Report - Data Processing  
- Send Weekly Report Email

**Node Details:**

- **Schedule Weekly Summary**  
  - *Type:* Schedule Trigger node  
  - *Role:* Triggers the workflow every Monday at 9:00 AM to collect onboarding data.  
  - *Config:* Cron expression `0 9 * * 1` (9 AM every Monday).  
  - *Inputs:* Time-based trigger.  
  - *Outputs:* Activates Weekly Report - Data Collection.  
  - *Version:* 1.2

- **Weekly Report - Data Collection**  
  - *Type:* Airtable node (list records)  
  - *Role:* Retrieves all onboarding records from Airtable for processing.  
  - *Config:* Selects the onboarding list table in Airtable; no filters applied (retrieves all records).  
  - *Credentials:* Airtable OAuth2  
  - *Inputs:* Trigger from Schedule Weekly Summary.  
  - *Outputs:* Passes records to Weekly Report - Data Processing.  
  - *Version:* 2.1

- **Weekly Report - Data Processing**  
  - *Type:* Code (JavaScript)  
  - *Role:* Filters records from the past week, computes total new clients, plan and tier breakdowns, earliest onboarding date, and formats a report object.  
  - *Config:* Custom JS code manipulating date and data arrays.  
  - *Inputs:* Airtable records.  
  - *Outputs:* JSON-formatted report for email.  
  - *Edge Cases:* Date parsing errors, empty datasets.  
  - *Version:* 2

- **Send Weekly Report Email**  
  - *Type:* Gmail node (send email)  
  - *Role:* Sends the weekly onboarding summary to management email.  
  - *Config:* Recipient fixed as `manager@company.com`; subject and body dynamically filled with report data.  
  - *Credentials:* Gmail OAuth2  
  - *Inputs:* JSON report from data processing node.  
  - *Outputs:* End of weekly reporting flow.  
  - *Edge Cases:* Email delivery issues, formatting errors in message body.  
  - *Version:* 2.1

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                        | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                                  |
|---------------------------|-------------------------|-------------------------------------|------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Client Onboarding Webhook  | Webhook                 | Entry point for client onboarding   | -                            | Email Validation               | See note: captures client details from form submission, triggers validation.                                                  |
| Email Validation          | VerifiEmail             | Validates client email authenticity | Client Onboarding Webhook     | Email Validity Check           | Email validation to avoid fake signups.                                                                                      |
| Email Validity Check       | If                      | Checks if email is valid             | Email Validation             | Log Client                    | Only valid emails proceed.                                                                                                   |
| Log Client                | Google Sheets           | Logs client details in Google Sheets | Email Validity Check          | Assign Tier Logic             | Logs client info for record-keeping and tracking.                                                                             |
| Assign Tier Logic         | Code                    | Assigns onboarding tier and priority | Log Client                   | Create Trello Task Card       | Assigns Basic, Pro, Premium tiers based on plan.                                                                              |
| Create Trello Task Card    | Trello                  | Creates onboarding task card in Trello | Assign Tier Logic            | Generate Welcome PDF          | Creates task card with onboarding checklist.                                                                                  |
| Generate Welcome PDF       | HTMLCSSToPDF            | Generates personalized welcome PDF  | Create Trello Task Card       | Send Slack Notification       | Generates branded Welcome Pack PDF.                                                                                           |
| Send Slack Notification    | Slack                   | Notifies team of new client onboarding | Generate Welcome PDF          | Download PDF File             | Slack notification with onboarding details.                                                                                   |
| Download PDF File          | HTTP Request            | Downloads generated PDF              | Send Slack Notification       | Send Welcome Email            | Downloads PDF for email attachment.                                                                                           |
| Send Welcome Email         | Gmail                   | Sends personalized welcome email    | Download PDF File             | Archive Data                  | Sends email with PDF to client.                                                                                               |
| Archive Data               | Airtable                | Archives client onboarding data     | Send Welcome Email            | -                            | Stores onboarding details in Airtable.                                                                                        |
| Schedule Weekly Summary    | Schedule Trigger        | Triggers weekly onboarding report   | -                            | Weekly Report - Data Collection| Scheduled weekly report trigger.                                                                                              |
| Weekly Report - Data Collection | Airtable          | Collects onboarding records         | Schedule Weekly Summary       | Weekly Report - Data Processing| Retrieves client onboarding data.                                                                                            |
| Weekly Report - Data Processing | Code              | Processes data into weekly summary  | Weekly Report - Data Collection| Send Weekly Report Email      | Filters and compiles weekly client onboarding metrics.                                                                       |
| Send Weekly Report Email   | Gmail                   | Sends weekly onboarding summary     | Weekly Report - Data Processing| -                            | Emails weekly summary to management.                                                                                          |
| Sticky Note               | Sticky Note             | Workflow overview note               | -                            | -                            | Main overview of workflow, purpose, and components.                                                                           |
| Sticky Note1              | Sticky Note             | Email validation overview            | -                            | -                            | Details on client capture and email validation process.                                                                       |
| Sticky Note2              | Sticky Note             | Client logging and tier assignment   | -                            | -                            | Logging client data and tier assignment explanation.                                                                          |
| Sticky Note3              | Sticky Note             | Task creation and PDF generation     | -                            | -                            | Outlines Trello task and PDF generation steps.                                                                                |
| Sticky Note4              | Sticky Note             | Team/client communications           | -                            | -                            | Slack notifications and welcome emails.                                                                                      |
| Sticky Note5              | Sticky Note             | Data storage in Airtable             | -                            | -                            | Archiving client onboarding details.                                                                                          |
| Sticky Note6              | Sticky Note             | Weekly reporting block               | -                            | -                            | Weekly trigger, data collection, processing, and email report.                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Name: `Client Onboarding Webhook`  
   - Path: `/client-onboarding`  
   - HTTP Method: POST  
   - Purpose: Receive client onboarding data (name, email, company, plan).

2. **Add VerifiEmail node**  
   - Name: `Email Validation`  
   - Email: Set expression `{{$json.body.email}}`  
   - Connect output of webhook to this node.  
   - Use VerifiEmail API credentials.

3. **Add If node**  
   - Name: `Email Validity Check`  
   - Condition: Check if `{{$json.valid}}` equals `true`  
   - Connect output of Email Validation node to this node.

4. **Add Google Sheets node**  
   - Name: `Log Client`  
   - Operation: Append  
   - Spreadsheet: Select or create a Google Sheet with columns: Name, Plan, Email, Company, Timestamp, Status  
   - Map columns with expressions from webhook data, e.g., Name = `={{ $('Client Onboarding Webhook').item.json.body.name }}`  
   - Set Status = "Onboarding Started"  
   - Authenticate with Google Sheets OAuth2.  
   - Connect the `true` output of Email Validity Check to this node.

5. **Add Code node**  
   - Name: `Assign Tier Logic`  
   - JavaScript code: Implement logic to assign `tier` and `priority` based on Plan field with cases for 'enterprise', 'pro', and default 'free'.  
   - Add `onboarding_id` as `ONB-<timestamp>`.  
   - Connect output of Log Client node here.

6. **Add Trello node**  
   - Name: `Create Trello Task Card`  
   - Operation: Create card  
   - Card name: `=Onboard {{ $json.Name }}- {{ $json.Plan }} Plan`  
   - List ID: Use your Trello list ID for onboarding tasks.  
   - Description: Include client details and checklist with next steps.  
   - Due date: `={{ $now.plus(3, 'days') }}`  
   - Authenticate with Trello API credentials.  
   - Connect Assign Tier Logic output.

7. **Add HTMLCSSToPDF node**  
   - Name: `Generate Welcome PDF`  
   - HTML content: Use a template with inline styles that dynamically inject client name, company, plan, priority, onboarding ID, and next steps.  
   - Authenticate with Htmlcsstopdf API.  
   - Connect from Trello node.

8. **Add Slack node**  
   - Name: `Send Slack Notification`  
   - Channel: Select your Slack channel ID.  
   - Message: Template with client and onboarding details.  
   - Authenticate with Slack OAuth2.  
   - Connect from Generate Welcome PDF.

9. **Add HTTP Request node**  
   - Name: `Download PDF File`  
   - Method: GET  
   - URL: `={{ $('Generate Welcome PDF').item.json.pdf_url }}`  
   - Response format: File (binary)  
   - Connect from Slack Notification.

10. **Add Gmail node**  
    - Name: `Send Welcome Email`  
    - To: `={{ $('Assign Tier Logic').item.json.Email }}`  
    - Subject: Welcome email including client plan.  
    - Message body: Personalized welcome message referencing client name, company, plan, onboarding ID, priority; mention attached PDF.  
    - Attach binary PDF downloaded in previous node.  
    - Authenticate with Gmail OAuth2.  
    - Connect from Download PDF File.

11. **Add Airtable node**  
    - Name: `Archive Data`  
    - Operation: Create record  
    - Base and Table: Select Airtable base and onboarding table.  
    - Fields: Map Name, Email, Company, Plan, Tier, Priority, Onboarding ID, Onboarding Date (current time), Status = "Completed".  
    - Authenticate with Airtable OAuth2.  
    - Connect from Send Welcome Email.

12. **Create Schedule Trigger node**  
    - Name: `Schedule Weekly Summary`  
    - Cron expression: `0 9 * * 1` (every Monday 9 AM)  
    - Purpose: Trigger weekly reporting workflow.

13. **Add Airtable node**  
    - Name: `Weekly Report - Data Collection`  
    - Operation: List records from onboarding table.  
    - Authenticate with Airtable OAuth2.  
    - Connect from Schedule Weekly Summary.

14. **Add Code node**  
    - Name: `Weekly Report - Data Processing`  
    - JavaScript code: Filter records from last 7 days; compute total clients, plan breakdown, tier breakdown, earliest onboarding date; format JSON report.  
    - Connect from Weekly Report - Data Collection.

15. **Add Gmail node**  
    - Name: `Send Weekly Report Email`  
    - To: `manager@company.com`  
    - Subject and message body: Use expressions to dynamically insert weekly report data.  
    - Authenticate with Gmail OAuth2.  
    - Connect from Weekly Report - Data Processing.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires configured credentials for Google Sheets, Trello, Slack, Gmail, Airtable, VerifiEmail, and Htmlcsstopdf APIs. | Credential configuration is essential for API access.                                          |
| The workflow drops invalid email submissions early to maintain data quality and reduce spam or fake accounts. | Email validation uses VerifiEmail API.                                                         |
| Personalized PDF generation uses inline HTML templates; update the logo URL and styling as per branding.   | Update logo URL: `https://yourcompany.com/logo.png`                                            |
| Slack notifications help internal teams stay informed of new onboarding activity in real time.             | Slack channel ID must be valid; unfurl media disabled for cleaner messages.                     |
| Weekly reporting runs every Monday at 9 AM IST; adjust timezone and schedule as needed.                     | Cron expression used: `0 9 * * 1`                                                              |
| For advanced customizations, modify the tier assignment logic in the Code node or the PDF HTML template.   | Tier assignment logic is in the `Assign Tier Logic` node.                                      |
| Workflow designed for B2B SaaS or agency onboarding scenarios, but adaptable to other client onboarding uses. | Consider adjusting fields and steps to fit different onboarding requirements.                   |

---

*Disclaimer:* The provided content is extracted exclusively from an n8n automated workflow. It adheres strictly to current content policies and contains no illegal or protected material. All handled data is legal and public.