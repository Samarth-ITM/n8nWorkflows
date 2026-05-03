Smart Job Search: Resume Scoring & Tailoring with OpenAI, Apify, and Airtable

https://n8nworkflows.xyz/workflows/smart-job-search--resume-scoring---tailoring-with-openai--apify--and-airtable-3724


# Smart Job Search: Resume Scoring & Tailoring with OpenAI, Apify, and Airtable

### 1. Workflow Overview

This workflow automates the process of job searching, resume scoring, and tailoring using AI and multiple integrations. It is designed for job seekers who want to efficiently find relevant job listings, evaluate their fit against their current CV, generate tailored resumes for each job, and organize all data for tracking.

**Target Use Case:**  
Professionals who want to automate daily job searches, match their resumes to job postings, and maintain an organized record of applications and AI-generated insights.

**Logical Blocks:**

- **1.1 Trigger & Input Reception**  
  Initiates the workflow daily and fetches user job preferences from Google Sheets.

- **1.2 Job Search & Data Extraction**  
  Iterates over job preferences, scrapes job listings from job boards using Apify, and cleans/extracts relevant job data.

- **1.3 Job Filtering & Archiving**  
  Filters jobs posted within the last 48 hours for processing; archives older jobs in Airtable.

- **1.4 AI Scoring of Job Fit**  
  Scores each recent job against the user's current CV using an OpenAI agent.

- **1.5 AI-Driven CV Revamping**  
  Generates a tailored CV for each job based on AI scoring and job details.

- **1.6 Data Storage**  
  Saves job listings, AI scores, match reasons, and revamped CVs into Airtable for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

- **Overview:**  
  This block triggers the workflow daily and reads the user's job search preferences from a Google Sheet.

- **Nodes Involved:**  
  - 🕒 Trigger: Daily Job Fetch  
  - 📄 Fetch Job Preferences (Google Sheets)  
  - Split Job Preferences

- **Node Details:**

  - **🕒 Trigger: Daily Job Fetch**  
    - Type: Cron Trigger  
    - Role: Initiates workflow execution daily at a scheduled time.  
    - Configuration: Default daily schedule (exact time configurable).  
    - Inputs: None (trigger node).  
    - Outputs: Triggers the next node to fetch job preferences.  
    - Edge cases: Cron misconfiguration or n8n downtime may delay execution.

  - **📄 Fetch Job Preferences (Google Sheets)**  
    - Type: Google Sheets node  
    - Role: Reads job preferences such as job titles and locations from a predefined Google Sheet.  
    - Configuration: Uses Google Sheets API credentials; reads specific sheet and range.  
    - Inputs: Trigger from Cron node.  
    - Outputs: Array of job preference records.  
    - Edge cases: API authentication failure, empty or malformed sheet data.

  - **Split Job Preferences**  
    - Type: SplitInBatches  
    - Role: Splits the array of job preferences into batches for iterative processing.  
    - Configuration: Batch size configurable (default likely 1).  
    - Inputs: Job preferences array from Google Sheets.  
    - Outputs: Single job preference per batch for processing.  
    - Edge cases: Large data sets may cause performance issues.

---

#### 2.2 Job Search & Data Extraction

- **Overview:**  
  For each job preference, this block scrapes job listings from job boards via Apify, then cleans and extracts relevant job data.

- **Nodes Involved:**  
  - 🔄 Loop: Search Jobs per Preference (SplitInBatches)  
  - 🔍 Apify: Scrape Jobs (HTTP Request)  
  - 🧹 Clean & Extract Job Data (Code)

- **Node Details:**

  - **🔄 Loop: Search Jobs per Preference**  
    - Type: SplitInBatches  
    - Role: Iterates over each job preference to perform job scraping.  
    - Configuration: Batch size configurable; processes one preference at a time.  
    - Inputs: Individual job preference from previous split node.  
    - Outputs: Passes job preference to Apify scraper and cleaning node.  
    - Edge cases: Long job preference lists may increase runtime.

  - **🔍 Apify: Scrape Jobs**  
    - Type: HTTP Request  
    - Role: Calls Apify API to scrape job listings matching the current job preference.  
    - Configuration: Uses Apify API key; sends job title, location, and other parameters to scraper actor (e.g., Indeed or LinkedIn scraper).  
    - Inputs: Job preference parameters.  
    - Outputs: Raw scraped job listings JSON.  
    - Edge cases: API key invalid, scraper actor downtime, rate limits, unexpected data format.

  - **🧹 Clean & Extract Job Data**  
    - Type: Code (JavaScript)  
    - Role: Parses and normalizes raw job data into structured format with fields like job title, company, location, description, posting date, etc.  
    - Configuration: Custom script to extract and clean data fields.  
    - Inputs: Raw job listings from Apify.  
    - Outputs: Cleaned job listings array.  
    - Edge cases: Unexpected data structure, missing fields, parsing errors.

---

#### 2.3 Job Filtering & Archiving

- **Overview:**  
  Filters job listings to process only those posted within the last 48 hours; archives older jobs in Airtable.

- **Nodes Involved:**  
  - Filter: Recent Jobs (<48h) (If node)  
  - 🗂️ Archive: Old Jobs (Airtable)

- **Node Details:**

  - **Filter: Recent Jobs (<48h)**  
    - Type: If node  
    - Role: Checks each job's posting date to determine if it is recent (within 48 hours).  
    - Configuration: Compares job posting date with current date minus 48 hours.  
    - Inputs: Cleaned job listings.  
    - Outputs:  
      - True branch: Recent jobs for further scoring.  
      - False branch: Older jobs for archiving.  
    - Edge cases: Incorrect date formats, timezone issues.

  - **🗂️ Archive: Old Jobs (Airtable)**  
    - Type: Airtable node  
    - Role: Stores older job listings (>48 hours) into an Airtable archive table.  
    - Configuration: Uses Airtable API key; targets archive base and table; maps job fields accordingly.  
    - Inputs: Jobs filtered as old.  
    - Outputs: Confirmation of stored records.  
    - Edge cases: API authentication failure, rate limits, data mapping errors.

---

#### 2.4 AI Scoring of Job Fit

- **Overview:**  
  Uses an OpenAI agent to score how well each recent job matches the user's current CV.

- **Nodes Involved:**  
  - 🔄 Loop: Score New Jobs (SplitInBatches)  
  - 🤖 AI: Score CV vs Job (Langchain Agent)  
  - 📄 Parse AI Score Output (Code)

- **Node Details:**

  - **🔄 Loop: Score New Jobs**  
    - Type: SplitInBatches  
    - Role: Iterates over recent jobs to score each individually.  
    - Configuration: Batch size configurable (likely 1).  
    - Inputs: Recent jobs from filter node.  
    - Outputs: Passes each job to AI scoring node.  
    - Edge cases: Large job volume may increase processing time.

  - **🤖 AI: Score CV vs Job**  
    - Type: Langchain Agent (OpenAI)  
    - Role: Sends job description and user CV to OpenAI to obtain a compatibility score and match reasoning.  
    - Configuration: Uses OpenAI API key; configured with prompt templates to evaluate fit.  
    - Inputs: Job details and CV text.  
    - Outputs: AI-generated score and textual explanation.  
    - Edge cases: API rate limits, prompt failures, model errors, network timeouts.

  - **📄 Parse AI Score Output**  
    - Type: Code (JavaScript)  
    - Role: Parses AI response to extract numeric score and textual match reason.  
    - Configuration: Custom parsing logic to handle AI output format.  
    - Inputs: Raw AI response.  
    - Outputs: Structured score and reason fields.  
    - Edge cases: Unexpected AI output format, parsing errors.

---

#### 2.5 AI-Driven CV Revamping

- **Overview:**  
  Generates a tailored CV for each job based on the AI scoring and job details, enhancing the user's resume to better match the job.

- **Nodes Involved:**  
  - 🔄 Loop: Generate CV Suggestions (SplitInBatches)  
  - ✍️ AI: Revamp CV Based on Job (OpenAI Chat Model)  
  - Job Extract with revamped scoring (Code)

- **Node Details:**

  - **🔄 Loop: Generate CV Suggestions**  
    - Type: SplitInBatches  
    - Role: Iterates over jobs scored to generate tailored CVs.  
    - Configuration: Batch size configurable.  
    - Inputs: Jobs with AI scores and match reasons.  
    - Outputs: Passes each job to AI CV revamping node.  
    - Edge cases: Processing time for large job sets.

  - **✍️ AI: Revamp CV Based on Job**  
    - Type: OpenAI Chat Model (Langchain)  
    - Role: Uses OpenAI GPT model to generate a customized CV version tailored to the job description and scoring insights.  
    - Configuration: Uses OpenAI API key; prompt designed to rewrite CV content accordingly.  
    - Inputs: Job details, AI score, match reason, and original CV.  
    - Outputs: Revamped CV text.  
    - Edge cases: API errors, prompt misconfiguration, output length limits.

  - **Job Extract with revamped scoring**  
    - Type: Code (JavaScript)  
    - Role: Processes and prepares the final job data including revamped CV and updated scoring for storage.  
    - Configuration: Custom script to merge all relevant data fields.  
    - Inputs: Revamped CV and job data.  
    - Outputs: Structured final job record.  
    - Edge cases: Data inconsistencies, script errors.

---

#### 2.6 Data Storage

- **Overview:**  
  Saves all processed job data, including job details, AI scores, match reasons, and revamped CVs into Airtable for tracking and future reference.

- **Nodes Involved:**  
  - 🗂️ Save: Final Job Data (Airtable)

- **Node Details:**

  - **🗂️ Save: Final Job Data (Airtable)**  
    - Type: Airtable node  
    - Role: Inserts or updates records in Airtable with full job and AI-enhanced CV data.  
    - Configuration: Uses Airtable API key; targets main jobs tracking base and table; maps fields such as job_title, company, location, compatibilityScore, matchReason, revampedCV, etc.  
    - Inputs: Final job data from code node.  
    - Outputs: Confirmation of saved records.  
    - Edge cases: API limits, data mapping errors, network issues.

---

### 3. Summary Table

| Node Name                        | Node Type                          | Functional Role                         | Input Node(s)                     | Output Node(s)                        | Sticky Note                           |
|---------------------------------|----------------------------------|---------------------------------------|----------------------------------|-------------------------------------|-------------------------------------|
| 🕒 Trigger: Daily Job Fetch       | Cron Trigger                     | Starts workflow daily                  | None                             | 📄 Fetch Job Preferences (Google Sheets) |                                     |
| 📄 Fetch Job Preferences (Google Sheets) | Google Sheets                   | Reads user job preferences             | 🕒 Trigger: Daily Job Fetch       | Split Job Preferences                |                                     |
| Split Job Preferences             | SplitInBatches                   | Splits job preferences for iteration  | 📄 Fetch Job Preferences          | 🔄 Loop: Search Jobs per Preference  |                                     |
| 🔄 Loop: Search Jobs per Preference | SplitInBatches                   | Iterates over job preferences          | Split Job Preferences             | 🧹 Clean & Extract Job Data, 🔍 Apify: Scrape Jobs |                                     |
| 🔍 Apify: Scrape Jobs             | HTTP Request                    | Scrapes job listings from job boards  | 🔄 Loop: Search Jobs per Preference | 🔄 Loop: Search Jobs per Preference  |                                     |
| 🧹 Clean & Extract Job Data       | Code                            | Cleans and structures scraped job data | 🔄 Loop: Search Jobs per Preference | Filter: Recent Jobs (<48h)           |                                     |
| Filter: Recent Jobs (<48h)        | If                              | Filters jobs posted within last 48h   | 🧹 Clean & Extract Job Data       | 🔄 Loop: Score New Jobs, 🗂️ Archive: Old Jobs (Airtable) |                                     |
| 🗂️ Archive: Old Jobs (Airtable)   | Airtable                        | Archives old jobs (>48h)               | Filter: Recent Jobs (<48h)        | None                                |                                     |
| 🔄 Loop: Score New Jobs           | SplitInBatches                   | Iterates over recent jobs for scoring | Filter: Recent Jobs (<48h)        | 🤖 AI: Score CV vs Job, 🔄 Loop: Generate CV Suggestions |                                     |
| 🤖 AI: Score CV vs Job           | Langchain Agent (OpenAI)         | Scores job fit against user CV         | 🔄 Loop: Score New Jobs           | 📄 Parse AI Score Output             |                                     |
| 📄 Parse AI Score Output          | Code                            | Parses AI scoring response             | 🤖 AI: Score CV vs Job            | 🔄 Loop: Score New Jobs              |                                     |
| 🔄 Loop: Generate CV Suggestions | SplitInBatches                   | Iterates over jobs to generate CVs    | 🔄 Loop: Score New Jobs           | ✍️ AI: Revamp CV Based on Job, 🗂️ Save: Final Job Data (Airtable) |                                     |
| ✍️ AI: Revamp CV Based on Job     | OpenAI Chat Model               | Generates tailored CV for each job    | 🔄 Loop: Generate CV Suggestions  | Job Extract with revamped scoring    |                                     |
| Job Extract with revamped scoring | Code                            | Prepares final job data with revamped CV | ✍️ AI: Revamp CV Based on Job     | 🔄 Loop: Generate CV Suggestions    |                                     |
| 🗂️ Save: Final Job Data (Airtable) | Airtable                        | Saves final job and CV data            | 🔄 Loop: Generate CV Suggestions  | None                                |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node**  
   - Type: Cron  
   - Configure to run daily at your preferred time.

2. **Add Google Sheets node to fetch job preferences**  
   - Type: Google Sheets  
   - Connect to Cron Trigger node.  
   - Configure with Google Sheets API credentials.  
   - Specify spreadsheet ID and range containing job preferences (columns like `job_title`, `job_location`).

3. **Add SplitInBatches node to split job preferences**  
   - Connect to Google Sheets node.  
   - Configure batch size (e.g., 1) to process one preference at a time.

4. **Add SplitInBatches node to loop over job preferences**  
   - Connect to previous SplitInBatches node.  
   - This node manages iteration over each job preference for scraping.

5. **Add HTTP Request node to call Apify scraper**  
   - Connect to Loop node.  
   - Configure with Apify API key in headers.  
   - Set method to POST or GET as per Apify API.  
   - Pass job title and location from current batch as parameters to the scraper actor.

6. **Add Code node to clean and extract job data**  
   - Connect to Loop node (parallel to HTTP Request node).  
   - Write JavaScript to parse Apify response, extract fields: job title, company, location, description, posting date, etc.

7. **Add If node to filter recent jobs (<48h)**  
   - Connect from Code node.  
   - Configure condition to compare job posting date with current date minus 48 hours.

8. **Add Airtable node to archive old jobs**  
   - Connect to False branch of If node.  
   - Configure Airtable API credentials.  
   - Set base and table for archived jobs.  
   - Map job fields accordingly.

9. **Add SplitInBatches node to loop over recent jobs for scoring**  
   - Connect to True branch of If node.  
   - Configure batch size (e.g., 1).

10. **Add Langchain Agent node for AI scoring**  
    - Connect to scoring Loop node.  
    - Configure with OpenAI API credentials.  
    - Set prompt template to compare job description with user CV and produce compatibility score and match reason.

11. **Add Code node to parse AI score output**  
    - Connect to AI scoring node.  
    - Write JavaScript to extract numeric score and textual reason from AI response.

12. **Add SplitInBatches node to generate CV suggestions**  
    - Connect to scoring Loop node (after parsing).  
    - Configure batch size (e.g., 1).

13. **Add OpenAI Chat Model node to revamp CV**  
    - Connect to CV generation Loop node.  
    - Configure with OpenAI API credentials.  
    - Use prompt to rewrite CV tailored to the job and AI score.

14. **Add Code node to prepare final job data**  
    - Connect to OpenAI CV node.  
    - Merge job details, AI scores, match reasons, and revamped CV into a single structured object.

15. **Add Airtable node to save final job data**  
    - Connect to Code node.  
    - Configure Airtable API credentials.  
    - Set base and table for current processed jobs.  
    - Map all relevant fields: job_title, company, location, date_posted, job_type, description, link, compatibilityScore, matchReason, revampedCV, newCompatibilityScore, newMatchReason.

16. **Connect all nodes in the order described, ensuring proper data flow and error handling.**

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow saves 10+ hours/week by automating job search and CV tailoring.                          | Workflow Overview                                                                                       |
| Customize AI prompts to favor specific criteria like remote work, leadership, or certifications. | Setup instructions                                                                                      |
| Replace Airtable with other storage or notification systems like Notion, Google Sheets, or Slack.| Setup instructions                                                                                      |
| Swap Apify scraper to target different job boards or niche sources easily.                        | Setup instructions                                                                                      |
| For setup guidance or customization, contact: ashish060921@gmail.com                             | Support contact                                                                                        |
| Airtable tables require specific columns for job data and AI outputs as per setup instructions.  | Setup instructions                                                                                      |
| Use GPT-4 Turbo or equivalent for best AI scoring and CV enhancement results.                     | API Credentials and AI node configuration                                                              |

---

This document provides a detailed, structured understanding of the "Smart Job Search: Resume Scoring & Tailoring with OpenAI, Apify, and Airtable" workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.