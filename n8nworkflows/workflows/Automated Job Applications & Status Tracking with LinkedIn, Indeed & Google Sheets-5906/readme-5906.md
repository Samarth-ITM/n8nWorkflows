Automated Job Applications & Status Tracking with LinkedIn, Indeed & Google Sheets

https://n8nworkflows.xyz/workflows/automated-job-applications---status-tracking-with-linkedin--indeed---google-sheets-5906


# Automated Job Applications & Status Tracking with LinkedIn, Indeed & Google Sheets

---

### 1. Workflow Overview

This workflow automates job applications from a Google Sheets list and tracks the status of those applications over time. It is designed for job seekers, recruiters, and career coaches who want to streamline bulk job applications and monitor outcomes efficiently.

The workflow consists of two main logical blocks:

- **1.1 Job Application Automation (Daily)**  
  Reads job listings marked as "Not Applied" from a Google Sheet, generates personalized cover letters, applies to jobs via LinkedIn or Indeed APIs, updates the sheet with application results, and sends email notifications.

- **1.2 Application Status Tracking (Every 2 Days)**  
  Reads all applied jobs from the sheet, checks for status updates (mocked currently), updates the sheet if statuses have changed, and notifies the user by email.

These blocks operate on scheduled triggers and share a central configuration node to maintain key parameters like spreadsheet ID, resume URL, and email addresses.

---

### 2. Block-by-Block Analysis

#### 2.1 Job Application Automation (Daily)

**Overview:**  
Triggered every weekday morning, this block reads pending job applications from a Google Sheet, processes each job one by one, applies via the appropriate platform API (LinkedIn or Indeed), updates the application status in the sheet, and sends notification emails.

**Nodes Involved:**  
- 🕘 Daily Application Trigger  
- ⚙️ Configuration  
- 📖 Read Jobs Sheet  
- 🎯 Filter Pending Applications  
- 🔄 Process Jobs One by One  
- 📝 Prepare Application Data  
- 🔀 Route by Platform  
- 💼 Apply via LinkedIn  
- 🔍 Apply via Indeed  
- 📊 Process Application Result  
- 📝 Update Job Status  
- 📧 Send Application Notification  

---

**Node Details:**  

- **🕘 Daily Application Trigger**  
  - Type: Cron Trigger  
  - Role: Initiates the daily application process Monday to Friday at 9:00 AM.  
  - Config: Cron expression "0 9 * * 1-5".  
  - Failure: Cron misconfiguration unlikely; depends on n8n scheduler stability.  

- **⚙️ Configuration**  
  - Type: Set Node  
  - Role: Holds constants such as Google Sheet ID, resume URL, cover letter template, and user email.  
  - Config: Variables include `spreadsheetId`, `resumeUrl`, `coverLetterTemplate` (with placeholders), and `userEmail`.  
  - Failure: Missing or incorrect IDs/URLs will cause downstream errors.  

- **📖 Read Jobs Sheet**  
  - Type: Google Sheets (Read)  
  - Role: Fetches job listings from the configured Google Sheet ("Jobs" sheet).  
  - Config: Reads columns A:J, maps header row. Uses OAuth2 for Google Sheets authentication.  
  - Input: Takes `spreadsheetId` from Configuration node.  
  - Failure: Authentication errors, sheet not found, or permission issues possible.  

- **🎯 Filter Pending Applications**  
  - Type: Filter  
  - Role: Filters jobs where `Status` equals "Not Applied" and `Job_URL` is not empty.  
  - Config: Combined conditions with strict string matching.  
  - Failure: Case sensitivity or missing fields may cause incorrect filtering.  

- **🔄 Process Jobs One by One**  
  - Type: SplitInBatches  
  - Role: Processes one job record at a time to handle API requests sequentially.  
  - Config: Batch size = 1.  
  - Failure: Large datasets may cause long runtime; batching ensures API limits are respected.  

- **📝 Prepare Application Data**  
  - Type: Set  
  - Role: Prepares variables for applying: detects platform from URL, personalizes cover letter by replacing placeholders, sets application date, and adds resume URL.  
  - Key Expressions:  
    - Platform detection based on substrings in `Job_URL` (linkedin.com, indeed.com, else generic).  
    - Cover letter template replaces `{{position}}`, `{{company}}`, and `{{skills}}` placeholders.  
    - Date formatted as YYYY-MM-DD.  
  - Failure: If `Job_URL` is malformed or missing expected substrings, platform detection may fail. Expression errors possible if placeholders are missing.  

- **🔀 Route by Platform**  
  - Type: Switch  
  - Role: Routes application requests based on platform: LinkedIn or Indeed. Defaults to generic but only LinkedIn and Indeed paths are implemented.  
  - Conditions: Checks if `platform == linkedin`, else Indeed.  
  - Failure: Unhandled platforms fall through without processing.  

- **💼 Apply via LinkedIn**  
  - Type: HTTP Request  
  - Role: Sends POST request to LinkedIn jobs application API with job ID, cover letter, and resume URL.  
  - Config: OAuth2 credential type "linkedInOAuth2Api". Retries enabled (3 attempts), 30s timeout.  
  - Failure: Auth errors, request timeouts, API rate limits, invalid job IDs.  
  
- **🔍 Apply via Indeed**  
  - Type: HTTP Request  
  - Role: Sends POST request to Indeed applications API with job key and cover letter message.  
  - Config: OAuth2 credential type "indeedApi". Retries and timeout same as LinkedIn node.  
  - Failure: Similar to LinkedIn node failures.  

- **📊 Process Application Result**  
  - Type: Set  
  - Role: Processes API response to set application status ("Applied" if success, else "Failed"), extracts or generates application ID, and creates status notes.  
  - Key Expressions:  
    - Status based on HTTP status code range 200-299.  
    - Application ID extracted from response body or auto-generated with timestamp.  
  - Failure: Missing response fields, malformed JSON, or unexpected status codes can cause incorrect status.  

- **📝 Update Job Status**  
  - Type: Google Sheets (Update)  
  - Role: Updates job row with new status, application date, last checked date, application ID, and notes.  
  - Config: Updates columns D:H of the job’s row in the "Jobs" sheet.  
  - Failure: Incorrect row index or sheet permissions cause update failure.  

- **📧 Send Application Notification**  
  - Type: Gmail  
  - Role: Sends an email to user summarizing the application attempt result with job details, status, and notes.  
  - Config: Email address from configuration node; HTML formatted message with dynamic content.  
  - Failure: Gmail API auth issues or invalid email address.  

---

#### 2.2 Application Status Tracking (Every 2 Days)

**Overview:**  
Runs every two days to check the current status of jobs already applied to. It filters applied jobs, simulates status updates (mock), detects changes, updates the sheet accordingly, and notifies the user by email.

**Nodes Involved:**  
- 📊 Status Tracking Info (Sticky Note)  
- 🕐 Status Check Trigger  
- 📖 Read Applied Jobs  
- 🎯 Filter Applied Jobs  
- 🔄 Check Status One by One  
- 🔍 Mock Status Check  
- 🔄 Check if Status Changed  
- 📝 Update Changed Status  
- 📧 Send Status Update  

---

**Node Details:**  

- **📊 Status Tracking Info**  
  - Type: Sticky Note  
  - Role: Describes the purpose and notes of the status tracking sub-process.  
  - Content: Reminder to replace mock status checks with real API calls.  

- **🕐 Status Check Trigger**  
  - Type: Cron Trigger  
  - Role: Triggers status tracking every two days at 10:00 AM.  
  - Config: Cron "0 10 */2 * *".  

- **📖 Read Applied Jobs**  
  - Type: Google Sheets (Read)  
  - Role: Reads all job entries from the "Jobs" sheet.  
  - Config: Same as the "Read Jobs Sheet" node.  

- **🎯 Filter Applied Jobs**  
  - Type: Filter  
  - Role: Filters rows where `Status` equals "Applied".  
  - Failure: Case sensitivity or missing fields could cause filtering errors.  

- **🔄 Check Status One by One**  
  - Type: SplitInBatches  
  - Role: Processes each applied job sequentially.  
  - Config: Batch size = 1.  

- **🔍 Mock Status Check**  
  - Type: Set  
  - Role: Simulates status updates by randomly selecting from four statuses: "Applied", "Under Review", "Interview Scheduled", "Rejected". Also sets current date and notes.  
  - Failure: This is a placeholder; real API integration needed for production.  

- **🔄 Check if Status Changed**  
  - Type: If  
  - Role: Compares existing `Status` with `newStatus` from mock check; routes only if changed.  
  - Failure: Expression errors if fields missing.  

- **📝 Update Changed Status**  
  - Type: Google Sheets (Update)  
  - Role: Updates status, applied date, and last checked date for changed jobs in the sheet.  
  - Config: Updates columns D:F of the job’s row.  

- **📧 Send Status Update**  
  - Type: Gmail  
  - Role: Sends email notification about the status change with job details.  
  - Config: Email and content similar to application notification node.  

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                              | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                                                      |
|-----------------------------|------------------------|----------------------------------------------|-----------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| 📋 Template Overview        | Sticky Note            | Workflow purpose and setup instructions      | —                           | —                                | Explains workflow purpose, requirements, and setup instructions.                                                |
| 📊 Sheet Structure          | Sticky Note            | Describes required Google Sheet columns      | —                           | —                                | Details expected spreadsheet columns for job tracking.                                                         |
| ⚙️ Setup Notes             | Sticky Note            | Configuration and security notes              | —                           | —                                | Advises on replacing placeholders and securing credentials.                                                    |
| 🕘 Daily Application Trigger | Cron                   | Starts daily job application process          | —                           | ⚙️ Configuration                 |                                                                                                                 |
| ⚙️ Configuration            | Set                    | Holds constants like spreadsheet ID, email   | 🕘 Daily Application Trigger | 📖 Read Jobs Sheet               |                                                                                                                 |
| 📖 Read Jobs Sheet          | Google Sheets (Read)   | Reads jobs from sheet                          | ⚙️ Configuration             | 🎯 Filter Pending Applications   |                                                                                                                 |
| 🎯 Filter Pending Applications | Filter                | Filters jobs not yet applied                   | 📖 Read Jobs Sheet           | 🔄 Process Jobs One by One       |                                                                                                                 |
| 🔄 Process Jobs One by One  | SplitInBatches         | Processes jobs sequentially                    | 🎯 Filter Pending Applications | 📝 Prepare Application Data     |                                                                                                                 |
| 📝 Prepare Application Data | Set                    | Prepares data including personalized cover letter | 🔄 Process Jobs One by One  | 🔀 Route by Platform             |                                                                                                                 |
| 🔀 Route by Platform        | Switch                 | Routes to platform-specific apply nodes       | 📝 Prepare Application Data  | 💼 Apply via LinkedIn, 🔍 Apply via Indeed |                                                                                                                 |
| 💼 Apply via LinkedIn       | HTTP Request           | Sends application request to LinkedIn API     | 🔀 Route by Platform         | 📊 Process Application Result   |                                                                                                                 |
| 🔍 Apply via Indeed         | HTTP Request           | Sends application request to Indeed API       | 🔀 Route by Platform         | 📊 Process Application Result   |                                                                                                                 |
| 📊 Process Application Result | Set                  | Processes API response to determine application status | 💼 Apply via LinkedIn, 🔍 Apply via Indeed | 📝 Update Job Status            |                                                                                                                 |
| 📝 Update Job Status        | Google Sheets (Update) | Updates job status and application details    | 📊 Process Application Result | 📧 Send Application Notification |                                                                                                                 |
| 📧 Send Application Notification | Gmail               | Sends email about application result          | 📝 Update Job Status         | —                                |                                                                                                                 |
| 📊 Status Tracking Info     | Sticky Note            | Describes status tracking process              | —                           | —                                | Notes to replace mock status checks with actual API calls.                                                     |
| 🕐 Status Check Trigger     | Cron                   | Triggers status tracking every 2 days          | —                           | 📖 Read Applied Jobs             |                                                                                                                 |
| 📖 Read Applied Jobs        | Google Sheets (Read)   | Reads all applied jobs                          | 🕐 Status Check Trigger      | 🎯 Filter Applied Jobs           |                                                                                                                 |
| 🎯 Filter Applied Jobs      | Filter                 | Filters jobs with status "Applied"             | 📖 Read Applied Jobs         | 🔄 Check Status One by One       |                                                                                                                 |
| 🔄 Check Status One by One  | SplitInBatches         | Processes applied jobs sequentially            | 🎯 Filter Applied Jobs       | 🔍 Mock Status Check             |                                                                                                                 |
| 🔍 Mock Status Check        | Set                    | Simulates status update                         | 🔄 Check Status One by One   | 🔄 Check if Status Changed       |                                                                                                                 |
| 🔄 Check if Status Changed  | If                     | Checks if status has changed                     | 🔍 Mock Status Check         | 📝 Update Changed Status         |                                                                                                                 |
| 📝 Update Changed Status    | Google Sheets (Update) | Updates changed status in sheet                 | 🔄 Check if Status Changed   | 📧 Send Status Update            |                                                                                                                 |
| 📧 Send Status Update       | Gmail                  | Sends email notification about status change   | 📝 Update Changed Status     | —                                |                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet**  
   - Name sheet "Jobs".  
   - Include columns: Job_ID, Company, Position, Status, Applied_Date, Last_Checked, Application_ID, Notes, Job_URL, Priority.  
   - Populate with sample data for testing.

2. **Set up Credentials**  
   - Configure Google OAuth2 credentials for Google Sheets access.  
   - Configure Gmail OAuth2 credentials for sending email.  
   - Set up OAuth2 credentials for LinkedIn API (`linkedInOAuth2Api`) and Indeed API (`indeedApi`).

3. **Create Sticky Notes for Documentation**  
   - Add three sticky notes: Template Overview, Sheet Structure, Setup Notes.  
   - Content as specified in the overview for user guidance.

4. **Daily Application Automation Block**

   4.1 **Add Cron Node "🕘 Daily Application Trigger"**  
       - Schedule: 9:00 AM Monday to Friday (Cron: `0 9 * * 1-5`).

   4.2 **Add Set Node "⚙️ Configuration"**  
       - Define variables:  
         - `spreadsheetId`: Your Google Sheet ID  
         - `resumeUrl`: Public URL to your resume (Google Drive recommended)  
         - `coverLetterTemplate`: Letter with placeholders `{{position}}`, `{{company}}`, `{{skills}}`  
         - `userEmail`: Your email address for notifications

   4.3 **Add Google Sheets Node "📖 Read Jobs Sheet"**  
       - Action: Read rows from sheet "Jobs"  
       - Range: A:J  
       - Document ID: Reference `spreadsheetId` from Configuration node  
       - Authentication: Use Google OAuth2 credentials

   4.4 **Add Filter Node "🎯 Filter Pending Applications"**  
       - Conditions:  
         - `Status` equals "Not Applied"  
         - `Job_URL` is not empty

   4.5 **Add SplitInBatches Node "🔄 Process Jobs One by One"**  
       - Batch size: 1

   4.6 **Add Set Node "📝 Prepare Application Data"**  
       - Assign:  
         - `platform`: Determine based on if `Job_URL` contains "linkedin.com" or "indeed.com"  
         - `personalizedCoverLetter`: Replace placeholders in `coverLetterTemplate` with job data and fixed "relevant technical skills" string  
         - `applicationDate`: Current date formatted `yyyy-MM-dd`  
         - `resumeUrl`: From configuration

   4.7 **Add Switch Node "🔀 Route by Platform"**  
       - Condition: If `platform` equals "linkedin", route to LinkedIn apply node, else to Indeed apply node

   4.8 **Add HTTP Request Node "💼 Apply via LinkedIn"**  
       - Method: POST  
       - URL: `https://api.linkedin.com/v2/jobs/applications`  
       - Body: JSON including `jobId`, `coverLetter`, `resumeUrl` from input data  
       - Headers: Content-Type: application/json  
       - Authentication: OAuth2 credential `linkedInOAuth2Api`  
       - Retry enabled (max 3), timeout 30s

   4.9 **Add HTTP Request Node "🔍 Apply via Indeed"**  
       - Method: POST  
       - URL: `https://api.indeed.com/ads/applications`  
       - Body: JSON including `jobkey`, `message` (cover letter)  
       - Headers: Content-Type: application/json  
       - Authentication: OAuth2 credential `indeedApi`  
       - Retry enabled (max 3), timeout 30s

   4.10 **Add Set Node "📊 Process Application Result"**  
        - Assign:  
          - `applicationStatus`: "Applied" if HTTP status 200-299, else "Failed"  
          - `applicationId`: Extract from response or generate with timestamp  
          - `statusNotes`: Success or failure message  
          - `originalJobData`: Store job data for downstream use

   4.11 **Add Google Sheets Node "📝 Update Job Status"**  
        - Update range D:H in job’s row index with new status, application date, last checked date, application ID, notes  
        - Document ID from configuration  
        - Authentication: Google OAuth2

   4.12 **Add Gmail Node "📧 Send Application Notification"**  
        - Send email to `userEmail` with application summary in HTML format  
        - Subject includes company and position

   4.13 **Connect nodes sequentially as per flow:**  
        Daily Trigger → Configuration → Read Jobs → Filter Pending → SplitInBatches → Prepare Data → Route → Platform apply HTTP Request → Process Result → Update Status → Send Notification

5. **Application Status Tracking Block**

   5.1 **Add Sticky Note "📊 Status Tracking Info"**  
       - Describe tracking process and note placeholder mock status checks.

   5.2 **Add Cron Node "🕐 Status Check Trigger"**  
       - Schedule: Every 2 days at 10:00 AM (Cron: `0 10 */2 * *`)

   5.3 **Add Google Sheets Node "📖 Read Applied Jobs"**  
       - Same config as reading jobs, from configuration node

   5.4 **Add Filter Node "🎯 Filter Applied Jobs"**  
       - Filter where `Status` equals "Applied"

   5.5 **Add SplitInBatches Node "🔄 Check Status One by One"**  
       - Batch size 1

   5.6 **Add Set Node "🔍 Mock Status Check"**  
       - Randomly assign `newStatus` from ["Applied", "Under Review", "Interview Scheduled", "Rejected"]  
       - Assign current date as `checkDate`  
       - Add notes "Status checked automatically via API"

   5.7 **Add If Node "🔄 Check if Status Changed"**  
       - Condition: `Status` != `newStatus`

   5.8 **Add Google Sheets Node "📝 Update Changed Status"**  
       - Update columns D:F with new status, applied date, check date  
       - Authentication: Google OAuth2

   5.9 **Add Gmail Node "📧 Send Status Update"**  
       - Send email to user about status change with details

   5.10 **Connect nodes:**  
         Status Trigger → Read Applied Jobs → Filter Applied Jobs → SplitInBatches → Mock Status Check → If Status Changed → Update Changed Status → Send Status Update

6. **Testing and Validation**  
   - Test with a few sample job entries before scaling.  
   - Replace mock API calls with real job platform APIs as needed.  
   - Monitor logs for errors, especially authentication and API rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Replace mock HTTP requests with actual job platform APIs for real-world automation.                           | Template Overview sticky note                                                                                   |
| Never hardcode API keys; use n8n’s credential management for all authentication.                              | Setup Notes sticky note                                                                                          |
| Google Sheets columns must be correctly named and populated as per the Sheet Structure sticky note.           | Sheet Structure sticky note                                                                                      |
| For best results, test on 1-2 jobs before enabling full automation.                                           | Template Overview sticky note                                                                                   |
| Status tracking currently uses random status simulation; integrate actual job platform status APIs for accuracy. | Status Tracking Info sticky note                                                                                 |
| For OAuth credential setup, refer to n8n documentation for Google Sheets, Gmail, LinkedIn, and Indeed APIs.   | n8n official docs (https://docs.n8n.io/credentials/)                                                             |

---

**Disclaimer:** The provided description and workflow stem exclusively from an automated n8n workflow. All data handled is legal and public, respecting content policies. No illegal or protected content is processed.

---