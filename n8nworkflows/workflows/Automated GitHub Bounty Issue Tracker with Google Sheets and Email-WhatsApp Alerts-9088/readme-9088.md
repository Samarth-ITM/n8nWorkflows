Automated GitHub Bounty Issue Tracker with Google Sheets and Email/WhatsApp Alerts

https://n8nworkflows.xyz/workflows/automated-github-bounty-issue-tracker-with-google-sheets-and-email-whatsapp-alerts-9088


# Automated GitHub Bounty Issue Tracker with Google Sheets and Email/WhatsApp Alerts

### 1. Workflow Overview

This workflow automates the tracking of GitHub bounty issues labeled with "💎 Bounty" and manages their lifecycle in Google Sheets while sending notifications via email and optionally WhatsApp. It is designed for open source bounty hunters, project maintainers, or community managers who want to monitor bounty issues efficiently and get alerts on new bounties or updates on existing ones.

The workflow is logically divided into two main sections:

- **1.1 New Bounty Discovery and Notification (Hourly Trigger):**  
  Fetches open bounty issues from GitHub, compares them with existing records in Google Sheets, appends new bounties to the sheet, and sends notifications for recent bounties.

- **1.2 Existing Bounty Status Update (Every 6 Hours):**  
  Checks previously tracked bounty issues for state or comment count changes and updates Google Sheets accordingly to keep data current.

---

### 2. Block-by-Block Analysis

---

#### 2.1 New Bounty Discovery and Notification

**Overview:**  
This block runs every hour to fetch all open GitHub bounty issues, identifies new bounties not already tracked in the main Google Sheet (Sheet1), appends these new entries, checks if they qualify for notification based on recency, and sends notification emails. WhatsApp notifications exist but are disabled.

**Nodes Involved:**  
- Schedule Trigger  
- GitHub Search HTTP Request  
- Split Out  
- Get row(s) in sheet (Sheet1)  
- Get row(s) in sheet2  
- Merge  
- Filter  
- Append row in sheet (Sheet1)  
- Filter1  
- Send message (WhatsApp - Disabled)  
- HTML  
- Send a message (Gmail)  
- Sticky Notes (1, 2, 3, 4, 5, 11, 12)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow every hour  
  - Config: Interval set to 1 hour  
  - Outputs: Triggers GitHub Search HTTP Request, and two Google Sheets read nodes in parallel  
  - Edge Cases: Trigger failure is rare, ensure time zone consistency  

- **GitHub Search HTTP Request**  
  - Type: HTTP Request  
  - Role: Queries GitHub API for open issues labeled "💎 Bounty" (max 100 per page, paginated)  
  - Config: URL `https://api.github.com/search/issues` with query `is:issue label:"💎 Bounty" state:open` and pagination enabled  
  - Auth: Bearer token via HTTP Bearer Auth credentials  
  - Outputs: JSON with items array of issues  
  - Edge Cases: GitHub API rate limiting, auth errors, pagination errors  

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of issues into individual items for processing  
  - Config: Split on field `items` from GitHub response  
  - Inputs: Output from GitHub Search HTTP Request  
  - Outputs: Individual issue JSON objects  

- **Get row(s) in sheet (Sheet1)**  
  - Type: Google Sheets  
  - Role: Fetches all existing bounty records from Sheet1 for comparison  
  - Config: Reads Sheet1 (`gid=0`) from the Google Spreadsheet  
  - Auth: Google Sheets OAuth2  
  - Outputs: Rows representing tracked bounties  

- **Get row(s) in sheet2**  
  - Type: Google Sheets  
  - Role: Fetches recent bounty entries for notification control (Sheet2)  
  - Config: Reads Sheet2 (`gid=1153061999`)  
  - Auth: Google Sheets OAuth2  

- **Merge**  
  - Type: Merge  
  - Role: Combines GitHub issues with Sheet1 data to match issues by URL, keeping unmatched GitHub issues for new bounty detection  
  - Config: Join mode `keepNonMatches`, merge by fields: GitHub `html_url` with Sheet1 `Issue URL`  
  - Inputs: Split Out (GitHub issues) and Get row(s) in sheet (Sheet1)  
  - Outputs: Combined data for filtering  

- **Filter**  
  - Type: Filter  
  - Role: Filters issues that are NOT found in Sheet1's "Issue Name" column, i.e., new bounties only  
  - Condition: Checks if the issue title is NOT contained in Sheet1's "Issue Name" column  
  - Inputs: Output from Merge  
  - Outputs: New bounty items  

- **Append row in sheet (Sheet1)**  
  - Type: Google Sheets  
  - Role: Adds new bounty issues to Sheet1 with relevant details (URL, name, state, repo URL, creation and update dates)  
  - Auth: Google Sheets OAuth2  
  - Inputs: Output from Filter  
  - Outputs: Confirmation of append operation  

- **Filter1**  
  - Type: Filter  
  - Role: Filters bounties to notify about: created within last 5 days and not already present in Sheet2 (notification control)  
  - Conditions:  
    - Issue Creation Date >= now minus 5 days  
    - Issue URL not in Sheet2's "Issue URL" column  
  - Inputs: Append row in sheet results  
  - Outputs: Items qualifying for notification  

- **Send message (WhatsApp - Disabled)**  
  - Type: WhatsApp  
  - Role: Intended to send new bounty notifications via WhatsApp (currently disabled)  
  - Config: Sends formatted text message with issue details to a specific phone number  
  - Auth: WhatsApp API credentials  
  - Edge Cases: Disabled, requires WhatsApp API setup and quota  

- **HTML**  
  - Type: HTML  
  - Role: Generates styled HTML email body for new bounty notifications  
  - Config: GitHub-themed email template with issue details such as title, number, status, bounty amount, comments, repo, and link  
  - Inputs: Data from Filter (new bounties)  
  - Outputs: HTML content for email  

- **Send a message (Gmail)**  
  - Type: Gmail  
  - Role: Sends the formatted HTML email alert for new bounties  
  - Config: Recipient email, subject "New Bounty Alert!", disables default attribution  
  - Auth: Gmail OAuth2 credentials  
  - Inputs: HTML content node output  
  - Outputs: Email sent confirmation  

- **Sticky Notes**  
  - Provide documentation and guidance on workflow sections, e.g., trigger frequency, data merging, filtering logic, notification rules, and disabled WhatsApp node.

---

#### 2.2 Existing Bounty Status Update

**Overview:**  
Runs every 6 hours to update the status of existing bounty issues tracked in Google Sheets. It fetches current issue data from GitHub for Sheet1 and Sheet2 entries, detects any changes in issue state or comments, and updates the sheets accordingly.

**Nodes Involved:**  
- Schedule Trigger1  
- Get row(s) in sheet1 (Sheet2)  
- Get row(s) in sheet3 (Sheet1)  
- GitHub Search HTTP Request1  
- GitHub Search HTTP Request2  
- Filter2  
- Filter3  
- Update row in sheet (Sheet2)  
- Update row in sheet1 (Sheet1)  
- Sticky Notes (6, 7, 8, 9, 10)

**Node Details:**

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Runs the update process every 6 hours  
  - Config: Interval 6 hours  

- **Get row(s) in sheet1 (Sheet2)**  
  - Type: Google Sheets  
  - Role: Reads all bounty entries from Sheet2 (recent bounties for notification)  
  - Auth: Google Sheets OAuth2  

- **Get row(s) in sheet3 (Sheet1)**  
  - Type: Google Sheets  
  - Role: Reads all bounty entries from Sheet1 (full bounty list)  
  - Auth: Google Sheets OAuth2  

- **GitHub Search HTTP Request1**  
  - Type: HTTP Request  
  - Role: Fetches current GitHub issue data for each bounty in Sheet2, using the repo URL and issue number  
  - Config: Uses dynamic URL: converts repo URL to GitHub API repo path and appends issue number  
  - Auth: Bearer token  
  - Inputs: Output from Get row(s) in sheet1 (Sheet2)  

- **GitHub Search HTTP Request2**  
  - Type: HTTP Request  
  - Role: Fetches current GitHub issue data for each bounty in Sheet1, similar to above  
  - Auth: Bearer token  
  - Inputs: Output from Get row(s) in sheet3 (Sheet1)  

- **Filter2**  
  - Type: Filter  
  - Role: Determines if the issue state or comment count for Sheet2 bounties has changed compared to stored data  
  - Conditions:  
    - Issue creation date is within last 5 days  
    - Issue URL not already in Sheet2 (to avoid duplicate notifications)  
  - Inputs: GitHub Search HTTP Request1 outputs  
  - Outputs: Changed items for update  

- **Filter3**  
  - Type: Filter  
  - Role: Detects changes in Sheet1 entries based on issue state or updated timestamp differences  
  - Conditions:  
    - Issue state differs from stored state  
    - Updated timestamp differs from stored timestamp  
  - Inputs: GitHub Search HTTP Request2 outputs  

- **Update row in sheet (Sheet2)**  
  - Type: Google Sheets  
  - Role: Updates Sheet2 rows with new issue state and comment count if changes detected  
  - Inputs: Filter2 outputs  
  - Auth: Google Sheets OAuth2  

- **Update row in sheet1 (Sheet1)**  
  - Type: Google Sheets  
  - Role: Updates Sheet1 rows with latest issue state and updated date if changes detected  
  - Inputs: Filter3 outputs  
  - Auth: Google Sheets OAuth2  

- **Sticky Notes**  
  - Document update trigger frequency, change detection logic, and update actions on sheets.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                                      | Input Node(s)                       | Output Node(s)                  | Sticky Note                                                                                   |
|----------------------------|-------------------------|-----------------------------------------------------|-----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger        | Starts hourly new bounty discovery                   | -                                 | GitHub Search HTTP Request, Get row(s) in sheet, Get row(s) in sheet2 | ## Main Workflow Trigger Runs every hour to check for new GitHub bounty issues with the '💎 Bounty' label |
| GitHub Search HTTP Request | HTTP Request            | Fetches open GitHub bounty issues                    | Schedule Trigger                  | Split Out                      | ## Fetch & Split Bounties - Fetches all open GitHub issues with bounty label - Pagination enabled (100 per page) - Split Out converts array into individual items |
| Split Out                 | Split Out               | Splits GitHub issues array into individual items    | GitHub Search HTTP Request        | Merge                         | Same sticky note as GitHub Search HTTP Request                                               |
| Get row(s) in sheet       | Google Sheets           | Reads existing bounty issues from Sheet1             | Schedule Trigger                  | Merge                         | ## Compare with Existing Data - Merges GitHub results with Sheet1 to identify which bounties are already tracked |
| Merge                     | Merge                   | Combines GitHub and Sheet1 data to find new bounties| Split Out, Get row(s) in sheet    | Filter                        | ## Compare with Existing Data (same as Get row(s) in sheet)                                  |
| Filter                    | Filter                  | Filters new bounties not in Sheet1                   | Merge                           | Append row in sheet           | ## Filter New Bounties Only - Keeps only issues where title is NOT found in Sheet1's "Issue Name" column |
| Append row in sheet       | Google Sheets           | Adds new bounty issues to Sheet1                      | Filter                          | Filter1                       | (No specific sticky note)                                                                    |
| Get row(s) in sheet2      | Google Sheets           | Reads recent bounties from Sheet2 for notification  | Schedule Trigger                 | Split Out                     | (No specific sticky note)                                                                    |
| Filter1                   | Filter                  | Filters bounties to notify: recent and not notified | Append row in sheet              | Send message (WhatsApp)       | ## Filter for Notifications After adding to Sheet1, checks if bounty should trigger notification: - Created within last 5 days - Not already sent (not in Sheet2) |
| Send message (WhatsApp)   | WhatsApp                | Sends WhatsApp alert for new bounties (disabled)    | Filter1                        | HTML                         | ## WhatsApp Notification (Disabled) Currently using Gmail instead - enable if needed          |
| HTML                      | HTML                    | Formats HTML email for new bounty notification       | Send message (WhatsApp)          | Send a message (Gmail)        | ## Format Email Template - Converts bounty data into styled HTML email with GitHub-themed design |
| Send a message (Gmail)    | Gmail                   | Sends email notification for new bounties            | HTML                          | Append row in sheet1          | (No specific sticky note)                                                                    |
| Append row in sheet1      | Google Sheets           | Adds notified bounty data to Sheet2                   | Send a message (Gmail)           | Filter1                      | (No specific sticky note)                                                                    |
| Schedule Trigger1         | Schedule Trigger        | Starts 6-hourly bounty status update                  | -                             | Get row(s) in sheet1, Get row(s) in sheet3 | ## Update Workflow Trigger - Runs every 6 hours to update existing bounty statuses          |
| Get row(s) in sheet1      | Google Sheets           | Reads bounty entries from Sheet2                       | Schedule Trigger1               | GitHub Search HTTP Request1   | ## Update Sheet2 Status - Fetches fresh data from GitHub API for each bounty in Sheet2 to detect changes |
| GitHub Search HTTP Request1| HTTP Request           | Fetches current GitHub issue data for Sheet2 bounties | Get row(s) in sheet1           | Filter2                      | ## Detect Changes (Sheet2) - Updates only if: - Issue state changed (open/closed) - Comment count changed |
| Filter2                   | Filter                  | Detects changes in Sheet2 bounty state or comments    | GitHub Search HTTP Request1     | Update row in sheet           | Same as GitHub Search HTTP Request1                                                        |
| Update row in sheet       | Google Sheets           | Updates Sheet2 rows with latest issue state/comments  | Filter2                        | -                            | (No specific sticky note)                                                                    |
| Get row(s) in sheet3      | Google Sheets           | Reads bounty entries from Sheet1                       | Schedule Trigger1               | GitHub Search HTTP Request2   | ## Update Sheet1 Status - Checks Sheet1 bounties for state or timestamp changes             |
| GitHub Search HTTP Request2| HTTP Request           | Fetches current GitHub issue data for Sheet1 bounties | Get row(s) in sheet3           | Filter3                      | ## Detect Changes (Sheet1) - Updates only if: - Issue state changed - Updated timestamp changed |
| Filter3                   | Filter                  | Detects changes in Sheet1 bounty state or updated time| GitHub Search HTTP Request2     | Update row in sheet1          | Same as GitHub Search HTTP Request2                                                        |
| Update row in sheet1      | Google Sheets           | Updates Sheet1 rows with latest issue state and update| Filter3                        | -                            | (No specific sticky note)                                                                    |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Credentials**  
- Create Google Sheets OAuth2 credentials with access to your target spreadsheet.  
- Create HTTP Bearer Auth credentials with a GitHub personal access token that has `repo` and `read:issues` scopes.  
- Create Gmail OAuth2 credentials for sending emails.  
- (Optional) Setup WhatsApp API credentials if WhatsApp notifications are desired.

---

**Step 2: Create Main Workflow Trigger (New Bounties)**  
- Add `Schedule Trigger` node: Set to run every 1 hour.

---

**Step 3: Fetch GitHub Bounty Issues**  
- Add `HTTP Request` node (GitHub Search HTTP Request):  
  - URL: `https://api.github.com/search/issues`  
  - Query parameters: `q=is:issue label:"💎 Bounty" state:open`, `per_page=100`  
  - Enable pagination with page parameter to fetch all pages  
  - Auth: GitHub Bearer token  
- Connect `Schedule Trigger` to this node.

---

**Step 4: Split GitHub Issues Array**  
- Add `Split Out` node: Split on field `items` from GitHub response.  
- Connect `GitHub Search HTTP Request` to this node.

---

**Step 5: Retrieve Existing Bounties from Google Sheets (Sheet1 and Sheet2)**  
- Add two `Google Sheets` nodes:  
  - `Get row(s) in sheet` reads Sheet1 (`gid=0`)  
  - `Get row(s) in sheet2` reads Sheet2 (`gid=1153061999`)  
- Connect `Schedule Trigger` to both nodes.

---

**Step 6: Merge GitHub and Sheet1 Data**  
- Add `Merge` node:  
  - Mode: Combine  
  - Join mode: Keep Non Matches  
  - Merge by fields: GitHub `html_url` and Sheet1 `Issue URL`  
- Connect `Split Out` to input 1 and `Get row(s) in sheet` to input 2.

---

**Step 7: Filter New Bounties**  
- Add `Filter` node:  
  - Condition: Issue title NOT in Sheet1 "Issue Name" column  
- Connect `Merge` output to this node.

---

**Step 8: Append New Bounties to Sheet1**  
- Add `Google Sheets` node:  
  - Operation: Append  
  - Sheet: Sheet1 (`gid=0`)  
  - Map fields: Issue URL, Issue Name, Issue state, Issue Number, Issue Repo URL (convert API URL to GitHub URL), Issue Updated Date, Issue Creation Date  
- Connect `Filter` to this node.

---

**Step 9: Filter Bounties for Notification**  
- Add `Filter1` node:  
  - Conditions:  
    - Issue Creation Date >= now - 5 days  
    - Issue URL NOT in Sheet2 "Issue URL" column  
- Connect `Append row in sheet` to this node.

---

**Step 10: (Optional) WhatsApp Notification**  
- Add `WhatsApp` node (disabled by default). Configure phone number and message template using issue data.  
- Connect `Filter1` output to this node.

---

**Step 11: Format HTML Email Notification**  
- Add `HTML` node: Use provided GitHub-themed HTML template. Insert issue details with expressions referencing `Filter` node.  
- Connect `Send message (WhatsApp)` node output to this node (or connect `Filter1` if WhatsApp is disabled).

---

**Step 12: Send Email Notification**  
- Add `Gmail` node:  
  - Recipient: your email  
  - Subject: "New Bounty Alert!"  
  - Message: Use HTML node output  
- Connect `HTML` node output to this node.

---

**Step 13: Append Notified Bounties to Sheet2**  
- Add `Google Sheets` node:  
  - Append operation on Sheet2 (`gid=1153061999`)  
  - Map fields: Issue URL, Issue Name, Issue state, Issue Number, Bounty Amount (extracted from labels), Issue Repo URL, Number of Issue Comments, Issue Bounty Creation Date  
- Connect `Send a message (Gmail)` node to this node.

---

**Step 14: Setup Update Workflow Trigger (Existing Bounties)**  
- Add `Schedule Trigger1` node: Set to run every 6 hours.

---

**Step 15: Read Sheet2 and Sheet1 for Updates**  
- Add two `Google Sheets` nodes:  
  - `Get row(s) in sheet1` reads Sheet2 (`gid=1153061999`)  
  - `Get row(s) in sheet3` reads Sheet1 (`gid=0`)  
- Connect `Schedule Trigger1` to both.

---

**Step 16: Fetch Latest GitHub Data for Sheet2 Bounties**  
- Add `GitHub Search HTTP Request1` node:  
  - URL template: Replace `https://github.com/` with `https://api.github.com/repos/` in `Issue Repo URL` + `/issues/` + `Issue Number`  
  - Auth: Bearer token  
- Connect `Get row(s) in sheet1` to this node.

---

**Step 17: Detect Changes in Sheet2 Data**  
- Add `Filter2` node:  
  - Conditions: Issue state or comment count differ from Sheet2 stored values  
- Connect `GitHub Search HTTP Request1` output to this node.

---

**Step 18: Update Changed Rows in Sheet2**  
- Add `Update row in sheet` node:  
  - Update operation on Sheet2 (`gid=1153061999`)  
  - Update fields: Issue state and Number of Issue Comments  
  - Use row_number from `Get row(s) in sheet1`  
- Connect `Filter2` output to this node.

---

**Step 19: Fetch Latest GitHub Data for Sheet1 Bounties**  
- Add `GitHub Search HTTP Request2` node:  
  - URL template same as step 16  
- Connect `Get row(s) in sheet3` to this node.

---

**Step 20: Detect Changes in Sheet1 Data**  
- Add `Filter3` node:  
  - Conditions: Issue state or updated_at timestamp differ from Sheet1 stored values  
- Connect `GitHub Search HTTP Request2` output to this node.

---

**Step 21: Update Changed Rows in Sheet1**  
- Add `Update row in sheet1` node:  
  - Update operation on Sheet1 (`gid=0`)  
  - Update fields: Issue state and Issue Updated Date  
  - Use row_number from `Get row(s) in sheet3`  
- Connect `Filter3` output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow manages bounties using two Google Sheets: Sheet1 (all bounties), Sheet2 (recent bounties for notifications).  | Sticky Note12                                                                                   |
| The GitHub API is queried with pagination to ensure all open bounty issues are retrieved.                                    | Sticky Note1                                                                                   |
| WhatsApp notification node is disabled and replaced by email alerts via Gmail.                                               | Sticky Note11                                                                                   |
| Email notifications use a custom GitHub-themed HTML template for readability and branding.                                   | Sticky Note5                                                                                   |
| Status update checks run every 6 hours to minimize API calls and keep data fresh.                                            | Sticky Note6                                                                                   |
| Update detection is based on issue state changes, comment count changes, and updated timestamps for accuracy.               | Sticky Notes8, 10                                                                             |
| GitHub API rate limits and authentication errors are potential failure points; ensure valid tokens and quota are maintained.| General best practice                                                                            |
| Google Sheets OAuth2 credentials must have write access to the target spreadsheet.                                           | General best practice                                                                            |

---

**Disclaimer:**  
The text above is solely derived from an automated n8n workflow designed for legal and public data management. No illegal or protected content is handled. All actions comply with respective API policies.