Automated Employee Onboarding Document Generator with Google Drive, Gmail & Slack Integration

https://n8nworkflows.xyz/workflows/automated-employee-onboarding-document-generator-with-google-drive--gmail---slack-integration-10686


# Automated Employee Onboarding Document Generator with Google Drive, Gmail & Slack Integration

### 1. Workflow Overview

This workflow automates the generation and delivery of a comprehensive employee onboarding package. It is designed for HR teams integrating with HRIS (Human Resource Information Systems) or ATS (Applicant Tracking Systems) to provide new hires with personalized onboarding documents. The workflow includes the following logical blocks:

- **1.1 Input Reception & Validation:** Receives employee data via webhook, validates required fields, enriches data with company defaults and role-specific information.
- **1.2 Document Generation:** Creates four personalized HTML documents (Welcome Letter, Benefits Guide, IT Setup Instructions, Emergency Contact & Direct Deposit Forms) based on enriched employee data.
- **1.3 PDF Processing:** Organizes the generated HTML documents into a numbered array, converts them to PDFs, and archives them to Google Drive.
- **1.4 Email Packaging & Delivery:** Aggregates PDFs, prepares a personalized email with attachments, and sends it to the new employee via Gmail.
- **1.5 Tracking & HR Notification:** Logs onboarding package delivery details in the HR system and sends a Slack notification to HR with employee and package information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  Captures new employee data from external HRIS or ATS via webhook, then validates and enriches this data with additional company and role-specific information.

- **Nodes Involved:**  
  - Webhook Trigger2  
  - Validate & Enrich Data2  
  - Sticky Note22 (workflow overview)  
  - Sticky Note23 (input & processing explanation)

- **Node Details:**

  - **Webhook Trigger2**  
    - *Type:* Webhook  
    - *Role:* Entry point receiving POST requests at `/onboard-employee` path.  
    - *Configuration:* HTTP POST method, webhook named "employee-onboarding".  
    - *Inputs:* Incoming HTTP requests with employee JSON data.  
    - *Outputs:* Passes data to validation node.  
    - *Failures:* Invalid requests, missing body, or network issues.  
    - *Version:* 1

  - **Validate & Enrich Data2**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Ensures required fields are present and valid; enriches data with defaults, formats, and role-specific flags.  
    - *Key Logic:*  
      - Validates presence of fields: firstName, lastName, email, jobTitle, department, startDate.  
      - Validates email format and start date correctness.  
      - Generates employeeId if missing (format EMP-year-random).  
      - Adds formatted dates, full name, employment type, location, manager and HR contact info defaults.  
      - Determines role category flags (IT equipment required, management training, sales tools).  
      - Adds department-specific office info (floor, dress code, parking).  
      - Sets company info defaults.  
      - Creates documentPackId and timestamp for tracking.  
    - *Inputs:* JSON employee data from webhook.  
    - *Outputs:* Enriched employee data for document generation.  
    - *Failures:* Throws descriptive errors for missing fields, invalid email, or date.  
    - *Version:* 2

#### 2.2 Document Generation

- **Overview:**  
  Generates four separate, fully styled HTML documents tailored to the employee’s role and department: Welcome Letter, Benefits Guide, IT Setup Instructions, and Emergency Contact/Direct Deposit Forms.

- **Nodes Involved:**  
  - Generate Welcome Letter2  
  - Generate Benefits Guide2  
  - Generate IT Setup Guide2  
  - Generate Forms2  
  - Sticky Note24 (PDF processing note)  
  - Sticky Note27 (document generation explanation)

- **Node Details:**

  - **Generate Welcome Letter2**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Creates a personalized welcome letter HTML with first-day logistics, manager info, benefits eligibility, and company branding.  
    - *Key Variables:* Uses enriched employee data such as fullName, startDateFormatted, jobTitle, department, manager info, dress code, and benefits eligibility flags.  
    - *Outputs:* JSON with `welcomeLetterHtml` property containing HTML string.  
    - *Failures:* Rare, mostly syntax or data injection errors.  
    - *Version:* 2

  - **Generate Benefits Guide2**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Produces detailed benefits enrollment guide HTML including health, dental, vision, retirement, PTO, insurance, and wellness programs. Enrollment deadlines and company contributions are included.  
    - *Key Variables:* employeeId, fullName, companyName, benefitsStartDate, hrContact info.  
    - *Outputs:* JSON with `benefitsGuideHtml`.  
    - *Failures:* Similar to welcome letter node, dependent on valid input data.  
    - *Version:* 2

  - **Generate IT Setup Guide2**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Creates IT setup instructions HTML with role-specific equipment lists, login credentials, step-by-step first-day IT setup, system links, support contacts, and IT policies.  
    - *Key Variables:* requiresITEquipment, requiresSalesTools, officeFloor, companyWebsite, firstName, lastName.  
    - *Outputs:* JSON with `itSetupHtml`.  
    - *Failures:* Dependent on proper role flag detection and input data.  
    - *Version:* 2

  - **Generate Forms2**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Generates emergency contact and direct deposit authorization forms in HTML, with sections for primary and secondary contacts, bank info, authorization signature, and checklist.  
    - *Key Variables:* employeeId, fullName, companyName, hrContactEmail, officeFloor.  
    - *Outputs:* JSON with `formsHtml`.  
    - *Failures:* Minimal; syntax errors possible if input data malformed.  
    - *Version:* 2

#### 2.3 PDF Processing

- **Overview:**  
  Collects all generated HTML documents, organizes them with numbered filenames, converts each to professional PDF files, and saves them to Google Drive.

- **Nodes Involved:**  
  - Prepare Document Array2  
  - Set HTML Field2  
  - Convert to PDF2  
  - Save to Drive2  
  - Aggregate All Documents2  
  - Sticky Note24 (PDF Processing explanation)  
  - Sticky Note25 (Storage & Delivery explanation)

- **Node Details:**

  - **Prepare Document Array2**  
    - *Type:* Code node  
    - *Role:* Aggregates the four HTML docs into an ordered array of objects with document names and filenames formatted as `order_name_employeeId.pdf`.  
    - *Outputs:* An array of JSON objects, each representing one document for conversion.  
    - *Failures:* Requires all four HTML fields to exist.  
    - *Version:* 2

  - **Set HTML Field2**  
    - *Type:* Set node  
    - *Role:* Assigns the document HTML string to the `html` field, preparing input for PDF conversion node.  
    - *Inputs:* Document array items.  
    - *Outputs:* Passes HTML content for PDF conversion.  
    - *Version:* 3.4

  - **Convert to PDF2**  
    - *Type:* n8n HTML/CSS to PDF node  
    - *Role:* Converts HTML content to PDF files using external API (configured with HTMLCSSToPDF API key).  
    - *Inputs:* HTML content from Set HTML Field2.  
    - *Outputs:* Binary PDF data.  
    - *Failures:* Possible API errors, rate limits, or invalid HTML.  
    - *Version:* 1

  - **Save to Drive2**  
    - *Type:* Google Drive node  
    - *Role:* Saves each generated PDF to Google Drive under a specified folder (default "root" or configured folder).  
    - *Configuration:* Uses Google Drive OAuth2 credentials; filename set dynamically from input JSON.  
    - *Failures:* Authentication errors, insufficient permissions, or quota issues.  
    - *Version:* 3

  - **Aggregate All Documents2**  
    - *Type:* Aggregate node  
    - *Role:* Aggregates all saved PDF documents into a single array for email attachment preparation.  
    - *Version:* 1

#### 2.4 Email Packaging & Delivery

- **Overview:**  
  Prepares a personalized welcome email with all onboarding PDFs attached, then sends the email to the new employee.

- **Nodes Involved:**  
  - Prepare Email Package2  
  - Send to Employee2  
  - Sticky Note25 (Storage & Delivery note)

- **Node Details:**

  - **Prepare Email Package2**  
    - *Type:* Code node  
    - *Role:* Compiles all PDF attachments into an email-ready format, constructs a personalized email body with first-day information and action items checklist.  
    - *Inputs:* Aggregated PDF binary data and enriched employee data.  
    - *Outputs:* JSON containing email fields: `emailTo`, `emailSubject`, `emailBody`, `attachments`.  
    - *Failures:* Throws error if no documents present.  
    - *Version:* 2

  - **Send to Employee2**  
    - *Type:* Gmail node  
    - *Role:* Sends the prepared email with attachments via authenticated Gmail account.  
    - *Configuration:* Uses Gmail OAuth2 credentials; email fields set dynamically.  
    - *Failures:* Auth errors, sending limits, attachment size limits.  
    - *Version:* 2.1

#### 2.5 Tracking & HR Notification

- **Overview:**  
  Logs the onboarding package delivery in the internal HR system for tracking and compliance, then sends a Slack notification to the HR team confirming completion.

- **Nodes Involved:**  
  - Log in HR System2  
  - Notify HR Team2  
  - Sticky Note28 (Tracking & Alerts explanation)

- **Node Details:**

  - **Log in HR System2**  
    - *Type:* Code node  
    - *Role:* Creates a structured log object with employee details, documents sent, signature requirements, status flags, and an activity log. Intended for HR system API or internal database integration (though no HTTP request is shown).  
    - *Outputs:* JSON with `hrSystemLog` details and timestamps.  
    - *Failures:* Depends on subsequent integration to persist data.  
    - *Version:* 2

  - **Notify HR Team2**  
    - *Type:* HTTP Request node  
    - *Role:* Sends a formatted Slack message via webhook URL to notify HR of package delivery, including employee name, ID, position, department, documents sent, and manager info.  
    - *Configuration:* Slack webhook URL must be set; message uses Slack Block Kit JSON with markdown.  
    - *Failures:* Slack webhook invalid or network errors.  
    - *Version:* 4.1

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                            | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                        |
|-------------------------|----------------------------|--------------------------------------------|-------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------|
| Sticky Note22           | Sticky Note                | Workflow overview and setup instructions   |                         |                          | ## How it works ... Required data: firstName, lastName, email, jobTitle, department, startDate                      |
| Webhook Trigger2        | Webhook                   | Receives employee onboarding data          |                         | Validate & Enrich Data2   | ## Input & Smart Processing ...                                                                                     |
| Sticky Note23           | Sticky Note                | Input reception and validation explanation |                         |                          | ## Input & Smart Processing ...                                                                                     |
| Validate & Enrich Data2 | Code                      | Validates and enriches employee data       | Webhook Trigger2         | Generate Welcome Letter2  |                                                                                                                    |
| Generate Welcome Letter2| Code                      | Creates Welcome Letter HTML document        | Validate & Enrich Data2  | Generate IT Setup Guide2  | ## Document Generation ...                                                                                         |
| Generate IT Setup Guide2| Code                      | Creates IT Setup Instructions HTML          | Generate Welcome Letter2 | Generate Benefits Guide2  |                                                                                                                    |
| Generate Benefits Guide2| Code                      | Creates Benefits Guide HTML                  | Generate IT Setup Guide2 | Generate Forms2           |                                                                                                                    |
| Generate Forms2         | Code                      | Creates Emergency Contact & Direct Deposit Forms HTML | Generate Benefits Guide2 | Prepare Document Array2  |                                                                                                                    |
| Sticky Note24           | Sticky Note                | PDF processing explanation                   |                         |                          | ## PDF Processing ...                                                                                               |
| Prepare Document Array2 | Code                      | Aggregates HTML docs into array for PDF     | Generate Forms2          | Set HTML Field2           |                                                                                                                    |
| Set HTML Field2         | Set                       | Sets HTML field for PDF conversion           | Prepare Document Array2  | Convert to PDF2           |                                                                                                                    |
| Convert to PDF2         | HTMLCSSToPDF              | Converts HTML to PDF                         | Set HTML Field2          | Save to Drive2            |                                                                                                                    |
| Save to Drive2          | Google Drive              | Saves generated PDFs to Google Drive        | Convert to PDF2          | Aggregate All Documents2  | ## Storage & Delivery ...                                                                                           |
| Aggregate All Documents2| Aggregate                 | Aggregates saved PDFs for email              | Save to Drive2           | Prepare Email Package2    |                                                                                                                    |
| Prepare Email Package2  | Code                      | Prepares email with attachments and body    | Aggregate All Documents2 | Send to Employee2         |                                                                                                                    |
| Send to Employee2       | Gmail                     | Sends email with onboarding package          | Prepare Email Package2   | Log in HR System2         |                                                                                                                    |
| Sticky Note25           | Sticky Note                | Storage and delivery explanation             |                         |                          | ## Storage & Delivery ...                                                                                           |
| Log in HR System2       | Code                      | Logs package delivery details for HR system | Send to Employee2        | Notify HR Team2           |                                                                                                                    |
| Notify HR Team2         | HTTP Request              | Sends Slack notification to HR team          | Log in HR System2        |                          | ## Tracking & Alerts ...                                                                                            |
| Sticky Note27           | Sticky Note                | Document generation explanation               |                         |                          | ## Document Generation ...                                                                                         |
| Sticky Note28           | Sticky Note                | Tracking and alerts explanation               |                         |                          | ## Tracking & Alerts ...                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node ("Webhook Trigger2"):**  
   - Set HTTP Method to POST.  
   - Set Path to `onboard-employee`.  
   - This node receives new employee JSON data from HRIS/ATS.

2. **Create Code Node ("Validate & Enrich Data2"):**  
   - JavaScript code to validate required fields (`firstName`, `lastName`, `email`, `jobTitle`, `department`, `startDate`).  
   - Validate email format and startDate validity.  
   - Enrich with defaults: generate employee ID, format dates, add manager/HR contacts, role-specific flags (IT equipment, management training, sales tools), department info, company info, and tracking IDs.  
   - Connect output of webhook node to this node.

3. **Create Code Node ("Generate Welcome Letter2"):**  
   - Use JavaScript to generate HTML content for a welcome letter using enriched employee data.  
   - Include first-day logistics, manager info, benefits eligibility, and company branding.  
   - Connect from "Validate & Enrich Data2" node.

4. **Create Code Node ("Generate IT Setup Guide2"):**  
   - Generate HTML for IT setup instructions, with equipment lists depending on role flags, login credentials, setup steps, policies, and contact info.  
   - Connect from "Generate Welcome Letter2".

5. **Create Code Node ("Generate Benefits Guide2"):**  
   - Generate HTML for benefits enrollment guide with enrollment deadlines, plan details, and instructions.  
   - Connect from "Generate IT Setup Guide2".

6. **Create Code Node ("Generate Forms2"):**  
   - Generate HTML for emergency contact and direct deposit forms, including signature and checklist sections.  
   - Connect from "Generate Benefits Guide2".

7. **Create Code Node ("Prepare Document Array2"):**  
   - Aggregate all generated HTML documents into an ordered array with filenames formatted as `<order>_<documentName>_<employeeId>.pdf`.  
   - Connect from "Generate Forms2".

8. **Create Set Node ("Set HTML Field2"):**  
   - Assign the `documentHtml` field to `html` for PDF conversion.  
   - Connect from "Prepare Document Array2".

9. **Create HTMLCSSToPDF Node ("Convert to PDF2"):**  
   - Use the HTMLCSSToPDF node to convert HTML to PDF files.  
   - Configure API key from htmlcsstoimage.com (paid service, ~1-5¢ per doc).  
   - Connect from "Set HTML Field2".

10. **Create Google Drive Node ("Save to Drive2"):**  
    - Configure Google Drive OAuth2 credentials.  
    - Set to upload file with name `={{ $json.fileName }}`.  
    - Set folder ID to employee folder or root.  
    - Connect from "Convert to PDF2".

11. **Create Aggregate Node ("Aggregate All Documents2"):**  
    - Aggregate all saved PDF files to prepare for email attachment.  
    - Connect from "Save to Drive2".

12. **Create Code Node ("Prepare Email Package2"):**  
    - Create an email body summarizing onboarding package, list all PDF attachments with names and binary data.  
    - Set email recipient (`emailTo`), subject, and body dynamically.  
    - Connect from "Aggregate All Documents2".

13. **Create Gmail Node ("Send to Employee2"):**  
    - Configure Gmail OAuth2 credentials.  
    - Use dynamic fields for To, Subject, Body, and Attachments.  
    - Connect from "Prepare Email Package2".

14. **Create Code Node ("Log in HR System2"):**  
    - Format detailed log entry of package delivery, including employee info, documents sent, signature requirements, and activity timestamps.  
    - Connect from "Send to Employee2".

15. **Create HTTP Request Node ("Notify HR Team2"):**  
    - Configure Slack webhook URL (must be obtained from Slack App management).  
    - Prepare JSON payload with Slack Block Kit formatting including employee and package info.  
    - Connect from "Log in HR System2".

16. **Add Sticky Notes:**  
    - Add explanatory sticky notes as shown in original workflow for documentation and setup instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow requires HTML to PDF API key from https://htmlcsstoimage.com (cost approx. 1-5¢ per document conversion).    | PDF generation node (Convert to PDF2)                                                           |
| Slack webhook URL must be created and configured in Slack workspace to enable HR team notifications.                  | Notify HR Team2 node HTTP request configuration                                                 |
| Google Drive API OAuth2 credentials must have permissions to create folders and upload files for document archival.  | Save to Drive2 node                                                                             |
| Gmail OAuth2 credentials require scopes to send emails with attachments on behalf of HR or system account.            | Send to Employee2 node                                                                          |
| Sample employee data must include at least firstName, lastName, email, jobTitle, department, and startDate fields.   | Input data requirements in Sticky Note22 and Validate & Enrich Data2 node                       |
| The onboarding package includes 4 PDF documents: Welcome Letter, Benefits Guide, IT Setup Instructions, and Forms.   | Sticky Note22, Sticky Note27, Sticky Note24                                                    |
| HR system logging node outputs structured JSON for future integration with external HR APIs or databases.            | Log in HR System2 node                                                                         |
| Slack notification includes employee info, documents sent, and manager contact confirmation.                         | Notify HR Team2 node Slack message payload                                                     |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.