Automated Job Hunter: Upwork Opportunity Aggregator & AI-Powered Notifier

https://n8nworkflows.xyz/workflows/automated-job-hunter--upwork-opportunity-aggregator---ai-powered-notifier-4733


# Automated Job Hunter: Upwork Opportunity Aggregator & AI-Powered Notifier

### 1. Workflow Overview

This workflow automates the process of discovering freelance job opportunities on Upwork, organizing the data, summarizing it with AI, and sending daily email notifications. It is designed for freelance professionals or agencies seeking to streamline their job search and receive concise job summaries without manual effort.

The workflow can be logically divided into three main blocks:

- **1.1 Job Fetch & Preparation**: Automatically triggers daily, fetches job listings from Upwork via the Apify platform, and formats the job data into a clean, structured format.

- **1.2 Data Logging & AI Summarization**: Logs the cleaned job data into Google Sheets for record-keeping and uses OpenAI models to generate a natural language summary of the job listings.

- **1.3 Email Notification**: Sends the AI-generated job summary via Gmail to a specified recipient for quick review.

---

### 2. Block-by-Block Analysis

#### 1.1 Job Fetch & Preparation

**Overview:**  
This block is responsible for initiating the workflow on a scheduled basis, retrieving fresh job data from Upwork through an Apify actor task, and then extracting and formatting relevant fields for downstream processing.

**Nodes Involved:**  
- Daily Upwork Job Trigger  
- Fetch Upwork Jobs (Apify)  
- Format Job Fields  

**Node Details:**

- **Daily Upwork Job Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow automatically at 9:00 AM daily.  
  - *Configuration:* Trigger set to fire once daily at 9:00 hours (UTC by default).  
  - *Inputs:* None (start node)  
  - *Outputs:* Connects to "Fetch Upwork Jobs (Apify)".  
  - *Edge Cases:* Possible failure if n8n instance is down at trigger time; no direct authentication issues.  

- **Fetch Upwork Jobs (Apify)**  
  - *Type:* HTTP Request  
  - *Role:* Calls Apify’s Upwork scraper actor to fetch recent job listings synchronously.  
  - *Configuration:*  
    - Method: POST  
    - URL: `https://api.apify.com/v2/actor-tasks/<YOUR_TASK_ID>/run-sync-get-dataset-items?token=<YOUR_API_TOKEN>` (requires user to replace placeholders)  
  - *Key Variables:* None beyond raw JSON response.  
  - *Inputs:* Receives trigger from the schedule node.  
  - *Outputs:* JSON array of job objects with fields such as `title`, `url`, `description`, `budget`, and `datePosted`.  
  - *Edge Cases:*  
    - Invalid API token or task ID leads to auth or 404 errors.  
    - Network timeouts or API rate limits.  
    - Unexpected data format if Apify actor changes output schema.  

- **Format Job Fields**  
  - *Type:* Set (Field Formatter)  
  - *Role:* Extracts and standardizes key job fields: `title`, `url`, `description`, `budget`, `datePosted`.  
  - *Configuration:* Uses expressions to assign each field from the incoming JSON, e.g. `={{ $json.title }}`.  
  - *Inputs:* JSON data from Apify HTTP node.  
  - *Outputs:* Cleaned JSON with only the necessary fields to downstream nodes.  
  - *Edge Cases:*  
    - Missing fields in input JSON may produce empty or undefined results.  
    - Expression errors if input structure varies.  

---

#### 1.2 Data Logging & AI Summarization

**Overview:**  
This block logs the formatted job entries into a Google Sheet for persistent storage and uses an AI agent powered by OpenAI to generate a human-readable summary of the jobs.

**Nodes Involved:**  
- Log Jobs to Google Sheet  
- Summarize Job Listings (LangChain Agent group):  
  - OpenAI Job Summarizer  
  - Parse Summary Output  

**Node Details:**

- **Log Jobs to Google Sheet**  
  - *Type:* Google Sheets (Append)  
  - *Role:* Appends each job listing as a new row in a specified Google Sheet for historical tracking.  
  - *Configuration:*  
    - Document ID: `1dEU6uMB4ehiGXjIExjtFQHvUjANRODawKMBGKDaTcEc` (predefined spreadsheet)  
    - Sheet Name: `gid=0` (Sheet1)  
    - Columns mapped: `title`, `url`, `description`, `budget`, `datePosted`  
  - *Credentials:* Google Sheets OAuth2  
  - *Inputs:* Formatted job JSON  
  - *Outputs:* Passes data to AI summarization node  
  - *Edge Cases:*  
    - OAuth token expiry or revoked credentials.  
    - Sheet access rights or quota limits.  
    - Data mismatch if columns are missing or renamed in the sheet.  

- **Summarize Job Listings**  
  - *Type:* LangChain Agent (AI Agent)  
  - *Role:* Coordinates AI summarization of job listings using OpenAI language models with output parsing.  
  - *Configuration:*  
    - Custom prompt defines the summary format, embedding job fields via expressions, e.g.:

      ```
      Provide a summary of the upwork jobs. It should be in email format

      Title: {{ $json.title }}
      URL: {{ $json.url }}
      Description:{{ $json.description }}
      Budget:{{ $json.budget }}
      Date posted: {{ $json.datePosted }}
      ```
    - Uses structured output parser to enforce JSON response format.  
  - *Inputs:* Job data from the Google Sheets logging node.  
  - *Outputs:* Parsed summary object with fields like `subject` and `Summary`.  
  - *Edge Cases:*  
    - OpenAI API errors: rate limits, auth errors, model unavailability.  
    - Parsing failures if AI response deviates from schema.  
    - Large input size causing timeouts or truncation.  

- **OpenAI Job Summarizer**  
  - *Type:* LangChain Language Model (OpenAI Chat)  
  - *Role:* Executes the core OpenAI GPT-4o-mini model call to generate text summary.  
  - *Configuration:*  
    - Model: `gpt-4o-mini` (lightweight GPT-4 variant)  
  - *Credentials:* OpenAI API key  
  - *Inputs:* Prompt text from the agent node  
  - *Outputs:* Raw AI text for parsing  
  - *Edge Cases:* Same as above (API limits, auth errors).  

- **Parse Summary Output**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Converts the AI-generated text into a structured JSON object with keys such as `subject` and `Summary`.  
  - *Configuration:* Example JSON schema provided to guide parsing, e.g.:

    ```json
    {
      "subject": "Upwork Job Opportunity: Webflow Expert Needed for Portfolio Website",
      "Summary": "The client is seeking a designer with strong experience in Webflow to create a responsive and elegant portfolio site."
    }
    ```
  - *Inputs:* Raw AI response from the OpenAI Job Summarizer.  
  - *Outputs:* Structured summary for email composition.  
  - *Edge Cases:*  
    - Parsing errors if AI output format deviates.  
    - Invalid JSON causing node failure.  

---

#### 1.3 Email Notification

**Overview:**  
Sends an email notification containing the summarized job listings to a configured recipient.

**Nodes Involved:**  
- Send Job Summary Email  

**Node Details:**

- **Send Job Summary Email**  
  - *Type:* Gmail (Send Email)  
  - *Role:* Delivers the AI-generated job summary via email.  
  - *Configuration:*  
    - To: `shahkar.genai@gmail.com` (can be customized)  
    - Subject: Dynamic, taken from AI output's `subject` field.  
    - Message Body: Dynamic, uses AI output’s `Summary` field.  
  - *Credentials:* Gmail OAuth2 account  
  - *Inputs:* Structured summary JSON from the parser node.  
  - *Outputs:* None (end node)  
  - *Edge Cases:*  
    - OAuth token expiry or permission errors.  
    - Email sending limits or spam filtering.  
    - Empty summary input — workflow could optionally block sending empty emails.  

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                      | Input Node(s)                 | Output Node(s)                   | Sticky Note                                      |
|----------------------------|--------------------------------|------------------------------------|------------------------------|---------------------------------|-------------------------------------------------|
| Daily Upwork Job Trigger    | Schedule Trigger               | Initiates workflow daily at 9 AM   | None                         | Fetch Upwork Jobs (Apify)        | Part of Section 1: Job Fetch & Preparation       |
| Fetch Upwork Jobs (Apify)   | HTTP Request                  | Fetches jobs from Apify API        | Daily Upwork Job Trigger      | Format Job Fields               | Part of Section 1: Job Fetch & Preparation       |
| Format Job Fields           | Set                           | Extracts and cleans job fields     | Fetch Upwork Jobs (Apify)     | Log Jobs to Google Sheet         | Part of Section 1: Job Fetch & Preparation       |
| Log Jobs to Google Sheet    | Google Sheets (Append)         | Saves jobs persistently             | Format Job Fields             | Summarize Job Listings           | Part of Section 2: Data Logging & Summary        |
| Summarize Job Listings      | LangChain Agent                | Coordinates AI summarization       | Log Jobs to Google Sheet      | Send Job Summary Email           | Part of Section 2: Data Logging & Summary        |
| OpenAI Job Summarizer       | LangChain Chat OpenAI          | Calls OpenAI GPT-4o-mini model     | Summarize Job Listings        | Summarize Job Listings           | Part of Section 2: Data Logging & Summary        |
| Parse Summary Output        | LangChain Output Parser        | Parses AI summary into structured JSON | OpenAI Job Summarizer      | Summarize Job Listings           | Part of Section 2: Data Logging & Summary        |
| Send Job Summary Email      | Gmail Send                    | Sends email notification           | Summarize Job Listings        | None                           | Part of Section 3: Job Summary Notification      |
| Sticky Note                 | Sticky Note                   | Documentation and grouping notes   | None                         | None                           | See detailed notes in workflow comments          |
| Sticky Note1                | Sticky Note                   | Documentation and grouping notes   | None                         | None                           | See detailed notes in workflow comments          |
| Sticky Note2                | Sticky Note                   | Documentation and grouping notes   | None                         | None                           | See detailed notes in workflow comments          |
| Sticky Note4                | Sticky Note                   | Detailed workflow overview and instructions | None                 | None                           | See detailed notes in workflow comments          |
| Sticky Note9                | Sticky Note                   | Workflow assistance and contact info | None                      | None                           | See detailed notes in workflow comments          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Daily Upwork Job Trigger`  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9:00 AM (adjust timezone as needed).  

2. **Create an HTTP Request node**  
   - Name: `Fetch Upwork Jobs (Apify)`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apify.com/v2/actor-tasks/<YOUR_TASK_ID>/run-sync-get-dataset-items?token=<YOUR_API_TOKEN>` (replace placeholders with your Apify task ID and API token)  
   - Connect input from `Daily Upwork Job Trigger`.  

3. **Create a Set node**  
   - Name: `Format Job Fields`  
   - Type: Set  
   - Assign fields:  
     - title: `={{ $json.title }}`  
     - url: `={{ $json.url }}`  
     - description: `={{ $json.description }}`  
     - budget: `={{ $json.budget }}`  
     - datePosted: `={{ $json.datePosted }}`  
   - Connect input from `Fetch Upwork Jobs (Apify)`.  

4. **Create a Google Sheets node**  
   - Name: `Log Jobs to Google Sheet`  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use your Google Sheet ID (e.g., `1dEU6uMB4ehiGXjIExjtFQHvUjANRODawKMBGKDaTcEc`)  
   - Sheet Name: `Sheet1` or `gid=0`  
   - Map columns: title, url, description, budget, datePosted  
   - Connect input from `Format Job Fields`.  
   - Configure Google Sheets OAuth2 credentials with required permissions.  

5. **Create a LangChain Agent node**  
   - Name: `Summarize Job Listings`  
   - Type: LangChain Agent  
   - Prompt Type: Define  
   - Prompt Text:

     ```
     Provide a summary of the upwork jobs. It should be in email format

     Title: {{ $json.title }}
     URL: {{ $json.url }}
     Description:{{ $json.description }}
     Budget:{{ $json.budget }}
     Date posted: {{ $json.datePosted }}
     ```
   - Connect input from `Log Jobs to Google Sheet`.  

6. **Create a LangChain Chat OpenAI node**  
   - Name: `OpenAI Job Summarizer`  
   - Type: LangChain Language Model (Chat)  
   - Model: `gpt-4o-mini`  
   - Connect input from `Summarize Job Listings` under language model input.  
   - Configure OpenAI API credentials with valid API key.  

7. **Create a LangChain Output Parser node**  
   - Name: `Parse Summary Output`  
   - Type: LangChain Output Parser (Structured)  
   - Provide JSON schema example:

     ```json
     {
       "subject": "Upwork Job Opportunity: Webflow Expert Needed for Portfolio Website",
       "Summary": "The client is seeking a designer with strong experience in Webflow to create a responsive and elegant portfolio site."
     }
     ```
   - Connect input from `OpenAI Job Summarizer` under output parser input.  

8. **Link `Parse Summary Output` back to the `Summarize Job Listings` node as the parsed output**  
   - Set the “ai_outputParser” connection accordingly.  

9. **Connect main output of `Summarize Job Listings` to a Gmail Send node**  
   - Name: `Send Job Summary Email`  
   - Type: Gmail  
   - To: Your desired email address (e.g., `shahkar.genai@gmail.com`)  
   - Subject: Dynamic expression, e.g. `={{ $json.output.subject }}`  
   - Message: Dynamic expression, e.g. `={{ $json.output.Summary }}`  
   - Configure Gmail OAuth2 credentials with send-mail permissions.  

10. **Finalize node connections**  
    - `Daily Upwork Job Trigger` → `Fetch Upwork Jobs (Apify)`  
    - `Fetch Upwork Jobs (Apify)` → `Format Job Fields`  
    - `Format Job Fields` → `Log Jobs to Google Sheet`  
    - `Log Jobs to Google Sheet` → `Summarize Job Listings`  
    - `Summarize Job Listings` → `Send Job Summary Email`  

11. **Test the workflow**  
    - Adjust API tokens and credentials.  
    - Run manually or wait for scheduled trigger.  
    - Verify Google Sheet is updated and email received.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Workflow automates daily fetching, logging, AI summarization, and email notification of Upwork freelance jobs.                            | Automation overview                                               |
| Apify platform is used for Upwork scraping, requiring a valid actor task and API token.                                                    | https://apify.com                                                 |
| OpenAI GPT-4o-mini model is used for cost-effective AI summarization with structured output parsing.                                       | OpenAI API documentation                                           |
| Google Sheets used as persistent storage; ensure sheet has appropriate columns and OAuth2 credentials are set up.                          | Google Sheets API                                                  |
| Gmail node requires OAuth2 credentials with permission to send emails.                                                                      | Gmail API and OAuth2 documentation                                 |
| Suggested enhancements include keyword filtering, duplicate detection, Slack alerts, and dashboards.                                       | Ideas for expansion                                               |
| Workflow assistance and support contact: Yaron at nofluff.online; YouTube and LinkedIn channels available for further learning.           | https://www.youtube.com/@YaronBeen/videos  https://www.linkedin.com/in/yaronbeen/ |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements. All handled data is legal and public.