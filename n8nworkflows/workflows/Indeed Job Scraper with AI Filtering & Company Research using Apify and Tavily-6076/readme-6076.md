Indeed Job Scraper with AI Filtering & Company Research using Apify and Tavily

https://n8nworkflows.xyz/workflows/indeed-job-scraper-with-ai-filtering---company-research-using-apify-and-tavily-6076


# Indeed Job Scraper with AI Filtering & Company Research using Apify and Tavily

### 1. Workflow Overview

This workflow automates the process of scraping job postings from Indeed or similar platforms, filtering them using AI to identify roles suitable for AI automation, updating a company database, and enriching company data with decision-maker research. It is designed primarily for lead generation targeting low-complexity, automatable job positions in specific industries. The workflow integrates Apify for data scraping, OpenAI (via Langchain) for AI filtering, Google Sheets for data storage, and Tavily for company decision-maker research.

**Logical Blocks:**

- **1.1 Input Reception & Data Retrieval**  
  Receives job posting data via webhook, retrieves dataset items from Apify.

- **1.2 Data Preprocessing & Deduplication**  
  Cleans and filters dataset, removes duplicates, and checks for existing entries in Google Sheets.

- **1.3 AI-Based Job Filtering**  
  Uses OpenAI GPT-4.1-mini model to classify job posts as relevant or irrelevant based on predefined criteria.

- **1.4 Database Update & Filtering**  
  Merges filtered job posts with existing database entries, filters for relevance and uniqueness.

- **1.5 Company Decision Maker Enrichment**  
  For relevant jobs, uses Tavily to research and identify key decision makers at the hiring company.

- **1.6 Data Appending & Workflow Looping**  
  Appends new relevant job data to Google Sheets and loops with controlled wait times for rate limits and batching.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Retrieval

**Overview:**  
This block listens for incoming webhook POST requests which trigger dataset retrieval from Apify, providing raw job posting data for processing.

**Nodes Involved:**  
- Webhook  
- Get dataset items  
- Loop Over Items1

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook Trigger  
  - Configured to receive POST requests on a unique path.  
  - Entry point for external job posting data.  
  - Input: External POST request with job scraping metadata.  
  - Output: Passes webhook JSON body to next node.  
  - Errors: Network or request validation errors may occur.

- **Get dataset items**  
  - Type: Apify Dataset Retrieval  
  - Retrieves up to 500 items from the dataset ID provided dynamically from webhook body.  
  - Inputs webhook data, outputs raw scraped job items.  
  - Notes: Dataset ID is static but derived from webhook input.  
  - Potential errors: Invalid dataset ID, API rate limits, or connectivity issues.

- **Loop Over Items1**  
  - Type: Split in Batches  
  - Splits dataset items into batches of 55 for manageable processing downstream.  
  - Input: Dataset items array.  
  - Output: Batch of items per execution cycle.  
  - Useful for rate-limiting and chunked processing.

---

#### 1.2 Data Preprocessing & Deduplication

**Overview:**  
Transforms and filters raw data, removes duplicates, and cross-checks with existing Google Sheets rows to avoid processing already known companies.

**Nodes Involved:**  
- Edit Fields1  
- Filter 1  
- Merge  
- Filter 2  
- Remove Duplicates  
- Get row(s) in sheet  
- Filter Outlet

**Node Details:**

- **Edit Fields1**  
  - Type: Set Node  
  - Adds or modifies fields; sets default employee count "50 to 100" if missing.  
  - Excludes unnecessary fields to simplify payload.  
  - Input: Batch of job items.  
  - Output: Edited job items with cleaned fields.

- **Filter 1**  
  - Type: If Node  
  - Checks if `companyNumEmployees` field exists.  
  - If missing, routes data differently to ensure data completeness.  
  - Input: Edited job items.  
  - Potential failure: Null or malformed employee count.

- **Merge**  
  - Type: Merge Node  
  - Combines data streams from Filter 1's false branch and Edit Fields1's output for unified processing.

- **Filter 2**  
  - Type: Filter Node  
  - Filters jobs with an existing corporate website and company size less than 250 employees.  
  - Input: Merged data from previous node.  
  - Output: Filtered company records for further processing.

- **Remove Duplicates**  
  - Type: Remove Duplicates Node  
  - Removes duplicate companies by comparing `companyName`.  
  - Input: Filtered job entries.  
  - Output: Unique job entries for next step.

- **Get row(s) in sheet**  
  - Type: Google Sheets Node  
  - Looks up existing companies in Google Sheets by matching `companyName`.  
  - Input: Unique job entries.  
  - Output: Rows found or empty if new company.

- **Filter Outlet**  
  - Type: Merge Node (keepNonMatches)  
  - Separates new companies (not in Google Sheets) from those already in the database.  
  - Input: Data from Google Sheets lookup and unique job entries.  
  - Output: Two streams: new companies and existing companies.

---

#### 1.3 AI-Based Job Filtering

**Overview:**  
Uses OpenAI GPT-4.1-mini to evaluate job posts and tag them as "Relevant" or not, based on predefined filtering criteria focusing on automatable roles.

**Nodes Involved:**  
- Job Filter  
- Edit Fields  
- Relevant Job posting?  
- True Jobs

**Node Details:**

- **Job Filter**  
  - Type: Langchain OpenAI Node  
  - Uses GPT-4.1-mini model with a detailed system prompt instructing classification of jobs suitable for AI automation.  
  - Input: Job title, partial description, and URL.  
  - Output: JSON verdict (`true` or `false`) and job URL.  
  - Notes: Filtering criteria include urgency, role type, industry exclusions, and keywords.  
  - Potential errors: API rate limits, malformed inputs, or ambiguous job descriptions.

- **Edit Fields**  
  - Type: Set Node  
  - Extracts verdict and jobUrl from OpenAI JSON response into usable fields for filtering.  
  - Input: Job Filter output.  
  - Output: Fields `verdict` and `jobUrl` for conditional checks.

- **Relevant Job posting?**  
  - Type: If Node  
  - Checks if `verdict` equals `true`.  
  - Routes relevant jobs further, discards or archives others.

- **True Jobs**  
  - Type: Merge Node  
  - Combines streams of relevant jobs for subsequent processing.

---

#### 1.4 Database Update & Filtering

**Overview:**  
Filters out duplicate or irrelevant job URLs, appends new relevant jobs to Google Sheets, then waits before restarting the batch process.

**Nodes Involved:**  
- Anything to add?  
- Find DM  
- Append row in sheet  
- Wait

**Node Details:**

- **Anything to add?**  
  - Type: If Node  
  - Checks if `jobUrl` exists before continuing with decision maker search or looping.  
  - Input: Relevant jobs from True Jobs node.  
  - Output: Routes to Find DM or restarts loop.

- **Find DM**  
  - Type: Tavily Node  
  - Queries Tavily API to find decision makers (CEO/Owner/Founder) for the company using corporate website and job URL.  
  - Outputs JSON with company name, website, and decision maker.  
  - Input: Jobs with valid URLs.  
  - Errors: API failures, no decision maker found, or incorrect website URL.

- **Append row in sheet**  
  - Type: Google Sheets Append Node  
  - Adds new job and enriched company data (including decision maker, location, salary, etc.) into a Google Sheet database.  
  - Uses defined schema matching sheet columns.  
  - Configured to append rows in user-entered format.  
  - Errors: Sheet access issues, data type mismatches.

- **Wait**  
  - Type: Wait Node  
  - Pauses workflow for 1.5 minutes to avoid API rate limits and manage load.  
  - After waiting, triggers Loop Over Items1 to process next batch.

---

### 3. Summary Table

| Node Name          | Node Type                            | Functional Role                              | Input Node(s)                 | Output Node(s)                | Sticky Note                                               |
|--------------------|------------------------------------|----------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------|
| Webhook            | Webhook (HTTP Trigger)              | Receives incoming job scraping webhook POST  | -                            | Get dataset items             |                                                           |
| Get dataset items  | Apify Dataset                      | Retrieves job postings dataset from Apify    | Webhook                      | Loop Over Items1              | Incorrect Datasetid (Static)                              |
| Loop Over Items1   | Split in Batches                   | Batches dataset items for manageable processing | Get dataset items             | Filter 1, Filter 1 (false branch) |                                                           |
| Edit Fields1       | Set                              | Cleans and adds default data on employees    | Filter 1 (true branch)        | Merge                        |                                                           |
| Filter 1           | If                               | Checks existence of employee count            | Loop Over Items1 (false branch) | Edit Fields1, Merge          | #Employees exist?                                          |
| Merge              | Merge                            | Combines data streams                          | Edit Fields1, Filter 1 (false) | Filter 2                     |                                                           |
| Filter 2           | Filter                           | Filters companies with website & <250 employees | Merge                        | Remove Duplicates             | Website & Employees < 250?                                |
| Remove Duplicates  | Remove Duplicates                 | Removes duplicate company entries              | Filter 2                     | Get row(s) in sheet, Filter Outlet |                                                           |
| Get row(s) in sheet| Google Sheets                    | Checks existing companies in database          | Remove Duplicates             | Filter Outlet                 | here                                                      |
| Filter Outlet      | Merge (keepNonMatches)            | Separates new from existing companies          | Get row(s) in sheet, Remove Duplicates | Job Filter, True Jobs         | What do we not have in our DB?                            |
| Job Filter         | Langchain OpenAI (GPT-4.1-mini)  | AI filters job relevance using job text       | Filter Outlet                | Edit Fields                  | Filter needs tuning                                       |
| Edit Fields        | Set                              | Extracts verdict and jobUrl from AI response  | Job Filter                   | Relevant Job posting?         |                                                           |
| Relevant Job posting? | If                             | Checks if job is relevant                       | Edit Fields                  | True Jobs, discard branch     | Relevant Job posting?                                     |
| True Jobs          | Merge                            | Combines relevant job posts                     | Filter Outlet, Relevant Job posting? | Anything to add?             |                                                           |
| Anything to add?   | If                               | Checks if jobUrl exists before enrichment      | True Jobs                    | Find DM, Loop Over Items1     |                                                           |
| Find DM            | Tavily                           | Researches decision makers for companies       | Anything to add?              | Append row in sheet           |                                                           |
| Append row in sheet| Google Sheets                    | Appends enriched job and company data          | Find DM                      | Wait                         |                                                           |
| Wait               | Wait                            | Waits 1.5 minutes before next batch             | Append row in sheet          | Loop Over Items1              |                                                           |
| Sticky Note        | Sticky Note                     | Comments and block labels                       | -                            | -                            | ## Filter oncoming data                                   |
| Sticky Note1       | Sticky Note                     | Comments and block labels                       | -                            | -                            | ## Filter job postings                                   |
| Sticky Note2       | Sticky Note                     | Comments and block labels                       | -                            | -                            | ## Update DataBase                                       |
| Sticky Note3       | Sticky Note (disabled)           | Workflow notes and TODOs                         | -                            | -                            | Notes about workflow logic and future improvements       |
| Sticky Note4       | Sticky Note                     | Visual spacing or placeholders                   | -                            | -                            |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: unique string (e.g., `8b86cbbd-8f0f-424d-86b8-5f2cdb3b28a6`)  
   - No special authentication.

2. **Create Apify Node to Get Dataset Items**  
   - Type: Apify  
   - Operation: Get items from Dataset  
   - Dataset ID: Set dynamically from webhook JSON `{{$json.body.resource.defaultDatasetId}}`  
   - Limit: 500 items  
   - Credentials: Apify API credentials configured.

3. **Create Split In Batches Node (Loop Over Items1)**  
   - Batch Size: 55  
   - Connect from Apify node output.

4. **Create If Node (Filter 1) to Check Employee Count**  
   - Condition: `companyNumEmployees` field NOT exists  
   - True branch: Connect to "Edit Fields1"  
   - False branch: Connect to "Merge"

5. **Create Set Node (Edit Fields1)**  
   - Exclude fields: `descriptionHtml, benefits, occupation, attributes, shiftAndSchedule, applyUrl, requirements`  
   - Add field: `companyNumEmployees` = "50 to 100" (string)  
   - Connect from Filter 1 True branch.

6. **Create Merge Node**  
   - Mode: Merge two inputs  
   - Connect Edit Fields1 output and Filter 1 False branch to inputs.

7. **Create Filter Node (Filter 2)**  
   - Conditions:  
     - `companyLinks.corporateWebsite` exists  
     - Numeric value parsed from `companyNumEmployees` (last number after "to", remove commas) is less than 250  
   - Connect Merge node output.

8. **Create Remove Duplicates Node**  
   - Compare fields: `companyName`  
   - Connect from Filter 2 output.

9. **Create Google Sheets Node (Get row(s) in sheet)**  
   - Operation: Lookup rows  
   - Sheet ID and Document ID: Use your Google Sheet document and sheet IDs  
   - Lookup Column: `companyName`  
   - Lookup Value: from incoming items `{{$json.companyName}}`  
   - Credentials: Google Sheets OAuth2 configured  
   - Connect Remove Duplicates output.

10. **Create Merge Node (Filter Outlet)**  
    - Mode: Keep Non Matches (records not found in sheet)  
    - Connect Remove Duplicates as input 1, Get row(s) in sheet as input 2.

11. **Create Langchain OpenAI Node (Job Filter)**  
    - Model: GPT-4.1-mini  
    - System prompt: Detailed instructions for filtering relevant job posts (see overview)  
    - Input: Job title, description snippet, jobUrl  
    - Output: JSON with verdict and jobUrl  
    - Credentials: OpenAI API key  
    - Connect Filter Outlet output of new companies.

12. **Create Set Node (Edit Fields)**  
    - Extract fields:  
      - `verdict` from AI response JSON content  
      - `jobUrl` from AI response JSON content  
    - Connect from Job Filter output.

13. **Create If Node (Relevant Job posting?)**  
    - Condition: `verdict` equals `"true"`  
    - True branch: to True Jobs node  
    - False branch: discard or no further action.

14. **Create Merge Node (True Jobs)**  
    - Combine relevant jobs from Filter Outlet and Relevant Job Posting? node.

15. **Create If Node (Anything to add?)**  
    - Condition: `jobUrl` exists  
    - True branch: to Find DM node  
    - False branch: to Loop Over Items1 (restart batch)

16. **Create Tavily Node (Find DM)**  
    - Query: Use template JSON to get decision maker details from corporate website and job URL  
    - Parameters: topic general, max 9 results, advanced depth  
    - Include domains: corporate website and job URL  
    - Credentials: Tavily API  
    - Connect Anything to add? True branch.

17. **Create Google Sheets Append Node (Append row in sheet)**  
    - Operation: Append row  
    - Use schema matching columns in existing sheet  
    - Map fields from True Jobs and Find DM outputs, including company CEO info, job title, location, salary, description, and URLs  
    - Credentials: Google Sheets OAuth2  
    - Connect from Find DM output.

18. **Create Wait Node**  
    - Duration: 1.5 minutes  
    - Connect from Append row in sheet output.

19. **Connect Wait node output back to Loop Over Items1** to process next batch.

20. **Add Sticky Notes** for logical block descriptions as desired.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| When the Apify actor runs, the webhook gets activated, triggering the dataset retrieval used to filter and update the database with relevant jobs.                                                                              | Sticky Note3 in workflow                                                                          |
| Future improvements: Add decision maker email discovery and personalized icebreaker generation using tools like Anymailfinder and Perplexity.                                                                                  | Sticky Note3                                                                                     |
| Filter existing job post URLs before passing data to GPT to reduce unnecessary API calls.                                                                                                                                       | Sticky Note3                                                                                     |
| Filter tuning may be required for the AI job filter to improve accuracy.                                                                                                                                                         | Job Filter Node sticky note                                                                       |
| Google Sheets document and sheet IDs must be correctly configured with proper OAuth2 credentials.                                                                                                                               | Google Sheets nodes notes                                                                         |
| Official n8n documentation for nodes used: https://docs.n8n.io/                                                                                                                                                                  | General reference                                                                                |
| For Tavily API usage and configuration: https://tavily.com                                                                                                                                                                      | Tavily Node credentials and usage                                                                |

---

This document fully describes the "Indeed Job Scraper with AI Filtering & Company Research using Apify and Tavily" workflow, enabling advanced users and AI agents to understand, reproduce, and maintain it effectively.