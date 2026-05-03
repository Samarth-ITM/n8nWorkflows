Upwork Job Listings Auto-Export to Google Sheets with Apify

https://n8nworkflows.xyz/workflows/upwork-job-listings-auto-export-to-google-sheets-with-apify-4795


# Upwork Job Listings Auto-Export to Google Sheets with Apify

---

### 1. Workflow Overview

This workflow automates the process of extracting current job listings from Upwork and logging them into a Google Sheet for easy analysis and tracking. It is designed primarily for freelancers, agencies, product developers, or market analysts who want to monitor Upwork job trends in an automated, no-code fashion.

The workflow consists of two main logical blocks:

- **1.1 Data Scraping Automation**:  
  Periodically triggers the workflow and fetches fresh job listings from Upwork by invoking an Apify actor that scrapes Upwork's website, since Upwork does not provide a public API for job listings.

- **1.2 Data Transformation & Logging**:  
  Processes and formats the raw scraped data to extract and clean relevant fields, then appends the cleaned job data into a designated Google Sheets document for storage and further analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Scraping Automation

**Overview:**  
This block is responsible for automatically triggering the data scraping process on a schedule and retrieving Upwork job listings through the Apify API.

**Nodes Involved:**  
- Check Upwork Jobs - Trigger  
- Fetch Upwork Jobs using Apify

**Node Details:**

- **Check Upwork Jobs - Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow execution at defined intervals (e.g., hourly).  
  - Configuration: Trigger interval configured by hours (default every hour).  
  - Expressions/Variables: None.  
  - Input: None (trigger node).  
  - Output: Initiates the next node in the workflow.  
  - Version Requirements: n8n version supporting scheduleTrigger v1.2 or above.  
  - Potential Failures: Misconfiguration of schedule, system time issues.  
  - Notes: Automates periodic execution, eliminating manual runs.

- **Fetch Upwork Jobs using Apify**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Apify API to run a configured actor task that scrapes Upwork job listings and returns dataset items.  
  - Configuration:  
    - URL format: `https://api.apify.com/v2/actor-tasks/<TASK_ID>/run-sync-get-dataset-items?token=<YOUR_API_TOKEN>`  
    - Method: POST  
    - Body: Empty JSON object under `parameters` to trigger the actor.  
  - Expressions/Variables: URL includes placeholder tokens requiring user replacement.  
  - Input: Trigger from Schedule node.  
  - Output: JSON array of job listing objects, each containing fields like `title`, `description`, `skills`, `postedDate`, `link`.  
  - Version Requirements: HTTP Request node v4.2 or above.  
  - Potential Failures: HTTP errors (401 Unauthorized if token invalid, 404 if TASK_ID wrong), API timeouts, network errors.  
  - Notes: Requires valid Apify task ID and API token. Apify actor must be pre-configured to scrape Upwork jobs.

---

#### 1.2 Data Transformation & Logging

**Overview:**  
This block transforms raw job data into a clean, structured format and appends it to a Google Sheets document to maintain an ongoing log of job listings.

**Nodes Involved:**  
- Format scrape Data  
- Log Jobs to Google Sheets

**Node Details:**

- **Format scrape Data**  
  - Type: Set  
  - Role: Extracts and cleans relevant fields from the raw JSON response, standardizing the data structure for downstream use.  
  - Configuration: Assigns the following fields from input JSON:  
    - `title` → string  
    - `description` → string  
    - `postedDate` → string  
    - `skills` → array  
    - `link` → string  
  - Expressions/Variables: Uses expressions like `={{ $json.title }}` to map input fields.  
  - Input: JSON array from Apify HTTP Request node.  
  - Output: Cleaned JSON objects with only the required fields.  
  - Version Requirements: Set node v3.4 or above.  
  - Potential Failures: Missing fields in input JSON, expression evaluation errors.  
  - Notes: Ensures data consistency and prepares for Google Sheets ingestion.

- **Log Jobs to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends each job entry as a new row in a specified Google Sheet.  
  - Configuration:  
    - Operation: Append  
    - Document ID: Google Sheet document ID (must be accessible with linked credentials).  
    - Sheet Name: `gid=0` (default first sheet).  
    - Columns: Maps cleaned fields to columns: `title`, `description`, `postedDate`, `skills`, `link`.  
    - Mapping Mode: Define below with explicit column mappings.  
    - Credential: Google Sheets OAuth2 account credential with access to the target sheet.  
  - Expressions/Variables: Field mappings use expressions like `={{ $json.link }}`.  
  - Input: Cleaned JSON objects from Set node.  
  - Output: None (final node).  
  - Version Requirements: Google Sheets node v4.5 or above.  
  - Potential Failures: Authentication failures, insufficient permissions, sheet not found, quota limits, invalid data types.  
  - Notes: Allows data accumulation and easy sharing or analysis in Google Sheets.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                        | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                         |
|------------------------------|---------------------|-------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| Check Upwork Jobs - Trigger   | Schedule Trigger    | Initiate workflow on schedule       | None                        | Fetch Upwork Jobs using Apify  | Automates periodic triggering of job scraping workflow, removing manual execution need.            |
| Fetch Upwork Jobs using Apify | HTTP Request       | Fetch Upwork jobs via Apify API     | Check Upwork Jobs - Trigger  | Format scrape Data             | Uses Apify API to scrape Upwork jobs, returns structured job listing data.                         |
| Format scrape Data            | Set                 | Format and clean job data            | Fetch Upwork Jobs using Apify | Log Jobs to Google Sheets      | Extracts relevant fields and prepares data for Google Sheets ingestion.                            |
| Log Jobs to Google Sheets     | Google Sheets       | Append job data to Google Sheet     | Format scrape Data            | None                          | Adds each job as a row in Google Sheets for analysis and tracking.                                |
| Sticky Note                  | Sticky Note         | Documentation and explanation       | None                        | None                          | Section 1: Data Scraping Automation - explains the purpose and nodes of the scraping section.     |
| Sticky Note1                 | Sticky Note         | Documentation and explanation       | None                        | None                          | Section 2: Data Transformation & Logging - explains the formatting and logging section.          |
| Sticky Note9                 | Sticky Note         | Workflow assistance and contact info| None                        | None                          | Provides contact info and links for workflow support by Yaron Been.                              |
| Sticky Note4                 | Sticky Note         | Extended documentation combining both sections | None                    | None                          | Comprehensive explanation of the entire workflow, combining scraping and logging sections.        |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps in n8n to recreate the workflow completely:

1. **Create the "Check Upwork Jobs - Trigger" node:**  
   - Node Type: Schedule Trigger  
   - Set the trigger interval to your desired frequency (e.g., every 1 hour).  
   - Save and position at workflow start.

2. **Create the "Fetch Upwork Jobs using Apify" node:**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apify.com/v2/actor-tasks/<TASK_ID>/run-sync-get-dataset-items?token=<YOUR_API_TOKEN>` (replace `<TASK_ID>` and `<YOUR_API_TOKEN>` with your actual Apify actor task ID and API token)  
   - Body Parameters: Set to send an empty JSON object `{}` under parameters.  
   - Connect this node’s input to the Schedule Trigger node’s output.

3. **Create the "Format scrape Data" node:**  
   - Node Type: Set  
   - Add fields with assignment as follows:  
     - `title`: `={{ $json.title }}` (string)  
     - `description`: `={{ $json.description }}` (string)  
     - `postedDate`: `={{ $json.postedDate }}` (string)  
     - `skills`: `={{ $json.skills }}` (array)  
     - `link`: `={{ $json.link }}` (string)  
   - Connect this node’s input to the HTTP Request node’s output.

4. **Create the "Log Jobs to Google Sheets" node:**  
   - Node Type: Google Sheets  
   - Operation: Append  
   - Document ID: Paste your Google Sheets document ID (the long string in your Google Sheets URL).  
   - Sheet Name: Use `gid=0` or the exact sheet name or ID where you want to log the data.  
   - Columns Mapping:  
     - `title` → `={{ $json.title }}`  
     - `description` → `={{ $json.description }}`  
     - `postedDate` → `={{ $json.postedDate }}`  
     - `skills` → `={{ $json.skills }}` (ensure this is converted to string if needed)  
     - `link` → `={{ $json.link }}`  
   - Set up Google Sheets OAuth2 credentials with access permissions to the target sheet.  
   - Connect this node’s input to the Set node’s output.

5. **Validate and activate the workflow:**  
   - Test the workflow manually or wait for the scheduled trigger.  
   - Monitor execution for errors (auth issues, API errors).  
   - Adjust Apify actor filters or Google Sheets permissions as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                    | Context or Link                                              |
|------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow scrapes Upwork jobs using Apify because Upwork lacks a public jobs API.          | [Apify platform](https://apify.com)                          |
| You can customize your Apify actor to filter by keywords like "AI", "Python", or "Design".     | Apify actor configuration                                    |
| Google Sheets is used for easy visualization, sharing, and further data processing.             | Google Sheets documentation                                  |
| Add additional features like deduplication, alerts, or integration with Airtable or Telegram.  | Workflow extension ideas                                     |
| Workflow support and tips by Yaron Been:                                                        | Email: Yaron@nofluff.online                                  |
| YouTube tutorials and professional tips:                                                       | https://www.youtube.com/@YaronBeen/videos                     |
| LinkedIn profile for professional contact:                                                     | https://www.linkedin.com/in/yaronbeen/                        |

---

**Disclaimer**: This text is derived exclusively from an automated workflow created with n8n, a no-code automation tool. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---