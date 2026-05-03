Send Automated Recruitment Rejection Email at End-of-Day (Google Sheets | Gmail)

https://n8nworkflows.xyz/workflows/send-automated-recruitment-rejection-email-at-end-of-day--google-sheets---gmail--7766


# Send Automated Recruitment Rejection Email at End-of-Day (Google Sheets | Gmail)

### 1. Workflow Overview

This workflow automates sending recruitment rejection emails to candidates listed in a Google Sheets spreadsheet. It is designed to run once daily at 6 PM (18:00) and processes candidates marked as "Rejected" (or other configured statuses) who have not yet been sent a rejection email. The workflow supports dry-run mode for testing, respects weekend sending policies, and updates the spreadsheet to log emails sent.

**Key use cases:**
- Automate end-of-day bulk sending of personalized rejection emails to candidates.
- Ensure no duplicate emails are sent.
- Customize email content using templates with candidate and company data.
- Rate-limit email sending to avoid hitting provider limits.
- Skip sending on weekends if configured.

**Logical blocks:**

- **1.1 Schedule and Configuration Setup:** Trigger workflow daily, load config values.
- **1.2 Weekend Sending Policy Check:** Determine if workflow should proceed based on weekend settings.
- **1.3 Fetch and Filter Candidates:** Retrieve candidate data from Google Sheets and filter those eligible for rejection emails.
- **1.4 Candidate Processing Decision:** Check if any candidates remain after filtering; branch for dry-run or actual sending.
- **1.5 Email Preparation:** Prepare personalized email subject and body (HTML and text) per candidate.
- **1.6 Email Sending Loop:** Rate-limit sending emails, send each email via Gmail, update the sheet, and log success.
- **1.7 No Candidates or Skip Message:** Handle cases where no candidates to email or skipping due to weekend policy.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Schedule and Configuration Setup

**Overview:**  
Triggers the workflow daily at 6 PM and sets all configurable parameters used throughout the workflow.

**Nodes Involved:**  
- Schedule Trigger  
- Set: Config

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow daily at specified time.  
  - Configuration: Cron expression `"0 18 * * *"` (6 PM every day).  
  - Inputs: None (trigger only).  
  - Outputs: Triggers next node.  
  - Edge cases: If n8n instance is down at scheduled time, trigger is missed; no retries.

- **Set: Config**  
  - Type: Set  
  - Role: Defines constants and templates for the workflow, such as spreadsheet ID, sheet name, timezone, rejection statuses, email templates, SMTP sender, rate limit, weekend inclusion, and dry run flag.  
  - Configuration highlights:  
    - `SPREADSHEET_ID`: Google Sheets document ID.  
    - `SOURCE_SHEET`: Sheet tab name ("Candidate Status").  
    - `TIMEZONE`: "Asia/Kolkata" for date/time operations if needed.  
    - `REJECT_STATUS_CSV`: CSV string of statuses considered rejected ("Rejected").  
    - Email templates for subject, HTML body, text body with placeholders.  
    - `RATE_LIMIT_SECONDS`: 10 seconds between emails.  
    - `INCLUDE_WEEKENDS`: boolean flag whether to process on weekends.  
    - `DRY_RUN`: boolean flag to simulate sending without actual emails.  
  - Inputs: Triggered by Schedule Trigger.  
  - Outputs: Configuration data for downstream nodes.  
  - Edge cases: Misconfiguration (e.g., wrong spreadsheet ID) will cause downstream failures.

---

#### 2.2 Weekend Sending Policy Check

**Overview:**  
Determines if the workflow should proceed based on whether sending emails on weekends is allowed.

**Nodes Involved:**  
- Check Weekend Policy  
- Weekend Skip Message

**Node Details:**

- **Check Weekend Policy**  
  - Type: If (Boolean conditions)  
  - Role: Decides if workflow proceeds based on weekend flag and current day.  
  - Logic: Proceeds if either `INCLUDE_WEEKENDS` is true or current day is a weekday (Mon-Fri).  
  - Inputs: Config from Set node.  
  - Outputs: Two branches: true (proceed), false (skip).  
  - Edge cases: Timezone on server could affect day calculation; logic uses JavaScript `new Date().getDay()` which returns 0 (Sun) to 6 (Sat).

- **Weekend Skip Message**  
  - Type: Set  
  - Role: Outputs a skip message with timestamp if workflow is skipping execution on weekend.  
  - Inputs: From false branch of Check Weekend Policy.  
  - Outputs: End of workflow path for skipping execution.

---

#### 2.3 Fetch and Filter Candidates

**Overview:**  
Retrieves candidate data from Google Sheets and filters candidates eligible for rejection emails.

**Nodes Involved:**  
- Fetch Candidate Data  
- Filter Candidates for Rejection

**Node Details:**

- **Fetch Candidate Data**  
  - Type: Google Sheets (Read)  
  - Role: Reads all rows from "Candidate Status" sheet in configured spreadsheet.  
  - Configuration: Uses OAuth2 credential for Google Sheets.  
  - Inputs: From Check Weekend Policy true path.  
  - Outputs: All candidate rows as JSON objects.  
  - Edge cases: API quota limits, invalid sheet name, or permissions errors can cause failures.

- **Filter Candidates for Rejection**  
  - Type: Code (JavaScript)  
  - Role: Filters candidates based on multiple criteria:  
    - Must have candidate_email.  
    - Must not have rejection_sent_at already set.  
    - Status must be in rejection status list from config.  
    - Required fields (`candidate_name`, `role`, `company_name`, `recruiter_name`) must be present.  
  - Outputs: Candidates to email or empty if none.  
  - Logs count of candidates filtered and skipped with reasons.  
  - Edge cases: Missing or malformed data rows, case-sensitive status matching, empty fields.

---

#### 2.4 Candidate Processing Decision

**Overview:**  
Checks if any candidates remain after filtering and routes workflow to dry-run preview, actual email sending, or no-candidates message.

**Nodes Involved:**  
- Has Candidates to Email?  
- Is Dry Run?  
- No Candidates Message  
- Dry Run Preview  
- Process Email Template

**Node Details:**

- **Has Candidates to Email?**  
  - Type: If  
  - Role: Checks if input JSON array is not empty.  
  - True branch: At least one candidate to email.  
  - False branch: No candidates; triggers no candidates message.  
  - Edge cases: Empty arrays or null input.

- **Is Dry Run?**  
  - Type: If  
  - Role: Checks DRY_RUN flag from config.  
  - True branch: Proceeds to dry run preview.  
  - False branch: Proceeds to prepare actual email content.  
  - Edge cases: Misconfigured DRY_RUN flag format.

- **No Candidates Message**  
  - Type: Set  
  - Role: Outputs message indicating no candidates found to email today with timestamp.  
  - Inputs: False output of Has Candidates node.  
  - Outputs: End of workflow path.

- **Dry Run Preview**  
  - Type: Code  
  - Role: Logs preview of emails that would be sent without sending them. Prints candidate email, subject, status, role.  
  - Outputs: Message summarizing dry run count.  
  - Edge cases: Large candidate lists may cause large logs.

- **Process Email Template**  
  - Type: Code  
  - Role: For each candidate, generates personalized email content based on templates and feedback fields:  
    - Substitutes placeholders in subject, HTML body, and text body.  
    - Adds feedback section if present.  
    - Adds SMTP sender email.  
  - Outputs: Enriched candidate JSON with `email_subject`, `email_html`, `email_text`, and `smtp_from`.  
  - Edge cases: Missing template fields, malformed feedback text.

---

#### 2.5 Email Sending Loop

**Overview:**  
Sends rejection emails one by one respecting rate limit, updates Google Sheets marking email sent, and logs success.

**Nodes Involved:**  
- Rate Limit Wait  
- Send Rejection Email1  
- Mark as Sent in Sheet  
- Log Success

**Node Details:**

- **Rate Limit Wait**  
  - Type: Wait  
  - Role: Pauses between sending emails for configured seconds (default 10 seconds) to avoid rate limits.  
  - Inputs: From Process Email Template.  
  - Outputs: Triggers Send Email node after delay.  
  - Edge cases: Delays add to overall execution time; long lists could exceed workflow time limits.

- **Send Rejection Email1**  
  - Type: Gmail  
  - Role: Sends email to candidate using Gmail OAuth credentials.  
  - Configuration:  
    - To: candidate email from JSON.  
    - Subject and HTML body from processed template.  
    - Uses OAuth2 Gmail credential.  
  - Inputs: From Rate Limit Wait.  
  - Outputs: On success, triggers marking email as sent.  
  - Edge cases: Gmail API limits, authentication failures, invalid email addresses, message size limits.

- **Mark as Sent in Sheet**  
  - Type: Google Sheets (Update)  
  - Role: Updates candidate row in sheet setting `rejection_sent_at` timestamp to current time, marking email sent.  
  - Matching: Uses candidate email to find row.  
  - Inputs: From Send Email node.  
  - Outputs: Triggers logging.  
  - Edge cases: Sheet update conflicts, row not found, permissions.

- **Log Success**  
  - Type: Set  
  - Role: Logs success message with candidate name, email, and timestamp for audit.  
  - Inputs: From Sheet update node.  
  - Outputs: End of loop for candidate.  
  - Edge cases: None critical.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                              | Input Node(s)             | Output Node(s)                      | Sticky Note                                                                                                 |
|---------------------------|-------------------------|----------------------------------------------|---------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger        | Daily trigger at 6 PM                         | -                         | Set: Config                       |                                                                                                             |
| Set: Config              | Set                     | Loads and defines config variables & templates| Schedule Trigger           | Check Weekend Policy              |                                                                                                             |
| Check Weekend Policy      | If                      | Checks if emails should be sent on weekend   | Set: Config                | Fetch Candidate Data / Weekend Skip Message |                                                                                                             |
| Weekend Skip Message      | Set                     | Outputs skip message when skipping weekends  | Check Weekend Policy (false) | -                               |                                                                                                             |
| Fetch Candidate Data      | Google Sheets           | Reads candidate rows from spreadsheet        | Check Weekend Policy (true) | Filter Candidates for Rejection   |                                                                                                             |
| Filter Candidates for Rejection | Code (JavaScript)        | Filters candidates eligible for rejection    | Fetch Candidate Data       | Has Candidates to Email          |                                                                                                             |
| Has Candidates to Email?  | If                      | Checks if there are candidates to process    | Filter Candidates for Rejection | Is Dry Run? / No Candidates Message |                                                                                                             |
| Is Dry Run?               | If                      | Branches workflow into dry-run or actual send| Has Candidates to Email?   | Dry Run Preview / Process Email Template |                                                                                                             |
| No Candidates Message     | Set                     | Outputs message when no candidates found     | Has Candidates to Email? (false) | -                               |                                                                                                             |
| Dry Run Preview           | Code (JavaScript)       | Logs preview info of emails without sending  | Is Dry Run? (true)         | -                                 |                                                                                                             |
| Process Email Template    | Code (JavaScript)       | Generates personalized email subject and body| Is Dry Run? (false)        | Rate Limit Wait                  |                                                                                                             |
| Rate Limit Wait           | Wait                    | Enforces delay between sending emails        | Process Email Template     | Send Rejection Email1            |                                                                                                             |
| Send Rejection Email1     | Gmail                   | Sends rejection email                         | Rate Limit Wait            | Mark as Sent in Sheet            |                                                                                                             |
| Mark as Sent in Sheet     | Google Sheets           | Updates sheet marking rejection email sent   | Send Rejection Email1      | Log Success                     |                                                                                                             |
| Log Success               | Set                     | Logs success message after sending email     | Mark as Sent in Sheet      | -                                 |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set Cron Expression to `0 18 * * *` (6 PM daily).

2. **Add Set Node "Set: Config"**  
   - Define variables:  
     - `SPREADSHEET_ID`: Google Sheets document ID containing candidates.  
     - `SOURCE_SHEET`: "Candidate Status".  
     - `TIMEZONE`: "Asia/Kolkata".  
     - `REJECT_STATUS_CSV`: "Rejected".  
     - `SMTP_FROM`: "careers@company.com".  
     - `SUBJECT_TEMPLATE`: "Regarding your application for {{role}} at {{company_name}}".  
     - `HTML_TEMPLATE`: Full HTML rejection email template with placeholders (`{{candidate_name}}`, etc.).  
     - `TEXT_TEMPLATE`: Plain text version of the email.  
     - `RATE_LIMIT_SECONDS`: 10 (seconds between emails).  
     - `INCLUDE_WEEKENDS`: true or false depending on policy.  
     - `DRY_RUN`: false (set to true for testing without sending emails).  
   - Connect Schedule Trigger output to this node.

3. **Add If Node "Check Weekend Policy"**  
   - Condition: proceed if `INCLUDE_WEEKENDS` is true OR current day is weekday (Mon-Fri).  
   - Use expression for current day: `{{ [1, 2, 3, 4, 5].includes(new Date().getDay()) }}`.  
   - Connect from Set: Config node.

4. **Add Set Node "Weekend Skip Message"**  
   - Assign `message`: "Skipping execution - Weekend and INCLUDE_WEEKENDS is false".  
   - Assign `timestamp`: current ISO datetime.  
   - Connect false branch of Check Weekend Policy.

5. **Add Google Sheets Node "Fetch Candidate Data"**  
   - Operation: Read rows from `SOURCE_SHEET` in `SPREADSHEET_ID`.  
   - Use OAuth2 credential for Google Sheets.  
   - Connect true branch of Check Weekend Policy.

6. **Add Code Node "Filter Candidates for Rejection"**  
   - JavaScript code to filter candidates:  
     - Skip if no email.  
     - Skip if rejection email already sent (`rejection_sent_at` set).  
     - Only include rows with `status` in `REJECT_STATUS_CSV`.  
     - Validate required fields (`candidate_name`, `role`, `company_name`, `recruiter_name`).  
   - Connect from Fetch Candidate Data.

7. **Add If Node "Has Candidates to Email?"**  
   - Check if filtered candidates array is non-empty.  
   - Connect from Filter Candidates.

8. **Add If Node "Is Dry Run?"**  
   - Check if `DRY_RUN` is true in config.  
   - Connect true branch of Has Candidates to Email.

9. **Add Code Node "Dry Run Preview"**  
   - Log info about candidates and emails to be sent without sending.  
   - Connect true branch of Is Dry Run.

10. **Add Code Node "Process Email Template"**  
    - For each candidate, replace template placeholders in subject, HTML, and text bodies.  
    - Include feedback if present.  
    - Add `smtp_from` field from config.  
    - Connect false branch of Is Dry Run.

11. **Add Wait Node "Rate Limit Wait"**  
    - Wait duration set from `RATE_LIMIT_SECONDS` variable.  
    - Connect from Process Email Template.

12. **Add Gmail Node "Send Rejection Email1"**  
    - Use Gmail OAuth2 credentials.  
    - Send To: candidate's email from JSON.  
    - Subject, HTML message from processed template.  
    - Connect from Rate Limit Wait.

13. **Add Google Sheets Node "Mark as Sent in Sheet"**  
    - Operation: Update row in sheet matching candidate email.  
    - Set `rejection_sent_at` to current ISO timestamp.  
    - Connect from Send Rejection Email node.

14. **Add Set Node "Log Success"**  
    - Assign message: "Rejection email sent successfully to {{candidate_email}}".  
    - Assign candidate_name and sent_at timestamp.  
    - Connect from Mark as Sent in Sheet.

15. **Add Set Node "No Candidates Message"**  
    - Assign message: "No candidates found for rejection emails today".  
    - Assign timestamp.  
    - Connect false branch of Has Candidates to Email.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| The workflow respects a configurable weekend sending policy, allowing skipping emails on weekends if desired. | Weekend handling logic uses JavaScript `Date().getDay()` which depends on server timezone.                 |
| The use of dry-run mode enables testing without sending emails, logging the intended messages instead.         | DRY_RUN flag in config controls this behavior.                                                             |
| Rate limiting with a configurable wait node helps avoid email provider throttling or blocking.                  | Default 10 seconds delay between emails, configurable via RATE_LIMIT_SECONDS.                              |
| Email templates support placeholders for personalization and can include interview feedback dynamically.       | Placeholders: `{{candidate_name}}`, `{{role}}`, `{{company_name}}`, `{{recruiter_name}}`, `{{feedback_html}}`|
| Google Sheets rows are matched by candidate email to update the `rejection_sent_at` timestamp post email send.  | Permission and API quota for Google Sheets OAuth2 credential must be ensured.                              |
| Gmail OAuth2 credentials are required for sending emails; ensure these are valid and authorized.                |                                                                                                           |

---

**Disclaimer:** The provided content is derived exclusively from an n8n workflow automation and complies with all applicable content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.