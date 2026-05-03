Upwork Job Aggregator with OpenAI Summaries & Multi-channel Notifications

https://n8nworkflows.xyz/workflows/upwork-job-aggregator-with-openai-summaries---multi-channel-notifications-4793


# Upwork Job Aggregator with OpenAI Summaries & Multi-channel Notifications

### 1. Workflow Overview

This workflow automates the process of aggregating relevant job postings from Upwork, storing them for historical tracking, generating AI-based summaries, and sending multi-channel notifications via email. The main use case is to keep freelancers or agencies informed daily about the latest suitable job opportunities, summarized in a concise and readable format.

The workflow is logically divided into three main functional blocks:

- **1.1 Job Fetch & Preparation**: Automatically triggers the job data retrieval from Upwork using Apify, then formats the raw job data to a uniform structure for downstream processing.

- **1.2 Data Logging & AI Summarization**: Logs the structured jobs into Google Sheets for persistent storage, then uses OpenAI models to generate natural language summaries of the job listings. The AI output is parsed into structured data for clarity.

- **1.3 Notification Delivery**: Sends the AI-generated job summary via Gmail to a specified recipient, providing timely notifications with relevant job information.

---

### 2. Block-by-Block Analysis

#### 1.1 Job Fetch & Preparation

- **Overview:**  
  This block runs on a scheduled trigger, calls an Apify API to fetch the latest Upwork job listings, and formats the essential job fields into a clean structure.

- **Nodes Involved:**  
  - Daily Upwork Job Trigger  
  - Fetch Upwork Jobs (Apify)  
  - Format Job Fields

- **Node Details:**

  1. **Daily Upwork Job Trigger**  
     - Type: Schedule Trigger  
     - Role: Initiates the workflow automatically daily at 09:00 AM  
     - Configuration: Scheduled interval set to trigger once daily at hour 9  
     - Inputs: None (trigger node)  
     - Outputs: Fires to Fetch Upwork Jobs (Apify)  
     - Edge Cases: Timezone considerations; if workflow inactive at trigger time, jobs may be missed  
     - Version: 1.2

  2. **Fetch Upwork Jobs (Apify)**  
     - Type: HTTP Request  
     - Role: Executes a POST request to Apify’s Upwork Scraper task to retrieve fresh job data  
     - Configuration:  
       - URL: `https://api.apify.com/v2/actor-tasks/<YOUR_TASK_ID>/run-sync-get-dataset-items?token=<YOUR_API_TOKEN>` (replace placeholders with actual credentials)  
       - Method: POST  
       - No additional headers or body payload configured  
     - Inputs: Triggered by schedule node  
     - Outputs: JSON array of raw job objects with fields like title, url, description, budget, datePosted  
     - Edge Cases: API errors (invalid token, network timeouts), malformed JSON response, empty results  
     - Version: 4.2

  3. **Format Job Fields**  
     - Type: Set Node  
     - Role: Extracts and standardizes only the relevant job fields for easier logging and summarization  
     - Configuration: Assigns the following fields from input JSON to new keys:  
       - title  
       - url  
       - description  
       - budget  
       - datePosted  
     - Inputs: Raw job data from Apify node  
     - Outputs: Cleaned job objects forwarded to Google Sheets and AI summarizer  
     - Edge Cases: Missing fields in raw data, data type inconsistencies  
     - Version: 3.4

---

#### 1.2 Data Logging & AI Summarization

- **Overview:**  
  This block logs all structured job listings into a Google Sheet for record-keeping and uses an AI agent with OpenAI to generate human-readable summaries of the jobs. The AI output is parsed to extract structured summary data for notification.

- **Nodes Involved:**  
  - Log Jobs to Google Sheet  
  - Summarize Job Listings (AI Agent group)  
    - OpenAI Job Summarizer (Chat model)  
    - Parse Summary Output

- **Node Details:**

  1. **Log Jobs to Google Sheet**  
     - Type: Google Sheets Node  
     - Role: Appends each job as a new row into a predefined Google Sheet  
     - Configuration:  
       - Operation: Append  
       - Document ID: `1dEU6uMB4ehiGXjIExjtFQHvUjANRODawKMBGKDaTcEc`  
       - Sheet Name: `gid=0` (Sheet1)  
       - Columns mapped: title, url, description, budget, datePosted  
       - Credential: Google Sheets OAuth2 (configured with correct permissions)  
     - Inputs: Formatted job data from Format Job Fields node  
     - Outputs: Passes data forward to AI summarization  
     - Edge Cases: Google API quota limits, permission errors, malformed data causing Sheet append failures  
     - Version: 4.5

  2. **Summarize Job Listings** (AI Agent Node)  
     - Type: LangChain Agent Node  
     - Role: Crafts a prompt using job details and sends it to the OpenAI model to generate a summary in email format  
     - Configuration:  
       - Prompt uses template embedding job fields (title, url, description, budget, datePosted)  
       - Output parser enabled to parse AI response  
       - Model: Not specified here; controlled by sub-nodes  
     - Inputs: Job data from Google Sheets logging node  
     - Outputs: Parsed summary output sent to email node  
     - Edge Cases: API rate limits, prompt failures, unexpected AI output format  
     - Version: 1.9  
     - Sub-Workflow: Includes "OpenAI Job Summarizer" and "Parse Summary Output"

  3. **OpenAI Job Summarizer**  
     - Type: LangChain OpenAI Chat Model Node  
     - Role: Sends the prompt to OpenAI GPT-4o-mini model to generate job summaries  
     - Configuration:  
       - Model: GPT-4o-mini  
       - No additional options configured  
       - Credential: OpenAI API key (OpenAi account 2)  
     - Inputs: Receives prompt from AI Agent node  
     - Outputs: AI response JSON to output parser  
     - Edge Cases: API key expiration, model unavailability, network errors  
     - Version: 1.2

  4. **Parse Summary Output**  
     - Type: LangChain Structured Output Parser  
     - Role: Parses the AI response into a structured JSON object with fields like subject and Summary  
     - Configuration: Uses a JSON schema example to define expected output structure, including:  
       - subject (string)  
       - Summary (string)  
     - Inputs: Raw AI response from OpenAI Job Summarizer  
     - Outputs: Structured summary passed back to Summarize Job Listings node for final output  
     - Edge Cases: Parsing errors if AI output deviates from the schema, malformed JSON  
     - Version: 1.2

---

#### 1.3 Notification Delivery

- **Overview:**  
  Sends an email with the AI-generated job summary to a predefined recipient, providing a clean, readable digest of the latest Upwork jobs.

- **Nodes Involved:**  
  - Send Job Summary Email

- **Node Details:**

  1. **Send Job Summary Email**  
     - Type: Gmail Node  
     - Role: Sends the final job summary email to the recipient  
     - Configuration:  
       - Recipient: `shahkar.genai@gmail.com`  
       - Subject: Extracted from AI summary output’s `subject` field  
       - Message Body: Extracted from AI summary output’s `Summary` field  
       - Credential: Gmail OAuth2 (configured to allow sending email)  
     - Inputs: Structured AI summary from Summarize Job Listings node  
     - Outputs: None (final output)  
     - Edge Cases: Gmail API quota exceeded, authorization errors, empty or malformed email content  
     - Webhook ID: Present (for webhook-triggered scenarios)  
     - Version: 2.1

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                            | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                          |
|---------------------------|----------------------------------|--------------------------------------------|------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Daily Upwork Job Trigger   | Schedule Trigger                 | Triggers workflow daily at 09:00            | None                         | Fetch Upwork Jobs (Apify)         | See Section 1 Sticky Note (Job Fetch & Preparation group)                                          |
| Fetch Upwork Jobs (Apify)  | HTTP Request                    | Fetches Upwork jobs via Apify API            | Daily Upwork Job Trigger      | Format Job Fields                | See Section 1 Sticky Note (Job Fetch & Preparation group)                                          |
| Format Job Fields          | Set Node                       | Extracts and formats relevant job fields     | Fetch Upwork Jobs (Apify)     | Log Jobs to Google Sheet         | See Section 1 Sticky Note (Job Fetch & Preparation group)                                          |
| Log Jobs to Google Sheet   | Google Sheets Append Row       | Logs jobs into Google Sheets for persistence | Format Job Fields             | Summarize Job Listings           | See Section 2 Sticky Note (Data Logging & Summary Generation group)                                |
| Summarize Job Listings     | LangChain Agent                | AI agent node to generate job summaries      | Log Jobs to Google Sheet      | Send Job Summary Email           | See Section 2 Sticky Note (Data Logging & Summary Generation group)                                |
| OpenAI Job Summarizer      | LangChain OpenAI Chat          | Calls OpenAI GPT-4o-mini to summarize jobs  | Summarize Job Listings (internal) | Parse Summary Output             | See Section 2 Sticky Note (Data Logging & Summary Generation group)                                |
| Parse Summary Output       | LangChain Output Parser Structured | Parses AI output into structured JSON       | OpenAI Job Summarizer         | Summarize Job Listings (internal) | See Section 2 Sticky Note (Data Logging & Summary Generation group)                                |
| Send Job Summary Email     | Gmail Node                     | Sends email with summarized job listings    | Summarize Job Listings        | None                            | See Section 3 Sticky Note (Job Summary Notification group)                                         |
| Sticky Note                | Sticky Note                   | Documentation and grouping descriptions      | None                         | None                            | Section 1: Job Fetch & Preparation group details                                                   |
| Sticky Note1               | Sticky Note                   | Documentation and grouping descriptions      | None                         | None                            | Section 2: Data Logging & Summary Generation group details                                         |
| Sticky Note2               | Sticky Note                   | Documentation and grouping descriptions      | None                         | None                            | Section 3: Job Summary Notification group details                                                  |
| Sticky Note4               | Sticky Note                   | Overall workflow purpose and detailed notes  | None                         | None                            | Full workflow overview, advantages, and enhancement suggestions                                   |
| Sticky Note9               | Sticky Note                   | Workflow assistance contact and resources    | None                         | None                            | Contact info and links for workflow support                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named `Daily Upwork Job Trigger`:  
   - Set to trigger daily at 09:00 AM (hour 9).  
   - No credentials needed.

2. **Add an HTTP Request node** named `Fetch Upwork Jobs (Apify)`:  
   - Method: POST  
   - URL: `https://api.apify.com/v2/actor-tasks/<YOUR_TASK_ID>/run-sync-get-dataset-items?token=<YOUR_API_TOKEN>` (replace placeholders)  
   - Connect output of `Daily Upwork Job Trigger` to this node.  
   - No body or headers required unless your Apify task needs them.

3. **Add a Set node** named `Format Job Fields`:  
   - Add assignments for:  
     - title → `={{ $json.title }}`  
     - url → `={{ $json.url }}`  
     - description → `={{ $json.description }}`  
     - budget → `={{ $json.budget }}`  
     - datePosted → `={{ $json.datePosted }}`  
   - Connect output of `Fetch Upwork Jobs (Apify)` to this node.

4. **Add a Google Sheets node** named `Log Jobs to Google Sheet`:  
   - Operation: Append  
   - Document ID: Your Google Sheet ID (e.g., `1dEU6uMB4ehiGXjIExjtFQHvUjANRODawKMBGKDaTcEc`)  
   - Sheet Name: `Sheet1` or specify `gid=0`  
   - Map columns: title, url, description, budget, datePosted to corresponding fields from input.  
   - Set up Google Sheets OAuth2 credentials with write access.  
   - Connect output of `Format Job Fields` to this node.

5. **Add a LangChain Agent node** named `Summarize Job Listings`:  
   - Enable output parser.  
   - Set prompt template incorporating job fields (title, url, description, budget, datePosted), instructing AI to produce email-format summary.  
   - Connect output of `Log Jobs to Google Sheet` to this node.

6. **Within the LangChain Agent node, add sub-nodes:**  
   - **LangChain OpenAI Chat node** named `OpenAI Job Summarizer`:  
     - Model: `gpt-4o-mini`  
     - Credentials: OpenAI API key configured  
     - Connect as per LangChain agent setup.  
   - **LangChain Output Parser Structured node** named `Parse Summary Output`:  
     - Provide JSON schema example with fields `subject` and `Summary`.  
     - Connect output of `OpenAI Job Summarizer` to this node.  
   - Connect output of `Parse Summary Output` back to `Summarize Job Listings` node output.

7. **Add a Gmail node** named `Send Job Summary Email`:  
   - Recipient: specify email address (e.g., `shahkar.genai@gmail.com`)  
   - Subject: Use expression to map to `{{ $json.output.subject }}` from AI output  
   - Message: Use expression to map to `{{ $json.output.Summary }}`  
   - Configure Gmail OAuth2 credentials with send email permission.  
   - Connect output of `Summarize Job Listings` to this node.

8. **Verify all connections**:  
   - `Daily Upwork Job Trigger` → `Fetch Upwork Jobs (Apify)` → `Format Job Fields` → `Log Jobs to Google Sheet` → `Summarize Job Listings` → `Send Job Summary Email`  
   - LangChain sub-workflow nodes connected internally for summarization.

9. **Test the workflow** by triggering manually or waiting for the scheduled time. Validate data flows, sheet entries, AI summaries, and email receipt.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow designed to be fully automated for daily Upwork job aggregation, logging, summarization, and email notification.                                                   | General workflow purpose                                                                            |
| Apify actor task must be configured and deployed separately to scrape Upwork jobs. Replace `<YOUR_TASK_ID>` and `<YOUR_API_TOKEN>` accordingly.                              | Apify API integration instructions                                                                 |
| Google Sheets document should have columns: `title`, `url`, `description`, `budget`, `datePosted` for proper data logging.                                                   | Google Sheets setup                                                                                  |
| OpenAI GPT-4o-mini model is used for summarization; API key with appropriate quota needed.                                                                                    | OpenAI API usage                                                                                    |
| Gmail OAuth2 credentials must be authorized for sending emails on behalf of the user.                                                                                          | Gmail node credential setup                                                                          |
| Suggested workflow improvements include adding keyword filters, duplicate prevention, Slack notifications, and dashboard reporting.                                          | Optional enhancements                                                                              |
| Contact for workflow support: Yaron@nofluff.online. Additional resources available at YouTube (https://www.youtube.com/@YaronBeen/videos) and LinkedIn (https://www.linkedin.com/in/yaronbeen/) | Workflow assistance and learning resources                                                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a workflow automation tool. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.