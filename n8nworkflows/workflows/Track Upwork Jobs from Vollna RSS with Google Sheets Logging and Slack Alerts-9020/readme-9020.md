Track Upwork Jobs from Vollna RSS with Google Sheets Logging and Slack Alerts

https://n8nworkflows.xyz/workflows/track-upwork-jobs-from-vollna-rss-with-google-sheets-logging-and-slack-alerts-9020


# Track Upwork Jobs from Vollna RSS with Google Sheets Logging and Slack Alerts

### 1. Workflow Overview

This workflow automates the tracking of Upwork job postings from a Vollna RSS feed. It continuously fetches new job listings, filters and processes them, detects duplicates against a Google Sheets log, appends unique jobs to the sheet, and sends Slack alerts and email confirmations for new jobs found. The workflow is designed for freelancers, recruiters, or agencies who want to monitor Upwork jobs efficiently and get notified of new opportunities.

The workflow logic is organized into these main blocks:

- **1.1 Trigger & Input Reception**: Scheduled regular polling of the Vollna RSS feed for Upwork jobs.
- **1.2 Data Cleaning & Filtering**: Filters out job postings with foreign characters in their titles to focus on English listings.
- **1.3 Sorting & Data Extraction**: Sorts jobs by recency, then extracts and formats detailed job data such as title, budget, posting time, description, skills, categories, and Upwork link.
- **1.4 Duplicate Detection**: Checks Google Sheets for existing entries with matching job links to avoid duplicates.
- **1.5 Logging New Jobs**: Appends new, unique jobs to the Google Sheet tracking document.
- **1.6 Notifications**: Sends Slack messages to a dedicated channel about new jobs and emails a confirmation message after Slack notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

- **Overview:**  
  This block triggers the workflow every 3 minutes and reads the latest Upwork jobs from the Vollna RSS feed.

- **Nodes Involved:**  
  - Schedule Trigger  
  - RSS Read

- **Node Details:**  

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates the workflow on a fixed interval (every 3 minutes).  
    - *Config:* Interval set in minutes: 3.  
    - *Connections:* Output → RSS Read  
    - *Potential Failures:* Scheduler misconfiguration; workflow disabled.  
    - *Version:* 1.2

  - **RSS Read**  
    - *Type:* RSS Feed Read  
    - *Role:* Fetches job listings from the specified Vollna RSS feed URL.  
    - *Config:* URL set to `https://www.vollna.com/rss/rftHMpSQCGeEfr2Zwjzb`, no custom fields configured.  
    - *Connections:* Input ← Schedule Trigger; Output → Check for foreign characters  
    - *Potential Failures:* RSS feed unavailable or malformed; network timeout.  
    - *Version:* 1.2

---

#### 1.2 Data Cleaning & Filtering

- **Overview:**  
  Filters job items to pass only those with ASCII (English) character titles, removing entries with foreign or special characters.

- **Nodes Involved:**  
  - Check for foreign characters (Filter)

- **Node Details:**  

  - **Check for foreign characters**  
    - *Type:* Filter  
    - *Role:* Filters incoming items by checking if the job title contains only ASCII characters (hex 00-7F).  
    - *Config:* Regex condition on the `title` field: `^[\x00-\x7F]+$` (strict, case-sensitive).  
    - *Connections:* Input ← RSS Read; Output → Sort  
    - *Edge Cases:* Titles missing or empty will fail the regex and be filtered out; non-string titles may cause expression errors.  
    - *Version:* 2.2

---

#### 1.3 Sorting & Data Extraction

- **Overview:**  
  Sorts the filtered jobs by most recent publication date and extracts key data fields including title, budget, Upwork link, posting time, job description, skills, and categories for each job.

- **Nodes Involved:**  
  - Sort  
  - Title, Budget, Link, Posted, Date, Job Description, Skills, Categories (Code)

- **Node Details:**  

  - **Sort**  
    - *Type:* Sort  
    - *Role:* Orders job items descending by `pubDate` to prioritize recent jobs first.  
    - *Config:* Sort field: `pubDate`, order: descending.  
    - *Connections:* Input ← Check for foreign characters; Output → Code node for extraction  
    - *Potential Failures:* Missing or invalid `pubDate` fields could affect sort order.  
    - *Version:* 1

  - **Title, Budget, Link, Posted, Date, Job Description, Skills, Categories**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses and formats each job item into structured fields.  
    - *Config Summary:*  
      - Splits `title` into job title and budget (if budget present in parentheses).  
      - Extracts Upwork job URL from link, decoding URL params if needed.  
      - Formats `pubDate` into human-readable relative posted time and date.  
      - Cleans content to extract job description, skills, and categories.  
    - *Key expressions:*  
      - Regex for title-budget: `/^(.*?)\s*\((.*?)\)$/`  
      - URL decoding with multi-level decoding function  
      - Date difference calculation for "Posted X ago" strings  
      - Content parsing for `Skills:` and `Categories:` lines  
    - *Connections:* Input ← Sort; Output → Parallel: Get row(s) in sheet and Compare Datasets (two outputs)  
    - *Edge Cases:* Missing or malformed fields, invalid URLs, empty content fields, dates that fail to parse.  
    - *Version:* 2

---

#### 1.4 Duplicate Detection

- **Overview:**  
  Checks if each extracted job already exists in the Google Sheet by matching the Upwork job link to avoid duplicate entries.

- **Nodes Involved:**  
  - Get row(s) in sheet  
  - Compare Datasets

- **Node Details:**  

  - **Get row(s) in sheet**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Looks up rows in the Google Sheet where the "UPWORK JOB LINK" column matches the current job's link.  
    - *Config:* Filter by "UPWORK JOB LINK" equals current item's `upwork_link` field.  
    - *Connections:* Input ← Code node; Output → Compare Datasets (as "existing" dataset)  
    - *Credentials:* Google Sheets OAuth2 account  
    - *Edge Cases:* API limits, sheet access issues, no matching rows returns empty dataset.  
    - *Version:* 4.7

  - **Compare Datasets**  
    - *Type:* Compare Datasets  
    - *Role:* Compares the new jobs dataset (from Code node) with existing rows from Google Sheets to detect new vs duplicates based on `upwork_link`.  
    - *Config:* Merge by fields: new `upwork_link` and sheet `UPWORK JOB LINK`.  
    - *Connections:* Input 1 ← Code node; Input 2 ← Get row(s) in sheet; Output (new items) → Append row in sheet  
    - *Edge Cases:* Mismatched or missing links, case sensitivity in URL strings.  
    - *Version:* 2.3

---

#### 1.5 Logging New Jobs

- **Overview:**  
  Appends newly identified unique job listings to the Google Sheet for record-keeping.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**  

  - **Append row in sheet**  
    - *Type:* Google Sheets (Append)  
    - *Role:* Adds new job data as rows in the Google Sheet with columns for date, title, budget, posted time, skills, categories, job description, and Upwork link.  
    - *Config:* Defines explicit column mapping to sheet columns by name.  
    - *Connections:* Input ← Compare Datasets (new unique jobs); Output → Send a message (Slack notification)  
    - *Credentials:* Google Sheets OAuth2 account  
    - *Edge Cases:* API limits, sheet permissions, schema mismatches.  
    - *Version:* 4.7

---

#### 1.6 Notifications

- **Overview:**  
  Notifies a Slack channel about new jobs and sends a confirmation email once the Slack message is sent.

- **Nodes Involved:**  
  - Send a message (Slack)  
  - Aggregate  
  - Send a message1 (Gmail)

- **Node Details:**  

  - **Send a message (Slack)**  
    - *Type:* Slack  
    - *Role:* Posts a formatted alert message about the new job to the `#n8n-jobs` Slack channel.  
    - *Config:*  
      - Message text includes: Title, Budget, Posted time, Upwork job link.  
      - Channel ID: `C09DV2HQ2M9` (n8n-jobs channel)  
      - Authentication: OAuth2 Slack account configured.  
    - *Connections:* Input ← Append row in sheet; Output → Aggregate  
    - *Edge Cases:* Slack API rate limits, invalid channel ID, auth token expiration.  
    - *Version:* 2.3

  - **Aggregate**  
    - *Type:* Aggregate  
    - *Role:* Aggregates messages to prepare for email confirmation (here it aggregates the `message` field).  
    - *Connections:* Input ← Slack message; Output → Send a message1 (Gmail)  
    - *Version:* 1

  - **Send a message1 (Gmail)**  
    - *Type:* Gmail  
    - *Role:* Sends a confirmation email indicating a Slack message was successfully sent.  
    - *Config:*  
      - To: `wright.jeremiah888@gmail.com`  
      - Subject: "Slack #n8n"  
      - Message: Simple confirmation text.  
      - OAuth2 Gmail credentials configured.  
    - *Connections:* Input ← Aggregate; Output → none  
    - *Edge Cases:* Gmail API limits, invalid credentials, email delivery failures.  
    - *Version:* 2.1

---

### 3. Summary Table

| Node Name                                              | Node Type           | Functional Role                        | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                   |
|--------------------------------------------------------|---------------------|-------------------------------------|----------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger                                       | Schedule Trigger    | Initiates workflow every 3 minutes  | None                             | RSS Read                            | Trigger & Input: Runs every 3 minutes; Fetches new jobs from Vollna RSS feed                  |
| RSS Read                                              | RSS Feed Read       | Reads Upwork jobs from RSS feed     | Schedule Trigger                 | Check for foreign characters         | Trigger & Input: Runs every 3 minutes; Fetches new jobs from Vollna RSS feed                  |
| Check for foreign characters                          | Filter              | Filters jobs with non-ASCII titles  | RSS Read                        | Sort                               | Filter Jobs: Only allow jobs with English (ASCII) titles; Prevents non-English jobs          |
| Sort                                                  | Sort                | Sorts jobs descending by pubDate    | Check for foreign characters    | Title, Budget, Link, Posted, Date, Job Description, Skills, Categories | Sort Jobs: Sorts jobs by most recent (descending by pubDate)                                |
| Title, Budget, Link, Posted, Date, Job Description, Skills, Categories | Code                | Extracts and formats job data       | Sort                           | Get row(s) in sheet, Compare Datasets     | Extract & Format Data: Splits title and budget; Extracts Upwork link; Formats posted date; Extracts job description, skills, and categories |
| Get row(s) in sheet                                  | Google Sheets (Read) | Looks for existing job link in sheet | Title, Budget, Link, Posted, Date, Job Description, Skills, Categories | Compare Datasets                     | Check for Duplicates: Looks up job link in Google Sheet; Compares to avoid duplicates         |
| Compare Datasets                                     | Compare Datasets    | Detects new vs existing jobs         | Title, Budget, Link, Posted, Date, Job Description, Skills, Categories, Get row(s) in sheet | Append row in sheet                  | Check for Duplicates: Looks up job link in Google Sheet; Compares to avoid duplicates         |
| Append row in sheet                                  | Google Sheets (Append) | Appends new unique jobs to sheet     | Compare Datasets                | Send a message (Slack)                | Save New Job: Appends new, unique jobs to Google Sheet                                       |
| Send a message (Slack)                              | Slack               | Sends Slack alert about new job      | Append row in sheet             | Aggregate                           | Slack Notification: Sends new job alert to #n8n-jobs Slack channel                           |
| Aggregate                                           | Aggregate           | Aggregates Slack messages for email  | Send a message (Slack)          | Send a message1 (Gmail)               | Email Confirmation: Sends email to confirm Slack message was sent                           |
| Send a message1 (Gmail)                             | Gmail               | Sends email confirmation             | Aggregate                      | None                              | Email Confirmation: Sends email to confirm Slack message was sent                           |
| Sticky Note                                         | Sticky Note         | Documentation notes                  | None                          | None                              | Trigger & Input: Runs every 3 minutes; Fetches new jobs from Vollna RSS feed                  |
| Sticky Note1                                        | Sticky Note         | Documentation notes                  | None                          | None                              | Sorting: Sorts jobs by most recent (descending by pubDate); Extract & Format Data: parsing   |
| Sticky Note3                                        | Sticky Note         | Documentation notes                  | None                          | None                              | Duplicate Check: Looks up job link in Google Sheet; Append to Sheet: Save new jobs           |
| Sticky Note4                                        | Sticky Note         | Documentation notes                  | None                          | None                              | Notification: Slack Notification and optional Email Confirmation                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Parameters: Interval → Every 3 minutes  
   - Connect output to the next node.

2. **Add RSS Feed Read node**  
   - Type: RSS Feed Read  
   - Parameters: URL → `https://www.vollna.com/rss/rftHMpSQCGeEfr2Zwjzb`  
   - Connect input to Schedule Trigger output.

3. **Add Filter node: "Check for foreign characters"**  
   - Type: Filter  
   - Condition: `{{ $json.title }}` matches regex `^[\x00-\x7F]+$` (case-sensitive, strict)  
   - Connect input to RSS Feed Read output.

4. **Add Sort node**  
   - Type: Sort  
   - Sort field: `pubDate` descending  
   - Connect input to Filter node output.

5. **Add Code node: "Title, Budget, Link, Posted, Date, Job Description, Skills, Categories"**  
   - Type: Function  
   - Paste provided JavaScript code that:  
     - Splits title and budget from `title`  
     - Extracts and decodes Upwork job link from `link`  
     - Formats posted time and date from `pubDate`  
     - Extracts job description, skills, categories from `content`  
   - Connect input to Sort node output.

6. **Add Google Sheets node: "Get row(s) in sheet"**  
   - Type: Google Sheets (Read)  
   - Document ID → Your Google Sheet ID  
   - Sheet Name → `Sheet1` (or your sheet's name)  
   - Filter: Lookup column `UPWORK JOB LINK`, lookup value `={{ $json.upwork_link }}`  
   - Connect input to Code node output.

7. **Add Compare Datasets node**  
   - Type: Compare Datasets  
   - Merge by fields: `upwork_link` (from Code node) and `UPWORK JOB LINK` (from Google Sheets)  
   - Connect input 1 to Code node output (new data)  
   - Connect input 2 to Google Sheets read node output (existing data).

8. **Add Google Sheets node: "Append row in sheet"**  
   - Type: Google Sheets (Append)  
   - Document ID and Sheet Name same as "Get row(s) in sheet"  
   - Mapping Mode: Define columns explicitly:  
     - DATE → `={{ $json.date }}`  
     - TITLE → `={{ $json.title }}`  
     - BUDGET → `={{ $json.budget }}`  
     - POSTED → `={{ $json.posted }}`  
     - SKILLS → `={{ $json.skills }}`  
     - CATEGORIES → `={{ $json.categories }}`  
     - JOB DESCRIPTION → `={{ $json.job_description }}`  
     - UPWORK JOB LINK → `={{ $json.upwork_link }}`  
   - Connect input to Compare Datasets output (new unique jobs).

9. **Add Slack node: "Send a message"**  
   - Type: Slack  
   - OAuth2 credentials required (Slack app with chat:write permission)  
   - Channel ID: your channel ID (e.g., `C09DV2HQ2M9` for #n8n-jobs)  
   - Message text:  
     ```
     New Job Alert :fire: :fire:

     {{ $json.TITLE }} - {{ $json.BUDGET }}

     {{ $json.POSTED }}

     {{ $json['UPWORK JOB LINK'] }}
     ```  
   - Connect input to Append row in sheet output.

10. **Add Aggregate node**  
    - Type: Aggregate  
    - Aggregate field: `message` (default)  
    - Connect input to Slack node output.

11. **Add Gmail node: "Send a message1"**  
    - Type: Gmail  
    - OAuth2 credentials required for Gmail account  
    - Send To: `wright.jeremiah888@gmail.com`  
    - Subject: `Slack #n8n`  
    - Message: `✅ A message was just sent to #n8n-jobs!`  
    - Connect input to Aggregate node output.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow runs every 3 minutes to ensure near real-time job tracking.                                              | Trigger & Input block                                                                            |
| Filtering out non-ASCII job titles helps focus on English-language jobs and avoids data noise.                    | Data Cleaning & Filtering block                                                                 |
| Sorting by publication date ensures the newest jobs get processed first.                                          | Sorting block                                                                                   |
| Detailed data extraction from content fields improves usability and downstream processing.                        | Data Extraction & Formatting block                                                              |
| Duplicate detection against Google Sheets prevents redundant entries and notifications.                           | Duplicate Detection block                                                                       |
| Slack channel notifications enable real-time alerts to team members or individuals monitoring jobs.              | Notifications block                                                                             |
| Email confirmation provides audit trail or fallback notification that Slack messages are sent.                    | Notifications block                                                                             |
| Google Sheets document ID and Slack channel ID must be updated according to user’s actual environment.           | Credential and environment-specific setup                                                      |
| Slack OAuth2 and Gmail OAuth2 credentials require appropriate API scopes and token refresh setup in n8n.          | Credential configuration                                                                        |
| The Vollna RSS feed URL is specific to Upwork jobs and may need updating if the source changes or breaks.        | Input RSS Feed URL                                                                              |
| The workflow is easily extendable for other job boards or notification channels by modifying or adding nodes.     | Workflow extensibility                                                                          |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.