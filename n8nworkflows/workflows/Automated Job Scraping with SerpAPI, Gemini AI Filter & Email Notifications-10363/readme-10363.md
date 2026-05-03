Automated Job Scraping with SerpAPI, Gemini AI Filter & Email Notifications

https://n8nworkflows.xyz/workflows/automated-job-scraping-with-serpapi--gemini-ai-filter---email-notifications-10363


# Automated Job Scraping with SerpAPI, Gemini AI Filter & Email Notifications

### 1. Workflow Overview

This workflow automates the process of scraping job listings using SerpAPI, filtering them with Google Gemini AI, and sending email notifications based on filtered results. It integrates Google Sheets for data storage and Microsoft Outlook for notifications. The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger and Initial Setup:** Initiates the workflow on a schedule and prepares search parameters.
- **1.2 Job Data Retrieval:** Performs multiple HTTP requests to SerpAPI to scrape job data.
- **1.3 Data Aggregation and Merging:** Splits, merges, and prepares the scraped data for comparison.
- **1.4 Dataset Comparison:** Compares newly scraped job data against existing stored data to identify new or updated listings.
- **1.5 AI Filtering with Google Gemini:** Uses Google Gemini AI via LangChain to analyze and filter job listings.
- **1.6 Conditional Update and Storage:** Based on AI output, updates or appends job records in Google Sheets.
- **1.7 Email Notification:** Aggregates relevant filtered job listings and sends an email notification via Microsoft Outlook.
- **1.8 Sub-Workflow Execution:** Calls an additional sub-workflow for further processing after data update.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Initial Setup

- **Overview:**  
Starts the workflow on a defined schedule, sets or edits initial parameters for job search queries.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Edit Fields

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Trigger  
    - Role: Entry point to run the workflow on a fixed schedule (e.g., daily/weekly).  
    - Configuration: Default scheduling parameters (not detailed in JSON).  
    - Inputs: None  
    - Outputs: Connected to Edit Fields node.  
    - Potential failures: Misconfigured schedule, workflow disabled.  

  - **Edit Fields**  
    - Type: Set node  
    - Role: Prepares or updates input parameters such as search queries or API keys for subsequent HTTP requests.  
    - Configuration: Defines or modifies variables used in HTTP requests.  
    - Inputs: From Schedule Trigger  
    - Outputs: Multiple HTTP Request nodes and Google Sheets read node  
    - Edge cases: Missing variables causing HTTP request failures.

#### 1.2 Job Data Retrieval

- **Overview:**  
Executes multiple HTTP requests to SerpAPI to retrieve job listings data across different queries or pages.

- **Nodes Involved:**  
  - HTTP Request5  
  - HTTP Request6  
  - HTTP Request7  
  - HTTP Request8  
  - HTTP Request9  
  - Split Out5  
  - Split Out6  
  - Split Out7  
  - Split Out8  
  - Split Out9

- **Node Details:**  
  - **HTTP Request Nodes (5-9)**  
    - Type: HTTP Request  
    - Role: Call SerpAPI endpoints to fetch job search results.  
    - Configuration: Uses parameters set previously, likely includes API URL, query, headers, and authentication.  
    - Inputs: From Edit Fields node  
    - Outputs: Each connected to its corresponding Split Out node.  
    - Edge cases: API quota limits, network timeouts, invalid API keys, unexpected HTTP responses.

  - **Split Out Nodes (5-9)**  
    - Type: Split Out  
    - Role: Splits array responses into individual items for downstream processing.  
    - Inputs: From respective HTTP Request node  
    - Outputs: All connected to Merge2 node with different indexes (0 to 4).  
    - Edge cases: Empty responses, malformed data arrays.

#### 1.3 Data Aggregation and Merging

- **Overview:**  
Merges split outputs from multiple HTTP requests into a consolidated dataset and prepares for comparison.

- **Nodes Involved:**  
  - Merge2  
  - Get row(s) in sheet2

- **Node Details:**  
  - **Merge2**  
    - Type: Merge  
    - Role: Combines multiple streams of job data into one unified dataset.  
    - Inputs: From Split Out nodes  
    - Outputs: To Compare Datasets1 node  
    - Edge cases: Data duplication, merging errors.

  - **Get row(s) in sheet2**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves existing job records from a specific Google Sheet tab for comparison.  
    - Inputs: From Edit Fields node (parallel branch)  
    - Outputs: To Compare Datasets1 node  
    - Edge cases: Google Sheets API rate limits, access permission issues, empty sheets.

#### 1.4 Dataset Comparison

- **Overview:**  
Compares new job listings with existing ones to detect new or updated entries.

- **Nodes Involved:**  
  - Compare Datasets1  
  - Wait

- **Node Details:**  
  - **Compare Datasets1**  
    - Type: Compare Datasets  
    - Role: Identifies differences between the new dataset (merged job data) and existing data (Google Sheets).  
    - Inputs: From Merge2 and Get row(s) in sheet2  
    - Outputs: To Wait node  
    - Edge cases: Data format mismatches, missing keys causing comparison inaccuracies.

  - **Wait**  
    - Type: Wait  
    - Role: Introduces a delay or synchronization point before AI processing.  
    - Inputs: From Compare Datasets1  
    - Outputs: To AI Agent node  
    - Edge cases: Timeout settings causing workflow delays or timeouts.

#### 1.5 AI Filtering with Google Gemini

- **Overview:**  
Processes the filtered job listings through Google Gemini AI to analyze and further refine which jobs to keep.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Switch

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: LangChain AI Model Node  
    - Role: Provides AI language model interface using Google Gemini for chat-based analysis.  
    - Inputs: From AI Agent as language model.  
    - Outputs: To AI Agent node.  
    - Requirements: Valid Google Gemini API credentials.  
    - Edge cases: API quota limits, authentication failures.

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Orchestrates the AI language model interaction and processes dataset through Gemini.  
    - Inputs: From Wait node (job data after comparison) and Google Gemini Chat Model (AI responses)  
    - Outputs: To Switch node  
    - Edge cases: AI interpretation errors, invalid responses.

  - **Switch**  
    - Type: Switch  
    - Role: Routes workflow based on AI Agent's decision (e.g., job accepted or rejected).  
    - Inputs: From AI Agent  
    - Outputs: To Append or Update Row in Sheet1 node (on positive case)  
    - Edge cases: Unhandled AI outputs, routing failures.

#### 1.6 Conditional Update and Storage

- **Overview:**  
Updates existing job entries or appends new jobs into Google Sheets based on AI filtered results.

- **Nodes Involved:**  
  - Append or update row in sheet1  
  - Call sub workflow

- **Node Details:**  
  - **Append or update row in sheet1**  
    - Type: Google Sheets (Append or Update)  
    - Role: Writes filtered job data back to the primary Google Sheet, updating or adding rows.  
    - Inputs: From Switch node  
    - Outputs: To Call sub workflow node  
    - Edge cases: Google Sheets API limits, data integrity issues.

  - **Call sub workflow**  
    - Type: Execute Workflow  
    - Role: Executes a secondary workflow for extended processing or notifications.  
    - Inputs: From Append or update row in sheet1  
    - Outputs: None specified (end of this branch)  
    - Requirements: Properly configured sub-workflow with expected input parameters.  
    - Edge cases: Sub-workflow failures, parameter mismatches.

#### 1.7 Email Notification

- **Overview:**  
Aggregates filtered job data and sends a notification email via Microsoft Outlook.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Get row(s) in sheet3  
  - Update row in sheet  
  - Aggregate  
  - Send a message1

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for external triggers to start the email notification process.  
    - Inputs: External triggers  
    - Outputs: To Get row(s) in sheet3  
    - Edge cases: Trigger misconfiguration, external workflow communication errors.

  - **Get row(s) in sheet3**  
    - Type: Google Sheets (Read)  
    - Role: Reads job data aggregated for notification purposes.  
    - Inputs: From Execute Workflow Trigger  
    - Outputs: To Update row in sheet and Aggregate node  
    - Edge cases: Google Sheets API errors.

  - **Update row in sheet**  
    - Type: Google Sheets (Update)  
    - Role: Updates status or flags in the sheet after sending notification.  
    - Inputs: From Get row(s) in sheet3  
    - Outputs: None  
    - Edge cases: Update conflicts, API quota.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines multiple rows into a single dataset/message to be emailed.  
    - Inputs: From Get row(s) in sheet3  
    - Outputs: To Send a message1 node  
    - Edge cases: Large dataset handling.

  - **Send a message1**  
    - Type: Microsoft Outlook  
    - Role: Sends email notifications with aggregated job listings.  
    - Inputs: From Aggregate node  
    - Outputs: None  
    - Requirements: Valid Outlook OAuth2 credentials.  
    - Edge cases: Authentication failures, email delivery issues.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                                    | Input Node(s)                  | Output Node(s)                  | Sticky Note                      |
|-----------------------------|---------------------------------|--------------------------------------------------|-------------------------------|--------------------------------|---------------------------------|
| Schedule Trigger            | Schedule Trigger                 | Starts workflow on schedule                        | None                          | Edit Fields                    |                                 |
| Edit Fields                | Set                             | Prepares search parameters                         | Schedule Trigger              | HTTP Request5,6,7,8,9, Get row(s) in sheet2 |                                 |
| HTTP Request5              | HTTP Request                    | Calls SerpAPI for job data                         | Edit Fields                  | Split Out5                    |                                 |
| Split Out5                 | Split Out                      | Splits job data array                              | HTTP Request5                | Merge2                       |                                 |
| HTTP Request6              | HTTP Request                    | Calls SerpAPI for job data                         | Edit Fields                  | Split Out6                    |                                 |
| Split Out6                 | Split Out                      | Splits job data array                              | HTTP Request6                | Merge2                       |                                 |
| HTTP Request7              | HTTP Request                    | Calls SerpAPI for job data                         | Edit Fields                  | Split Out7                    |                                 |
| Split Out7                 | Split Out                      | Splits job data array                              | HTTP Request7                | Merge2                       |                                 |
| HTTP Request8              | HTTP Request                    | Calls SerpAPI for job data                         | Edit Fields                  | Split Out8                    |                                 |
| Split Out8                 | Split Out                      | Splits job data array                              | HTTP Request8                | Merge2                       |                                 |
| HTTP Request9              | HTTP Request                    | Calls SerpAPI for job data                         | Edit Fields                  | Split Out9                    |                                 |
| Split Out9                 | Split Out                      | Splits job data array                              | HTTP Request9                | Merge2                       |                                 |
| Merge2                     | Merge                          | Merges multiple job datasets                       | Split Out5,6,7,8,9           | Compare Datasets1             |                                 |
| Get row(s) in sheet2       | Google Sheets                  | Reads existing job data for comparison             | Edit Fields                  | Compare Datasets1             |                                 |
| Compare Datasets1          | Compare Datasets               | Compares new vs existing job listings              | Merge2, Get row(s) in sheet2 | Wait                         |                                 |
| Wait                       | Wait                           | Synchronizes before AI processing                   | Compare Datasets1            | AI Agent                     |                                 |
| Google Gemini Chat Model   | LangChain LM Chat Model        | Google Gemini AI model for filtering jobs          | AI Agent (languageModel)     | AI Agent                     | Requires Google Gemini credentials |
| AI Agent                   | LangChain Agent                | Processes job data with AI                           | Wait, Google Gemini Chat Model | Switch                      |                                 |
| Switch                     | Switch                        | Routes based on AI decision                          | AI Agent                    | Append or update row in sheet1 |                                 |
| Append or update row in sheet1 | Google Sheets               | Adds or updates job rows                             | Switch                      | Call sub workflow            |                                 |
| Call sub workflow          | Execute Workflow               | Runs extended processing workflow                    | Append or update row in sheet1 | None                        |                                 |
| When Executed by Another Workflow | Execute Workflow Trigger | External trigger for notification workflow          | None                        | Get row(s) in sheet3          |                                 |
| Get row(s) in sheet3       | Google Sheets                  | Reads job data for notification                       | When Executed by Another Workflow | Update row in sheet, Aggregate |                                 |
| Update row in sheet        | Google Sheets                  | Updates job status flags                              | Get row(s) in sheet3         | None                        |                                 |
| Aggregate                  | Aggregate                     | Combines rows for email content                       | Get row(s) in sheet3         | Send a message1              |                                 |
| Send a message1            | Microsoft Outlook             | Sends email notifications                             | Aggregate                   | None                        | Requires Outlook OAuth2 credentials |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure desired frequency (e.g., daily at 8 AM).

2. **Add a Set node (Edit Fields):**  
   - Use to define job search parameters such as keywords, API keys.  
   - Connect Schedule Trigger → Edit Fields.

3. **Add five HTTP Request nodes (HTTP Request5 to HTTP Request9):**  
   - Configure each node to call SerpAPI endpoints for job search queries (different pages or parameters).  
   - Use variables from Edit Fields for query and authentication.  
   - Connect Edit Fields → each HTTP Request node.

4. **Add five Split Out nodes (Split Out5 to Split Out9):**  
   - Each connected to corresponding HTTP Request node to split array responses into single job entries.  
   - Connect each HTTP Request node → respective Split Out node.

5. **Add a Merge node (Merge2):**  
   - Set mode to merge all split outputs into one stream.  
   - Connect all Split Out nodes → Merge2.

6. **Add a Google Sheets node (Get row(s) in sheet2):**  
   - Configure to read existing job data from a designated sheet/tab.  
   - Connect Edit Fields → Get row(s) in sheet2.

7. **Add a Compare Datasets node (Compare Datasets1):**  
   - Configure to compare incoming merged job data with existing Google Sheets data.  
   - Connect Merge2 and Get row(s) in sheet2 → Compare Datasets1.

8. **Add a Wait node:**  
   - Connect Compare Datasets1 → Wait.  
   - Configure delay if needed for rate limiting or process pacing.

9. **Add Google Gemini Chat Model node:**  
   - Configure with Google Gemini API credentials.  
   - Connect AI Agent language model input to this node.

10. **Add AI Agent node:**  
    - Configure to use Google Gemini Chat Model as its language model.  
    - Connect Wait → AI Agent main input, Google Gemini Chat Model → AI Agent language model input.

11. **Add Switch node:**  
    - Configure based on AI Agent output criteria (e.g., if job passes filter).  
    - Connect AI Agent → Switch.

12. **Add Google Sheets node (Append or update row in sheet1):**  
    - Configure to append or update jobs based on AI decision.  
    - Connect Switch (positive branch) → Append or update row in sheet1.

13. **Add Execute Workflow node (Call sub workflow):**  
    - Configure to run a sub-workflow for extended processing or notifications.  
    - Connect Append or update row in sheet1 → Call sub workflow.

14. **Add Execute Workflow Trigger node (When Executed by Another Workflow):**  
    - Used to trigger notification workflow externally.

15. **Add Google Sheets node (Get row(s) in sheet3):**  
    - Reads job data for notification purposes.  
    - Connect When Executed by Another Workflow → Get row(s) in sheet3.

16. **Add Google Sheets node (Update row in sheet):**  
    - Updates job notification flags/status.  
    - Connect Get row(s) in sheet3 → Update row in sheet.

17. **Add Aggregate node:**  
    - Aggregates job rows for email content.  
    - Connect Get row(s) in sheet3 → Aggregate.

18. **Add Microsoft Outlook node (Send a message1):**  
    - Configure with OAuth2 credentials for Outlook.  
    - Set email content with aggregated jobs.  
    - Connect Aggregate → Send a message1.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow uses Google Gemini AI via LangChain integration for advanced filtering.               | Requires Google Gemini API access and credentials. |
| Microsoft Outlook node requires OAuth2 credentials configured for sending emails.              | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.microsoftOutlook/ |
| The workflow integrates with Google Sheets for persistent job data storage and retrieval.      | Ensure Google Sheets API credentials are set up correctly. |
| Sub-workflow execution allows modular extension of processing, e.g., additional notifications. | Sub-workflow must be created separately with expected inputs. |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.