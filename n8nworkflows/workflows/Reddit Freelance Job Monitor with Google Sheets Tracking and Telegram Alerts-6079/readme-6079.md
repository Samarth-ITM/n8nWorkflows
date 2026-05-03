Reddit Freelance Job Monitor with Google Sheets Tracking and Telegram Alerts

https://n8nworkflows.xyz/workflows/reddit-freelance-job-monitor-with-google-sheets-tracking-and-telegram-alerts-6079


# Reddit Freelance Job Monitor with Google Sheets Tracking and Telegram Alerts

### 1. Workflow Overview

This workflow automates the monitoring of freelance job postings on Reddit, specifically filtering for paid opportunities, tracking them in a Google Sheets document, and sending Telegram alerts to notify the user of new relevant posts. It is designed for freelance professionals or agencies who want to efficiently discover and track paid freelance jobs from Reddit without manual searching.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger**: Periodically initiates the workflow every 5 minutes to ensure up-to-date monitoring.
- **1.2 Reddit Data Retrieval and Preparation**: Fetches recent freelance job posts from Reddit, extracts metadata, and splits the data into individual posts.
- **1.3 Filtering Paid Jobs**: Filters posts to keep only those that specify paid requirements.
- **1.4 Duplicate Detection and Data Validation**: Checks existing records in Google Sheets to avoid duplicate alerts and entries, identifying unique new posts.
- **1.5 Data Enrichment and Storage**: Converts post timestamps to UTC, saves unique posts to Google Sheets, and finally,
- **1.6 User Notification**: Sends Telegram alerts about the new freelance job listings.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow automatically every 5 minutes to ensure continual monitoring of Reddit for new freelance job posts.

- **Nodes Involved:**  
  - Run Every 5 Minutes

- **Node Details:**  
  - **Run Every 5 Minutes**  
    - **Type & Role:** Schedule Trigger node; initiates workflow on a fixed time interval.  
    - **Configuration:** Default setup with an interval of 5 minutes; no additional parameters configured.  
    - **Input/Output:** No input; output triggers the next node.  
    - **Edge Cases:** Workflow will not run if n8n instance is offline or if schedules are disabled. No authentication is needed here.  
    - **Version:** v1.2 (stable for scheduled triggers).

#### 1.2 Reddit Data Retrieval and Preparation

- **Overview:**  
  Fetches the latest posts from Reddit related to freelance jobs, extracts metadata for easier downstream processing, and splits the array of posts into individual items.

- **Nodes Involved:**  
  - Look for freelance requirement posts  
  - Extract Post Metadata  
  - Seperate array into individual posts

- **Node Details:**  
  - **Look for freelance requirement posts**  
    - **Type & Role:** HTTP Request node; sends an HTTP request to Reddit’s API or a relevant endpoint to retrieve posts.  
    - **Configuration:** No explicit parameters shown, but typically configured with Reddit API endpoint URL, method (GET), and appropriate headers or query parameters to search for freelance job posts.  
    - **Input/Output:** Triggered by schedule; outputs JSON array of posts.  
    - **Edge Cases:** Possible API rate limits, network timeouts, or empty responses. Requires valid Reddit API access or public endpoint availability.  
    - **Version:** 4.2  
  - **Extract Post Metadata**  
    - **Type & Role:** Set node; transforms and extracts relevant metadata fields (e.g., post title, author, URL, payment information) into structured format.  
    - **Configuration:** Likely configured with expressions to map fields from raw Reddit response to custom fields for filtering and storage.  
    - **Input/Output:** Receives posts array; outputs structured metadata array.  
    - **Edge Cases:** Expression errors if expected fields are missing or malformed in Reddit response.  
    - **Version:** 3.4  
  - **Seperate array into individual posts**  
    - **Type & Role:** SplitOut node; converts the array of posts into individual items for filtering and processing.  
    - **Configuration:** Default splitting of array items into separate workflow items.  
    - **Input/Output:** Input is array of posts; outputs one post per item.  
    - **Edge Cases:** Empty arrays result in no output items; no failure expected.  
    - **Version:** 1  

#### 1.3 Filtering Paid Jobs

- **Overview:**  
  Filters the individual posts to retain only those that indicate paid freelance requirements.

- **Nodes Involved:**  
  - Make sure the requirement is paid

- **Node Details:**  
  - **Make sure the requirement is paid**  
    - **Type & Role:** Filter node; evaluates each post’s metadata to check if it specifies payment.  
    - **Configuration:** Conditions might check for presence of keywords like “paid”, “budget”, or specific post field values indicating compensation.  
    - **Input/Output:** Input from split posts; outputs only posts passing the filter.  
    - **Edge Cases:** Posts without clear payment info get excluded; false negatives possible if payment info is implicit or in unusual format.  
    - **Version:** 2.2  

#### 1.4 Duplicate Detection and Data Validation

- **Overview:**  
  Checks existing Google Sheets rows to detect already processed posts, then filters out duplicates to keep only unique new posts for saving and notification.

- **Nodes Involved:**  
  - Check present Rows  
  - Filter Unique Posts

- **Node Details:**  
  - **Check present Rows**  
    - **Type & Role:** Google Sheets node; reads current rows from a specified sheet to retrieve previously saved job posts.  
    - **Configuration:** Configured with spreadsheet ID, sheet name, read range, and credentials for Google Sheets OAuth2 authentication.  
    - **Input/Output:** Receives filtered paid posts; outputs current sheet data for comparison.  
    - **Edge Cases:** Authentication errors, API quota limits, empty or corrupted sheets.  
    - **Version:** 4.6  
  - **Filter Unique Posts**  
    - **Type & Role:** Code node; runs custom JavaScript to compare new posts with existing rows, filtering out duplicates based on post ID or URL.  
    - **Configuration:** Contains code logic to iterate over new posts and existing data to detect uniqueness.  
    - **Input/Output:** Input from Google Sheets data and filtered posts; outputs only unique items.  
    - **Edge Cases:** Code errors if data format mismatches, missing keys, or empty inputs.  
    - **Version:** 2  

#### 1.5 Data Enrichment and Storage

- **Overview:**  
  Converts post timestamps to UTC format, saves unique posts into Google Sheets, and prepares data for notification.

- **Nodes Involved:**  
  - Get UTC of Post  
  - Save in sheets

- **Node Details:**  
  - **Get UTC of Post**  
    - **Type & Role:** Set node; transforms the post date/time to standardized UTC.  
    - **Configuration:** Uses expressions or functions to convert post timestamp fields to UTC format.  
    - **Input/Output:** Input unique posts; output enriched posts with UTC timestamps.  
    - **Edge Cases:** Invalid or missing date fields may cause conversion errors.  
    - **Version:** 3.4  
  - **Save in sheets**  
    - **Type & Role:** Google Sheets node; appends new unique posts with enriched data to the tracking sheet.  
    - **Configuration:** Spreadsheet and sheet specified, append mode enabled, credentials configured.  
    - **Input/Output:** Input enriched posts; outputs success confirmation or next nodes.  
    - **Edge Cases:** API quota exceeded, write conflicts, or authentication failure.  
    - **Version:** 4.6  

#### 1.6 User Notification

- **Overview:**  
  Sends Telegram messages to a user or group informing them of new freelance job posts that have been recorded.

- **Nodes Involved:**  
  - Inform User in telegram

- **Node Details:**  
  - **Inform User in telegram**  
    - **Type & Role:** Telegram node; sends text messages with post details.  
    - **Configuration:** Uses Telegram Bot credentials, configured with chat ID, message template (likely including post title, URL, payment info).  
    - **Input/Output:** Input from Google Sheets save confirmation; outputs success or failure status.  
    - **Edge Cases:** Telegram API rate limits, invalid chat IDs, bot permissions restrictions.  
    - **Version:** 1.2  

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                          | Input Node(s)                        | Output Node(s)                    | Sticky Note                                      |
|-------------------------------|-----------------------|----------------------------------------|------------------------------------|----------------------------------|-------------------------------------------------|
| Run Every 5 Minutes            | Schedule Trigger      | Periodic trigger every 5 minutes       | -                                  | Look for freelance requirement posts |                                                 |
| Look for freelance requirement posts | HTTP Request         | Retrieve freelance job posts from Reddit | Run Every 5 Minutes                 | Extract Post Metadata             |                                                 |
| Extract Post Metadata          | Set                   | Extract and structure post metadata    | Look for freelance requirement posts | Seperate array into individual posts |                                                 |
| Seperate array into individual posts | SplitOut              | Split posts array into single items    | Extract Post Metadata               | Make sure the requirement is paid |                                                 |
| Make sure the requirement is paid | Filter                | Filter posts to keep only paid jobs    | Seperate array into individual posts | Check present Rows                |                                                 |
| Check present Rows             | Google Sheets          | Read existing job posts from sheet     | Make sure the requirement is paid  | Filter Unique Posts              |                                                 |
| Filter Unique Posts            | Code                   | Filter out duplicates from new posts   | Check present Rows                  | Get UTC of Post                  |                                                 |
| Get UTC of Post                | Set                   | Convert post timestamps to UTC         | Filter Unique Posts                 | Save in sheets                   |                                                 |
| Save in sheets                | Google Sheets          | Save new unique posts to sheet          | Get UTC of Post                    | Inform User in telegram          |                                                 |
| Inform User in telegram        | Telegram               | Notify user of new job posts            | Save in sheets                     | -                                |                                                 |
| Sticky Note                   | Sticky Note            | -                                      | -                                  | -                                |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run every 5 minutes (default interval).  
   - Name it "Run Every 5 Minutes".

2. **Add an HTTP Request node**  
   - Name: "Look for freelance requirement posts"  
   - Configure with Reddit API endpoint to fetch posts from relevant subreddits or search queries for freelance jobs.  
   - Method: GET  
   - Add required headers or authentication if needed (e.g., Reddit OAuth2 credentials).  
   - Connect "Run Every 5 Minutes" output to this node’s input.

3. **Add a Set node**  
   - Name: "Extract Post Metadata"  
   - Configure to map and extract necessary metadata fields from Reddit’s JSON response, such as post ID, title, author, URL, and payment info. Use expressions referencing the HTTP Request output.  
   - Connect "Look for freelance requirement posts" output to this node.

4. **Add a SplitOut node**  
   - Name: "Seperate array into individual posts"  
   - Configure to split the array of posts into individual items (default settings).  
   - Connect "Extract Post Metadata" output to this node.

5. **Add a Filter node**  
   - Name: "Make sure the requirement is paid"  
   - Configure filter rules to check post metadata fields for indicators of payment (e.g., text contains “paid”, “budget”, or numeric budget field > 0).  
   - Connect "Seperate array into individual posts" output to this node.

6. **Add a Google Sheets node**  
   - Name: "Check present Rows"  
   - Set operation to "Read Rows".  
   - Configure with Google Sheets credentials (OAuth2).  
   - Specify Spreadsheet ID and Sheet Name where job posts are tracked.  
   - Configure to read relevant columns for duplicate detection (e.g., post ID or URL).  
   - Connect "Make sure the requirement is paid" output to this node.  
   - Enable “Execute Once” to prevent multiple identical reads per execution.

7. **Add a Code node**  
   - Name: "Filter Unique Posts"  
   - Write JavaScript to compare incoming posts against existing Google Sheets rows to identify and output only unique posts.  
   - Connect "Check present Rows" output to this node.

8. **Add a Set node**  
   - Name: "Get UTC of Post"  
   - Configure to convert post timestamps from Reddit into UTC format using expressions or JavaScript date functions.  
   - Connect "Filter Unique Posts" output to this node.

9. **Add another Google Sheets node**  
   - Name: "Save in sheets"  
   - Set operation to "Append Rows".  
   - Use the same Spreadsheet and Sheet as before.  
   - Append new unique posts with enriched data (including UTC timestamp).  
   - Connect "Get UTC of Post" output to this node.

10. **Add a Telegram node**  
    - Name: "Inform User in telegram"  
    - Configure with Telegram Bot API credentials.  
    - Specify chat ID (user or group) to receive alerts.  
    - Configure message text template to include post title, URL, and payment info.  
    - Connect "Save in sheets" output to this node.

11. **Test the workflow end-to-end**  
    - Manually trigger or wait for schedule.  
    - Verify Reddit posts are fetched, filtered, de-duplicated, saved, and Telegram messages are sent.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                 |
|-----------------------------------------------------------------------------------------------|------------------------------------------------|
| The workflow runs every 5 minutes to catch new freelance job postings promptly.                | Scheduling best practice for near real-time monitoring |
| Google Sheets nodes require OAuth2 credentials authorized for the target spreadsheet.          | Google Sheets API documentation and OAuth2 setup |
| Telegram Bot must have permission to send messages to the configured chat ID or group.         | Telegram Bot API documentation                  |
| Potential Reddit API rate limits may require adjustments or retry logic in HTTP Request node.  | https://www.reddit.com/dev/api/                 |
| For filtering paid jobs, custom expressions or code may need tuning for post format changes.   | Keep filter logic up to date with Reddit post styles |
| This workflow is suitable for freelancers and agencies wanting to automate job discovery.      |                                                 |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.