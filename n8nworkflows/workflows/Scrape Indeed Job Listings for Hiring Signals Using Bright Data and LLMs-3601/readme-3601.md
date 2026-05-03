Scrape Indeed Job Listings for Hiring Signals Using Bright Data and LLMs

https://n8nworkflows.xyz/workflows/workflow-3601-1747220158574.png


# Scrape Indeed Job Listings for Hiring Signals Using Bright Data and LLMs

### 1. Workflow Overview

This workflow automates the process of scraping Indeed job listings using Bright Data’s scraping API, then analyzing each job posting via a Large Language Model (LLM) to determine if the user is a good fit based on custom prompts. It targets job seekers or recruiters wanting to quickly identify relevant hiring signals from fresh Indeed job posts.

The workflow consists of these logical blocks:

- **1.1 Form Input Reception:** Collects user inputs for job location, keywords, and country via an n8n form.
- **1.2 Bright Data Scraper Trigger & Polling:** Initiates a scraping snapshot on Bright Data with user parameters, then polls until the scraping is complete.
- **1.3 Fetch and Store Raw Job Data:** Retrieves scraped job posts from Bright Data and appends them to a Google Sheet for record keeping.
- **1.4 Job Post Processing & Analysis with LLM:** Splits job post data, sends each job description to an LLM chain to assess fit, and writes the "YES" or "NO" fit results back to the Google Sheet adjacent to each job.

The workflow integrates external services requiring credentials: Bright Data API, Google Sheets API, and OpenAI API for GPT-4o-mini or similar LLM.

---

### 2. Block-By-Block Analysis

#### 2.1 Form Input Reception

**Overview:**  
This block collects input parameters from a user-submitted form that specifies their job hunting criteria (location, keyword, country code).

**Nodes Involved:**  
- On form submission - Discover Jobs

**Node Details:**  
- **On form submission - Discover Jobs**  
  - Type: Form Trigger  
  - Role: Captures user inputs dynamically to serve as parameters for the scraping request.  
  - Configuration: Form fields: Job Location (city or region), Keyword (role or skills), Country (2-letter ISO code). All required fields.  
  - Outputs: Triggers the next node with form data in JSON form.  
  - Edge Cases: Ensure valid inputs; missing or invalid country codes or empty fields will cause the workflow to malfunction.

---

#### 2.2 Bright Data Scraper Trigger & Polling

**Overview:**  
Posts a request to Bright Data’s API to initiate a scraping snapshot for Indeed job listings with user-supplied parameters, then periodically polls for the snapshot’s progress until completion.

**Nodes Involved:**
- HTTP Request- Post API call to Bright Data
- Wait - Polling Bright Data
- Snapshot Progress
- If - Checking status of Snapshot - if data is ready or not

**Node Details:**

- **HTTP Request- Post API call to Bright Data**  
  - Type: HTTP Request (POST)  
  - Role: Initiates scraping snapshot with Bright Data dataset, passing filters such as country, domain, keywords, location, and date posted (hardcoded as "Last 24 hours").  
  - Key expressions: Filters use expressions to inherit form data (`{{$json['Country (2 letters)']}}` etc).  
  - Potential failures: API key authentication errors, rate limits, invalid parameters, network timeouts.  
  - Outputs: Snapshot ID for polling.

- **Wait - Polling Bright Data**  
  - Type: Wait  
  - Role: Waits 1 minute intervals between status checks to avoid API flooding.  
  - Config: Waits for 1 minute (modifiable).  
  - Edge cases: Timing too short may trigger rate limits or unnecessary hits; too long delays workflow completion.

- **Snapshot Progress**  
  - Type: HTTP Request (GET)  
  - Role: Checks the scraping snapshot status by querying Bright Data with the snapshot ID.  
  - Header: Authorization with Bright Data API key required.  
  - Outputs: JSON payload including snapshot status (e.g., "running", "finished").

- **If - Checking status of Snapshot - if data is ready or not**  
  - Type: If Condition  
  - Role: Decides continuation based on snapshot status—loops back to wait if "running", or proceeds when ready.  
  - Condition: Checks if `status == "running"` to keep polling.  
  - Edge cases: If snapshot status is unexpected or errors occur, workflow could get stuck or terminate prematurely. No error branch defined explicitly.

---

#### 2.3 Fetch and Store Raw Job Data

**Overview:**  
Once the snapshot is ready, downloads the scraped job data in JSON format, then appends all job postings into a Google Sheet as raw records.

**Nodes Involved:**
- HTTP Request - Getting data from Bright Data
- Google Sheets - Adding All Job Posts

**Node Details:**

- **HTTP Request - Getting data from Bright Data**  
  - Type: HTTP Request (GET)  
  - Role: Fetches snapshot data from Bright Data’s API using the snapshot ID acquired earlier.  
  - Header: Bright Data API key required.  
  - Query Parameter: format=json to get structured JSON data.  
  - Output: JSON array of job posts with various fields (company_name, job_title, description_text, etc.).  
  - Failure modes: API errors, invalid snapshot IDs, empty results.

- **Google Sheets - Adding All Job Posts**  
  - Type: Google Sheets Append  
  - Role: Inserts all fetched job posts as new rows in a Google Sheet template designed to hold this data.  
  - Config: Auto-maps all relevant job post fields to predefined columns in the Google Sheet.  
  - Credentials: OAuth2 Google Sheets API credentials required.  
  - Edge cases: Sheet full or protected, Google API limits, credential expiration.  
  - Sticky Note included with a link to the recommended Sheet template for ease of setup.

---

#### 2.4 Job Post Processing & Analysis with LLM

**Overview:**  
Processes each job post row individually by splitting composite data fields, then runs an LLM chain (GPT-4o-mini) prompt to assess if the user is a good fit, finally updates the Google Sheet with the "YES" or "NO" results.

**Nodes Involved:**  
- Split Out  
- Basic LLM Chain - Checking Fit  
- Google Sheets (Update Fit Result)  
- OpenAI Chat Model (linked, but not actively used in main flow)  
- Sticky Notes explaining logic and setup

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Role: Splits incoming job post arrays into individual job/company/job title/description records for separate processing.  
  - Field to split: `company_name, job_title, description_text` (comma-separated fields treated as split points).  
  - Output: One item per job for downstream LLM analysis.

- **Basic LLM Chain - Checking Fit**  
  - Type: Langchain LLM Chain (OpenAI GPT-based)  
  - Role: Sends a prompt constructed dynamically from job post data to the LLM asking if the user is a good fit (response constrained to "YES" or "NO").  
  - Prompt example:  
    ```
    Read the following job post:
    Company Name {{ company_name }}, job Title: {{ job_title }},
    And job description {{ description_text }}, and tell me if you think I'm a good fit. Answer only YES or NO.
    I'm looking for roles in Pfizer
    ```  
  - Credentials: OpenAI API key necessary  
  - Edge cases: API key errors, LLM rate limits, ambiguous responses, or responses other than YES/NO that might confuse the downstream update.

- **Google Sheets (Update Fit Result)**  
  - Type: Google Sheets Update  
  - Role: Updates existing Google Sheet rows by matching `company_name` and sets "AM I a Fit?" column to the LLM's "YES" or "NO" answer.  
  - Mapping: Updates only `company_name` and "AM I a Fit?" columns, preserving other data.  
  - Credentials: Same Google Sheets OAuth2 credentials as append node.  
  - Edge cases: Matching errors if company names are inconsistent, concurrent update conflicts, API limits.

---

### 3. Summary Table

| Node Name                            | Node Type                       | Functional Role                                  | Input Node(s)                           | Output Node(s)                         | Sticky Note                                                  |
|------------------------------------|--------------------------------|-------------------------------------------------|---------------------------------------|---------------------------------------|--------------------------------------------------------------|
| Sticky Note9                       | Sticky Note                    | Workflow assistance and contact info             | -                                     | -                                     | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/; Bright Data Docs: https://docs.brightdata.com/introduction; Reminder to add API keys. |
| On form submission - Discover Jobs | Form Trigger                   | Receives user input form for job query            | -                                     | HTTP Request- Post API call to Bright Data |                                                              |
| HTTP Request- Post API call to Bright Data | HTTP Request (POST)            | Triggers Bright Data scraping snapshot             | On form submission - Discover Jobs    | Wait - Polling Bright Data             | Sticky Note2 explains customization of query filters         |
| Wait - Polling Bright Data         | Wait                          | Waits 1 minute before polling status again         | HTTP Request- Post API call to Bright Data | Snapshot Progress                     | Sticky Note3 content: "Bright Data Getting Jobs"              |
| Snapshot Progress                  | HTTP Request (GET)             | Checks the progress/status of Bright Data snapshot | Wait - Polling Bright Data             | If - Checking status of Snapshot - if data is ready or not |                                                              |
| If - Checking status of Snapshot - if data is ready or not | If Condition                   | Determines whether to continue polling or fetch data | Snapshot Progress                     | Wait - Polling Bright Data (if running), HTTP Request - Getting data from Bright Data (if ready) |                                                              |
| HTTP Request - Getting data from Bright Data | HTTP Request (GET)             | Retrieves scraped job data JSON snapshot             | If - Checking status of Snapshot       | Google Sheets - Adding All Job Posts   |                                                              |
| Google Sheets - Adding All Job Posts | Google Sheets Append           | Appends all job posts data into Google Sheet       | HTTP Request - Getting data from Bright Data | Split Out                            | Sticky Note10 with Google Sheets template link and setup instructions |
| Split Out                         | Split Out                      | Splits job posts into individual items for analysis | Google Sheets - Adding All Job Posts   | Basic LLM Chain - Checking Fit         | Sticky Note: Explains processing each job post with LLM       |
| Basic LLM Chain - Checking Fit    | Chain LLM (Langchain OpenAI)  | Runs LLM prompt to check if user fits job post     | Split Out                            | Google Sheets (Update Fit Result)      | Sticky Note: "Checking if each job post is relevant to you"    |
| Google Sheets                    | Google Sheets Update            | Updates "AM I a Fit?" column in Google Sheet       | Basic LLM Chain - Checking Fit          | -                                     |                                                              |
| Sticky Note1                     | Sticky Note                    | Indeed Jobs API Parameter Guide                      | -                                     | -                                     | Detailed parameter guide and examples for Bright Data API      |
| Sticky Note2                     | Sticky Note                    | Bright Data Trigger customization info              | -                                     | -                                     | See summary above                                           |
| Sticky Note3                     | Sticky Note                    | Indicates Bright Data job retrieval step             | -                                     | -                                     | See summary above                                           |
| Sticky Note10                    | Sticky Note                    | Google Sheets usage instructions with template link | -                                     | -                                     | See summary above                                           |
| Sticky Note                      | Sticky Note                    | Explains relevance checking step with LLM            | -                                     | -                                     | See summary above                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node** named `On form submission - Discover Jobs`:  
   - Set form title and description.  
   - Add required form fields: "Job Location", "Keyword", "Country (2 letters)" (all text).  
   - Save webhook ID for form submissions.

2. **Add an HTTP Request Node** named `HTTP Request- Post API call to Bright Data`:  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Headers: Authorization Bearer token (`<YOUR_BRIGHT_DATA_API_KEY>`)  
   - Body (JSON): Use expressions to insert form field data into Bright Data filters including `country`, `domain: "indeed.com"`, `keyword_search`, `location`, `date_posted` set to `"Last 24 hours"`.  
   - Query Parameters: Include `dataset_id=gd_l4dx9j9sscpvs7no2` and parameters for snapshot settings (`include_errors`, `type`, `discover_by`, `uncompressed_webhook`).

3. **Connect above node to a Wait Node** named `Wait - Polling Bright Data`:  
   - Wait duration: 1 minute (adjustable).

4. **Create an HTTP Request Node** named `Snapshot Progress`:  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (use snapshot_id from previous node)  
   - Headers: Bright Data API key authorization.

5. **Create an If Node** named `If - Checking status of Snapshot - if data is ready or not`:  
   - Check if `status` from previous node JSON equals `"running"`.  
   - True: Loop back to `Wait - Polling Bright Data` to wait and recheck.  
   - False: Proceed to next step.

6. **Add an HTTP Request Node** named `HTTP Request - Getting data from Bright Data`:  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}?format=json`  
   - Headers: Same Bright Data API Key.

7. **Add a Google Sheets Node** named `Google Sheets - Adding All Job Posts`:  
   - Operation: Append rows  
   - Select your Google Sheets document and Sheet name (gid=0) with the template having these columns aligned.  
   - Map all fields automatically from snapshot JSON.

8. **Add a Split Out Node** named `Split Out`:  
   - Field to split out: `company_name, job_title, description_text`.  
   - This splits all job posts into individual units for separate LLM analysis.

9. **Add a Langchain Chain LLM Node** named `Basic LLM Chain - Checking Fit`:  
   - Model: GPT-4o-mini or any supported GPT model you prefer.  
   - Prompt: Use dynamic expressions to input company_name, job_title, and description_text. Request binary answer: YES or NO if user is a fit.  
   - Credentials: Setup OpenAI API key.

10. **Add a Google Sheets Node** named `Google Sheets` (Update Fit Result):  
    - Operation: Update rows  
    - Match on `company_name` column to find rows to update.  
    - Update `"AM I a Fit?"` column with LLM text result.

11. **Connectivity:** Link nodes as follows:  
    - Form Trigger → Post API call → Wait → Snapshot Progress → If → Wait or Get Data → Add All Job Posts → Split Out → LLM Chain → Google Sheets Update

12. **Credentials Setup:**  
    - Add your Bright Data API Key to all HTTP Request nodes that interact with Bright Data.  
    - Add Google Sheets OAuth2 credentials in all Google Sheets nodes.  
    - Add OpenAI credentials to the Langchain Chain LLM node.

13. **Set Sticky Notes:** For ease of use, add sticky notes with instructions on where to add API keys, links to Google Sheets template (https://docs.google.com/spreadsheets/d/1vHHNShHD96AWsPnbXzlDAhPg_DbXr_Yx3wsAnQEtuyU), and tips for querying Bright Data filters.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow Assistance and Contact info for support                                                    | Email: Yaron@nofluff.online                                   |
| Video tutorials and workflow explanations                                                          | YouTube channel: https://www.youtube.com/@YaronBeen/videos    |
| Professional profile and support                                                                   | LinkedIn: https://www.linkedin.com/in/yaronbeen/              |
| Official Bright Data documentation                                                                 | https://docs.brightdata.com/introduction                       |
| Recommended Google Sheets template for job data                                                    | https://docs.google.com/spreadsheets/d/1vHHNShHD96AWsPnbXzlDAhPg_DbXr_Yx3wsAnQEtuyU/edit?usp=sharing |
| Tips to narrow Bright Data scraping filters for best efficiency and cost control                    | Use “Last 24 hours” filter often; narrow keywords             |
| Important: Add all API credentials to HTTP Request nodes (Bright Data), Google Sheets, and OpenAI | Reminder from Sticky Note9                                      |

---

This structured reference provides a comprehensive technical understanding of the workflow, functional modules for analysis or modification, a detailed node catalog, and stepwise instructions for full manual reconstruction in n8n. It anticipates common failure points and sets clear integration requirements for third-party APIs and services.