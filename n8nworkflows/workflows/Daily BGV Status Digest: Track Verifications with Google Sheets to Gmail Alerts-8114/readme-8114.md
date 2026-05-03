Daily BGV Status Digest: Track Verifications with Google Sheets to Gmail Alerts

https://n8nworkflows.xyz/workflows/daily-bgv-status-digest--track-verifications-with-google-sheets-to-gmail-alerts-8114


# Daily BGV Status Digest: Track Verifications with Google Sheets to Gmail Alerts

### 1. Workflow Overview

This workflow automates a daily digest email summarizing the status of background verifications (BGV) tracked in a Google Sheet. It is designed for organizations that need to monitor and communicate verification progress efficiently among executives responsible for candidate background checks.

The workflow logically divides into these blocks:

- **1.1 Scheduled Trigger:** Automatically initiates the workflow every day at a specified time.
- **1.2 Data Retrieval:** Reads all candidate background verification data from a designated Google Sheets document.
- **1.3 Data Normalization & Parsing:** Cleans and normalizes raw sheet data, parsing dates and calculating flags such as completion status and staleness.
- **1.4 Grouping & Filtering:** Organizes data by executive email, filtering for relevant completed and pending items per executive.
- **1.5 Digest Formatting:** Generates personalized HTML email content and subject lines summarizing the verification statuses for each executive.
- **1.6 Email Dispatch:** Sends the personalized digest emails via Gmail to each executive.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically once per day, ensuring the digest is generated and sent consistently every night.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow at a recurring interval.  
    - Configuration: Configured to run once every day (default daily interval).  
    - Input Connections: None (trigger node).  
    - Output Connections: Connects to the Google Sheets node.  
    - Version: 1.2  
    - Edge Cases: If the node or n8n instance is stopped or inactive at the scheduled time, the trigger will not fire; no retries.  
    - Credentials: None required.

---

#### 2.2 Data Retrieval

- **Overview:**  
  Retrieves the current status data of candidate background verifications from the Google Sheets document named "BGV Tracker".

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets  
    - Role: Reads all rows from the specified Google Sheet tab.  
    - Configuration:  
      - Document ID: `132inVcr60cNg5AUY51aKG-4R8UgQ6xlGgLgC3ZwKxuU` (BGV Tracker spreadsheet)  
      - Sheet Name: `gid=0` (default first tab)  
      - Reads all rows without filters.  
    - Input Connections: Connected from Schedule Trigger.  
    - Output Connections: Connected to "Normalize & Parse" node.  
    - Credentials: Uses OAuth2 credentials for Google Sheets access.  
    - Version: 4.6  
    - Edge Cases:  
      - Authentication failures due to expired or revoked OAuth tokens.  
      - Rate limiting from Google Sheets API.  
      - Changes in sheet structure (e.g., renamed columns) can cause downstream parsing issues.

---

#### 2.3 Data Normalization & Parsing

- **Overview:**  
  Standardizes the raw data by lowercasing keys, parses date fields in multiple formats, and calculates flags such as whether a verification was completed today or if a pending item is stale.

- **Nodes Involved:**  
  - Normalize & Parse (Code node)

- **Node Details:**

  - **Normalize & Parse**  
    - Type: Code (JavaScript)  
    - Role:  
      - Converts all keys to lowercase for consistent referencing.  
      - Parses date fields `bgv_completion_date` and `last_follow_up` supporting various formats including DD/MM/YYYY, DD-MM-YYYY, MM/DD/YYYY, and ISO strings.  
      - Calculates boolean flags:  
        - `isCompletedToday`: True if status is "Completed" and completion date is today (IST timezone).  
        - `isStale`: True if status is "Pending" and last follow-up was 3 or more days ago.  
      - Calculates `daysSinceFollowUp` to help determine staleness.  
    - Input Connections: Connected from Google Sheets node.  
    - Output Connections: Connected to "Group & Filter" node.  
    - Version: 2  
    - Key Expressions: JavaScript date parsing logic with Indian Standard Time (UTC+5:30) adjustments.  
    - Edge Cases:  
      - Invalid or missing date formats result in null parsed dates.  
      - Timezone handling relies on heuristic; date-only comparisons assume no time component.  
      - If input data has unexpected keys or missing fields, flags may be inaccurate.  
      - Parsing failures do not error the node but produce nulls, which downstream logic must handle.

---

#### 2.4 Grouping & Filtering

- **Overview:**  
  Groups verification entries by executive email and filters the records into categories: "Completed Today" and "Pending" (excluding those marked "To be Sent"). This prepares data for per-executive personalized digests.

- **Nodes Involved:**  
  - Group & Filter (Code node)

- **Node Details:**

  - **Group & Filter**  
    - Type: Code (JavaScript)  
    - Role:  
      - Creates a map keyed by `bgv_exe_email` (executive email).  
      - For each executive, collects:  
        - Completed verifications for today (`isCompletedToday` true).  
        - Pending verifications with status "Pending" but not "To be Sent".  
      - Preserves stale flags per item for highlighting.  
    - Input Connections: Connected from "Normalize & Parse".  
    - Output Connections: Connected to "Format Digest".  
    - Version: 2  
    - Edge Cases:  
      - Rows without an executive email are skipped silently.  
      - If an executive has no completed or pending items, their group will be empty and result in minimal or no output.  
      - Assumes `bgv_exe_email` is a valid email for later use in sending mail.  
      - No handling for duplicate emails with different casing or whitespace.

---

#### 2.5 Digest Formatting

- **Overview:**  
  Composes the personalized email subject and HTML content for each executive, summarizing completed verifications and pending cases, highlighting stale pending items with a warning icon.

- **Nodes Involved:**  
  - Format Digest (Code node)

- **Node Details:**

  - **Format Digest**  
    - Type: Code (JavaScript)  
    - Role:  
      - Escapes HTML content to prevent injection.  
      - Creates a subject line with the date and counts of completed and pending items.  
      - Builds HTML tables summarizing:  
        - Completed today candidates (basic info).  
        - Pending candidates, including last follow-up date and status with stale warning.  
      - Adds explanatory notes for stale items.  
      - Sets output JSON properties:  
        - `to` (executive email),  
        - `subject`,  
        - `html` (email body).  
    - Input Connections: Connected from "Group & Filter".  
    - Output Connections: Connected to Gmail node.  
    - Version: 2  
    - Edge Cases:  
      - Empty lists result in a message "None" instead of a table.  
      - Assumes valid emails and data fields are present; missing fields render empty table cells.  
      - Date in subject uses local IST with manual offset adjustment, which may drift if server time changes.  
      - HTML formatting errors could cause malformed emails if inputs contain unexpected characters.

---

#### 2.6 Email Dispatch

- **Overview:**  
  Sends the personalized digest email to each executive using their email address, subject, and message body prepared earlier.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**

  - **Gmail**  
    - Type: Gmail  
    - Role: Sends email messages.  
    - Configuration:  
      - `sendTo`: Uses expression with `{{$json.to}}` (executive email).  
      - `subject`: Uses `{{$json.subject}}`.  
      - `message`: Uses `{{$json.html}}` for HTML body.  
      - Options: Attribution disabled (no "Sent via n8n" footer).  
    - Input Connections: Connected from "Format Digest".  
    - Output Connections: None (end node).  
    - Credentials: Uses OAuth2 Gmail credentials.  
    - Version: 2.1  
    - Edge Cases:  
      - Authentication or permission issues with Gmail OAuth2.  
      - Email sending failures due to invalid recipient addresses or quota limits.  
      - HTML rendering issues in received email clients.

---

#### 2.7 Sticky Notes (Documentation Nodes)

- **Sticky Note**  
  - Content: Title and summary describing the workflow as an automation for sending daily BGV executive digests.

- **Sticky Note1**  
  - Content: Detailed node list and descriptions, plus a quick reference explaining the flow logic step-by-step.  
  - These notes provide human-readable documentation to aid understanding and maintenance.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                               | Input Node(s)       | Output Node(s)     | Sticky Note                                                                                                  |
|--------------------|---------------------|-----------------------------------------------|---------------------|--------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger    | Triggers workflow daily                        | None                | Google Sheets       | Kicks off the workflow every night at 23:00 IST ensuring consistent digests.                                 |
| Google Sheets      | Google Sheets       | Reads candidate BGV data from Google Sheet    | Schedule Trigger     | Normalize & Parse   | Reads all candidate data from “BGV Tracker” tab to pull latest statuses.                                     |
| Normalize & Parse  | Code                | Normalizes keys, parses dates, flags statuses | Google Sheets       | Group & Filter      | Parses `bgv_completion_date` and `last_follow_up`, adds flags like `isCompletedToday` and `isStale`.          |
| Group & Filter     | Code                | Groups data by executive email, filters items | Normalize & Parse   | Format Digest       | Groups rows by `bgv_exe_email`, separates completed today and pending items, excludes "To be Sent" status.   |
| Format Digest      | Code                | Formats personalized email subject & body     | Group & Filter      | Gmail               | Creates HTML email digest with tables for completed and pending candidates, highlights stale items with ⚠️. |
| Gmail              | Gmail               | Sends personalized digest emails               | Format Digest       | None                | Sends emails to executives with daily BGV status updates automatically.                                      |
| Sticky Note        | Sticky Note         | Documentation                                  | None                | None                | ##BGV Executive Digest Automation: Track Completed & Pending Verifications via Email (from Google Sheets)    |
| Sticky Note1       | Sticky Note         | Documentation                                  | None                | None                | Detailed node list & descriptions; quick reference flow logic explained step-by-step.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Node Type: Schedule Trigger  
   - Configure interval: Daily (every 24 hours), default time (adjust as per time zone needs, ideally 23:00 IST).  
   - No credentials needed.

2. **Add Google Sheets Node**  
   - Node Type: Google Sheets  
   - Credentials: Configure OAuth2 credentials with access to the target Google Sheet.  
   - Parameters:  
     - Document ID: `132inVcr60cNg5AUY51aKG-4R8UgQ6xlGgLgC3ZwKxuU`  
     - Sheet Name: `gid=0` (default tab)  
     - Operation: Read rows (no filters).  
   - Connect Schedule Trigger output to Google Sheets input.

3. **Add Code Node: Normalize & Parse**  
   - Node Type: Code  
   - Copy the JavaScript code that:  
     - Converts all keys to lowercase.  
     - Parses `bgv_completion_date` and `last_follow_up` fields supporting multiple date formats.  
     - Calculates `isCompletedToday`, `isStale`, and `daysSinceFollowUp` flags.  
     - Uses IST timezone for date calculations.  
   - Connect Google Sheets output to this node.

4. **Add Code Node: Group & Filter**  
   - Node Type: Code  
   - Implement JavaScript code that:  
     - Groups items by `bgv_exe_email`.  
     - Filters each group’s items into arrays: completed today and pending (excluding "To be Sent").  
     - Does not include rows missing executive emails.  
   - Connect "Normalize & Parse" output to this node.

5. **Add Code Node: Format Digest**  
   - Node Type: Code  
   - Include code that:  
     - Escapes HTML.  
     - Generates a subject line with date and counts.  
     - Builds HTML tables summarizing completed and pending candidates per executive.  
     - Adds stale warnings on pending items.  
     - Outputs JSON with `to`, `subject`, and `html` keys for email sending.  
   - Connect "Group & Filter" output to this node.

6. **Add Gmail Node**  
   - Node Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials with send email permissions.  
   - Parameters:  
     - Send To: expression `{{$json.to}}`  
     - Subject: expression `{{$json.subject}}`  
     - Message (HTML): expression `{{$json.html}}`  
     - Options: Disable attribution/append footer.  
   - Connect "Format Digest" output to Gmail input.

7. **Add Sticky Notes (Optional for Documentation)**  
   - Create two sticky notes summarizing the workflow title and node descriptions for quick reference.

8. **Set Workflow Settings**  
   - Execution order can remain default.  
   - Activate the workflow to enable scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow is configured for IST timezone (UTC+5:30) with manual offset adjustments in code nodes. Adjust as needed for different timezones. | Timezone handling in JavaScript code nodes for date parsing and comparisons.                     |
| Stale pending items are flagged if no follow-up has occurred in 3 or more days, marked by a ⚠️ icon in emails.        | Helps executives prioritize overdue background verifications.                                   |
| The Google Sheets node requires OAuth2 credentials with at least read access to the specified spreadsheet.           | Credential setup in n8n Google Sheets OAuth2 integration.                                       |
| Gmail node requires OAuth2 Gmail credentials with send permissions enabled to dispatch emails.                       | Credential setup in n8n Gmail OAuth2 integration.                                               |
| HTML escaping is implemented in the formatting node to prevent injection and malformed emails.                      | Security best practice in email content generation.                                             |
| The workflow is designed to run unattended nightly, reducing manual reporting overhead for BGV teams.                | Operational efficiency improvement.                                                             |
| Workflow version dependencies: Google Sheets node v4.6+, Gmail node v2.1+, n8n code nodes v2.                        | Ensure n8n environment is up to date to support these node versions and features.               |
| Sticky notes in the workflow provide embedded documentation and quick node reference for maintainers.                | Helps with future modifications and onboarding new users.                                       |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.