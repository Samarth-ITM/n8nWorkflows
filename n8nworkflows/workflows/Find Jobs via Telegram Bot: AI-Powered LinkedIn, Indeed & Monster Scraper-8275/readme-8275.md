Find Jobs via Telegram Bot: AI-Powered LinkedIn, Indeed & Monster Scraper

https://n8nworkflows.xyz/workflows/find-jobs-via-telegram-bot--ai-powered-linkedin--indeed---monster-scraper-8275


# Find Jobs via Telegram Bot: AI-Powered LinkedIn, Indeed & Monster Scraper

### 1. Workflow Overview

This workflow implements a Telegram Bot designed to help users find job opportunities by scraping multiple job platforms—LinkedIn, Indeed, and Monster—using AI and data scraping APIs. The bot listens to Telegram messages, interprets user commands for job searches, queries external job data sources, processes and filters the job listings, formats results, and sends them back to the user on Telegram. Additionally, it saves job results to Google Sheets and Airtable for record-keeping and logs usage analytics to an external webhook.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Input Reception & Command Routing**  
  Listens for Telegram messages and filters them by commands to trigger appropriate responses.

- **1.2 User Interaction & Command Parsing**  
  Sends welcome/help messages and parses the `/jobs` command for keyword and location parameters.

- **1.3 Job Search Status Notification**  
  Notifies the user that the job search is in progress.

- **1.4 Job Platforms Scraping**  
  Sends requests to LinkedIn, Indeed, and Monster job scraping APIs with search parameters.

- **1.5 Job Data Processing & Filtering**  
  Normalizes and filters raw job data, annotates with relevance, experience, and remote options, and limits results.

- **1.6 Formatting Results & Telegram Response**  
  Formats the filtered jobs into a markdown message suitable for Telegram and sends it to the user.

- **1.7 Data Persistence & Analytics Logging**  
  Saves job results to Google Sheets and Airtable and sends usage data to an analytics webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception & Command Routing

- **Overview:**  
  The workflow starts by listening to all Telegram messages through a Telegram Bot trigger, then filters incoming messages based on the command to either welcome the user or initiate a job search.

- **Nodes Involved:**  
  - Telegram Bot Trigger  
  - Command Filter  
  - Job Search Filter  
  - Send Welcome Message

- **Node Details:**  

  - **Telegram Bot Trigger**  
    - Type: `telegramTrigger`  
    - Role: Entry point node that listens to Telegram updates, specifically incoming messages.  
    - Config: Listens for "message" update type.  
    - Connections: Outputs to Command Filter and Job Search Filter nodes.  
    - Potential Failures: Telegram API connection issues or missing credentials.  
    - Credential: Telegram Bot API (OAuth token).

  - **Command Filter**  
    - Type: `filter`  
    - Role: Filters messages with exact text `/start` to trigger the welcome message.  
    - Config: Case-sensitive string equals condition against message text.  
    - Inputs: Telegram Bot Trigger  
    - Outputs: Send Welcome Message  
    - Edge Cases: Commands with extra spaces or different casing will not match.

  - **Job Search Filter**  
    - Type: `filter`  
    - Role: Filters messages starting with `/jobs` (case-insensitive) to trigger job search flow.  
    - Config: StartsWith condition on message text.  
    - Inputs: Telegram Bot Trigger  
    - Outputs: Parse Job Command  
    - Edge Cases: Messages not starting exactly with `/jobs` will be ignored.

  - **Send Welcome Message**  
    - Type: `telegram`  
    - Role: Sends a markdown-formatted welcome message with usage instructions to the user.  
    - Config: Uses the chat ID from the incoming message, markdown parse mode enabled.  
    - Inputs: Command Filter  
    - Potential Failures: Telegram API send message errors.

---

#### 2.2 User Interaction & Command Parsing

- **Overview:**  
  Parses the `/jobs` command to extract the keyword(s) and location for the job search, providing defaults if parameters are omitted.

- **Nodes Involved:**  
  - Parse Job Command

- **Node Details:**  

  - **Parse Job Command**  
    - Type: `code` (JavaScript)  
    - Role: Parses the Telegram message text to extract job search keywords and location parameters.  
    - Config:  
      - Defaults keyword to "sales marketing" and location to "New York".  
      - Handles `/jobs keyword location` and `/jobs keyword` formats.  
      - Outputs an object including parsed keyword, location, chat/user info, timestamp, and a search query string.  
    - Inputs: Job Search Filter  
    - Outputs: Send Search Status  
    - Edge Cases:  
      - Commands with no parameters revert to defaults.  
      - Multiple-word keywords are supported.  
      - Assumes last word is location if two or more parts exist.  
      - Does not validate location correctness or sanitize inputs.

---

#### 2.3 Job Search Status Notification

- **Overview:**  
  Sends a message to Telegram informing the user that the job search is underway with the parsed parameters.

- **Nodes Involved:**  
  - Send Search Status

- **Node Details:**  

  - **Send Search Status**  
    - Type: `telegram`  
    - Role: Sends a markdown message displaying the search keyword and location.  
    - Config: Uses chatId from parsed command node, disables web preview, markdown parse mode.  
    - Inputs: Parse Job Command  
    - Outputs: LinkedIn, Indeed, and Monster Scraper nodes  
    - Edge Cases: Failures in Telegram API sending message.

---

#### 2.4 Job Platforms Scraping

- **Overview:**  
  Sends API requests to three job platforms (LinkedIn, Indeed, Monster) to retrieve job listings matching the user's parameters.

- **Nodes Involved:**  
  - LinkedIn Jobs Scraper  
  - Indeed Jobs Scraper  
  - Monster Jobs Scraper

- **Node Details:**  

  - **LinkedIn Jobs Scraper**  
    - Type: `httpRequest`  
    - Role: Posts a request to Bright Data API triggering LinkedIn job dataset scrape.  
    - Config:  
      - POST to `https://api.brightdata.com/datasets/v3/trigger`, with dataset ID for LinkedIn.  
      - Body includes keyword and location from inputs.  
      - Generic HTTP header authentication (likely API key).  
      - Retry enabled (max 3 tries, 2 sec wait), timeout 30 sec.  
    - Inputs: Send Search Status  
    - Outputs: Process Jobs for Telegram  
    - Edge Cases: API failures, rate limits, network timeout, invalid API keys.

  - **Indeed Jobs Scraper**  
    - Type: `httpRequest`  
    - Role: Similar to LinkedIn scraper but targeting Indeed job dataset.  
    - Config: POST to Bright Data API with dataset ID for Indeed, sends keyword and location as "what" and "where".  
    - Same retry and timeout settings.  
    - Inputs: Send Search Status  
    - Outputs: Process Jobs for Telegram  
    - Edge Cases: Same as above.

  - **Monster Jobs Scraper**  
    - Type: `httpRequest`  
    - Role: GET request to Monster's public job search API endpoint.  
    - Config: Query parameters include `q` (keyword), `where` (location), and `page=1`.  
    - Retry and timeout configured similarly.  
    - Inputs: Send Search Status  
    - Outputs: Process Jobs for Telegram  
    - Edge Cases: API changes, request limits, malformed query parameters.

---

#### 2.5 Job Data Processing & Filtering

- **Overview:**  
  Receives raw job data from scraping nodes, normalizes fields into a unified structure, filters for relevance, deduplicates, annotates experience level and remote options, and limits results to a maximum number.

- **Nodes Involved:**  
  - Process Jobs for Telegram

- **Node Details:**  

  - **Process Jobs for Telegram**  
    - Type: `code` (JavaScript)  
    - Role:  
      - Iterates through job results from multiple sources.  
      - Identifies the platform by presence of specific fields.  
      - Constructs a standardized job object with fields: platform, job_id, title, company, location, description, salary, URL, posted date, scrape timestamp, Telegram context, relevance flag, experience level, remote option, salary range.  
      - Relevance determined by matching keywords related to sales/marketing/business roles.  
      - Experience level extracted by keyword matching senior/entry-level indicators.  
      - Remote option detected by presence of keywords like "remote" or "work from home" in description.  
      - Salary is cleaned and formatted.  
      - Filters out irrelevant jobs, deduplicates by title-company key, limits to 10 jobs.  
      - Prepends a summary object with statistics if jobs exist.  
    - Inputs: LinkedIn, Indeed, Monster Scraper nodes  
    - Outputs: Format Jobs Message, Save to Google Sheets, Save to Airtable  
    - Edge Cases:  
      - Missing or inconsistent data fields from APIs.  
      - Parsing errors in text processing.  
      - Empty results lead to no jobs found scenario downstream.

---

#### 2.6 Formatting Results & Telegram Response

- **Overview:**  
  Formats the filtered job results into a human-readable markdown message tailored for Telegram, including job details and summary, then sends the message to the user.

- **Nodes Involved:**  
  - Format Jobs Message  
  - Send Job Results

- **Node Details:**  

  - **Format Jobs Message**  
    - Type: `code` (JavaScript)  
    - Role:  
      - Separates the summary object from job listings.  
      - If no jobs, creates a "No Jobs Found" message.  
      - Otherwise, builds a markdown message with total jobs found, platforms searched, count of remote jobs, and up to 5 job listings.  
      - Each job shows platform icon, title, company, location, salary if available, remote option, experience level, and a clickable "Apply Now" link.  
      - Adds a footer with instructions for new search and note about data saving.  
    - Inputs: Process Jobs for Telegram  
    - Outputs: Send Job Results, Log Usage Analytics  
    - Edge Cases: Empty job list, malformed URLs, markdown special characters in job info.

  - **Send Job Results**  
    - Type: `telegram`  
    - Role: Sends the formatted markdown message to the Telegram chat.  
    - Config: Disables web page preview, uses chat ID from formatted message.  
    - Inputs: Format Jobs Message  
    - Edge Cases: Telegram send message failures, message too long (Telegram limits).

---

#### 2.7 Data Persistence & Analytics Logging

- **Overview:**  
  Saves the processed job data to Google Sheets and Airtable for tracking and logs search usage data to an external analytics webhook.

- **Nodes Involved:**  
  - Save to Google Sheets  
  - Save to Airtable  
  - Log Usage Analytics

- **Node Details:**  

  - **Save to Google Sheets**  
    - Type: `googleSheets`  
    - Role: Appends or updates job records in a specified Google Sheets document and sheet.  
    - Config:  
      - Uses OAuth2 authentication.  
      - Matches on `job_id` column to avoid duplicates.  
      - Sheet name: "Telegram_Jobs".  
      - Document ID placeholder "your_google_sheet_id" must be replaced with actual ID.  
    - Inputs: Process Jobs for Telegram  
    - Edge Cases: Authentication failures, API quota exceeded, invalid document ID.

  - **Save to Airtable**  
    - Type: `airtable`  
    - Role: Creates new records in an Airtable base and table dedicated to storing Telegram job data.  
    - Config:  
      - Base ID and table ID are placeholders and must be configured.  
      - Bulk size 10, ignores missing columns to prevent errors.  
      - Uses Airtable token authentication.  
    - Inputs: Process Jobs for Telegram  
    - Edge Cases: Invalid base or table ID, authentication failure, rate limits.

  - **Log Usage Analytics**  
    - Type: `httpRequest`  
    - Role: Sends a JSON payload with search usage data to a Zapier webhook for analytics tracking.  
    - Config:  
      - POST method, 10-second timeout.  
      - JSON body includes user ID, chat ID, search query, job count, timestamp, platform, and workflow identifier.  
      - Webhook URL placeholder to be replaced.  
    - Inputs: Format Jobs Message  
    - Edge Cases: Webhook unavailability, network errors.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                          | Input Node(s)            | Output Node(s)                               | Sticky Note                                            |
|-------------------------|--------------------|----------------------------------------|--------------------------|----------------------------------------------|--------------------------------------------------------|
| Telegram Bot Trigger     | telegramTrigger    | Listen for Telegram messages            | —                        | Command Filter, Job Search Filter             |                                                        |
| Command Filter          | filter             | Filter `/start` command                  | Telegram Bot Trigger      | Send Welcome Message                          |                                                        |
| Job Search Filter       | filter             | Filter `/jobs` command                   | Telegram Bot Trigger      | Parse Job Command                             |                                                        |
| Send Welcome Message    | telegram           | Send welcome/help message                | Command Filter            | —                                            |                                                        |
| Parse Job Command       | code               | Parse `/jobs` command for parameters    | Job Search Filter         | Send Search Status                            |                                                        |
| Send Search Status      | telegram           | Notify user that search started          | Parse Job Command         | LinkedIn, Indeed, Monster Jobs Scraper        |                                                        |
| LinkedIn Jobs Scraper   | httpRequest        | Scrape LinkedIn job listings             | Send Search Status        | Process Jobs for Telegram                     |                                                        |
| Indeed Jobs Scraper     | httpRequest        | Scrape Indeed job listings               | Send Search Status        | Process Jobs for Telegram                     |                                                        |
| Monster Jobs Scraper    | httpRequest        | Scrape Monster job listings              | Send Search Status        | Process Jobs for Telegram                     |                                                        |
| Process Jobs for Telegram| code               | Normalize, filter, deduplicate jobs      | LinkedIn, Indeed, Monster Scraper | Format Jobs Message, Save to Google Sheets, Save to Airtable |                                                        |
| Format Jobs Message     | code               | Format jobs list into Telegram message   | Process Jobs for Telegram | Send Job Results, Log Usage Analytics         |                                                        |
| Send Job Results        | telegram           | Send formatted job results message       | Format Jobs Message       | —                                            |                                                        |
| Save to Google Sheets   | googleSheets       | Save jobs to Google Sheets                | Process Jobs for Telegram | —                                            |                                                        |
| Save to Airtable        | airtable           | Save jobs to Airtable                     | Process Jobs for Telegram | —                                            |                                                        |
| Log Usage Analytics     | httpRequest        | Send usage analytics to webhook           | Format Jobs Message       | —                                            |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot Trigger node:**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates only.  
   - Set credentials with Telegram Bot API token ("Job Scraper Bot").

2. **Add two Filter nodes:**  
   - **Command Filter:**  
     - Filter for message text exactly equals `/start` (case sensitive).  
     - Connect input from Telegram Bot Trigger.  
   - **Job Search Filter:**  
     - Filter for message text starting with `/jobs` (case insensitive).  
     - Connect input from Telegram Bot Trigger.

3. **Add Telegram node for Welcome Message:**  
   - Connect from Command Filter output.  
   - Configure message text with markdown instructions and examples.  
   - Use chat ID from incoming message JSON.  
   - Use same Telegram Bot API credentials.

4. **Add Code node to parse `/jobs` command:**  
   - Connect from Job Search Filter output.  
   - Paste JavaScript code to extract keyword and location from the message text with defaults.  
   - Output JSON must include keyword, location, chatId, userId, userName, timestamp, original message, and a search query string.

5. **Add Telegram node to send Search Status message:**  
   - Connect from Parse Job Command output.  
   - Send a markdown message showing keyword and location with "Searching for Jobs..." text.  
   - Use chat ID from the parsed command node.

6. **Add three HTTP Request nodes for job scraping APIs:**  
   - Connect all three nodes from Send Search Status output.  
   - **LinkedIn Jobs Scraper:**  
     - POST to Bright Data API (dataset_id for LinkedIn).  
     - Include `keyword` and `location` from input JSON.  
     - Configure generic header authentication (API key).  
     - Retry enabled, 3 attempts, 2s wait, 30s timeout.  
   - **Indeed Jobs Scraper:**  
     - Similar to LinkedIn but different dataset ID.  
     - Use parameters `what` (keyword), `where` (location).  
   - **Monster Jobs Scraper:**  
     - GET request to Monster API endpoint.  
     - Query parameters: `q` (keyword), `where` (location), `page=1`.  
     - Same retry and timeout.

7. **Add Code node for Job Processing:**  
   - Connect inputs from all three scraper nodes.  
   - Paste JavaScript code that normalizes job data, filters relevant jobs by keyword, deduplicates by title/company, annotates experience and remote status, formats salary.  
   - Limit jobs to 10 max.  
   - Add a summary object with search statistics prepended.

8. **Add Code node to Format Jobs Message:**  
   - Connect from Job Processing node.  
   - Paste code to build a markdown message with summary, up to 5 jobs formatted with platform icons, company, location, salary, remote flag, experience level, and apply links.  
   - Handle no jobs found case with a fallback message.

9. **Add Telegram node to Send Job Results:**  
   - Connect from Format Jobs Message.  
   - Send message text from formatted message JSON.  
   - Use chat ID from formatted message.  
   - Disable web page preview, enable markdown.

10. **Add Google Sheets node to Save Jobs:**  
    - Connect from Job Processing node.  
    - Use operation `appendOrUpdate` on a sheet named "Telegram_Jobs".  
    - Provide Google Sheets document ID and OAuth2 credentials.  
    - Match on `job_id` column.

11. **Add Airtable node to Save Jobs:**  
    - Connect from Job Processing node.  
    - Use `create` operation on configured base ID and table.  
    - Use Airtable token authentication.  
    - Allow bulk size 10 and ignore missing columns.

12. **Add HTTP Request node for Usage Analytics:**  
    - Connect from Format Jobs Message.  
    - POST JSON data with user, chat, search query, jobs found, timestamp, platform, and workflow info to your Zapier webhook URL.  
    - Set 10-second timeout.

13. **Set all connections according to the JSON workflow:**  
    - Telegram Bot Trigger outputs to Command Filter and Job Search Filter.  
    - Filters connect to their respective next nodes.  
    - Parse Job Command connects to Send Search Status.  
    - Send Search Status fans out to all three scraper nodes.  
    - All scraper nodes connect to Process Jobs.  
    - Process Jobs connects to Format Jobs Message, Google Sheets, and Airtable nodes.  
    - Format Jobs Message connects to Send Job Results and Usage Analytics.

14. **Verify credentials:**  
    - Telegram Bot API credentials with correct bot token.  
    - Bright Data API keys or equivalent generic HTTP header auth for scrapers.  
    - Google OAuth2 credentials with access to the specified Google Sheet.  
    - Airtable API token with access to the base and table.  
    - Ensure Zapier webhook URL is valid.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow combines multiple job scraping APIs and formats results for delivery through Telegram Bot with user-friendly commands. | Workflow purpose description                                    |
| Bright Data API is used to scrape LinkedIn and Indeed job listings; ensure API keys and quotas are properly configured.          | https://brightdata.com                                         |
| The Monster Jobs API endpoint is public but subject to change; monitor for breaking changes in production.                       | Monster jobs API documentation                                  |
| Markdown formatting in Telegram messages supports clickable links and emojis for better UX.                                     | https://core.telegram.org/bots/api#markdown-style               |
| Save job results in both Google Sheets and Airtable for redundancy and flexible data management.                                | Google Sheets and Airtable integrations                          |
| Usage analytics are logged to a Zapier webhook for monitoring bot usage statistics and performance.                             | Zapier webhook setup guide                                      |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.