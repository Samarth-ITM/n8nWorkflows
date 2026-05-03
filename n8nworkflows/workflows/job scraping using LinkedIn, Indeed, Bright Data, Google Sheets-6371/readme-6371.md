job scraping using LinkedIn, Indeed, Bright Data, Google Sheets

https://n8nworkflows.xyz/workflows/job-scraping-using-linkedin--indeed--bright-data--google-sheets-6371


# job scraping using LinkedIn, Indeed, Bright Data, Google Sheets

### 1. Workflow Overview

This workflow automates job data scraping from two major platforms, LinkedIn and Indeed, using Bright Data's scraping API, and consolidates results into a Google Sheets document for further review and comparison. It is designed for users who want to search for jobs by specifying parameters such as city, job title, country, and job type through a form interface.

**Target Use Cases:**
- Automating job market research by aggregating listings from multiple job boards.
- Streamlining job search data collection for recruitment or market analysis.
- Exporting scraped job data into a Google Sheet for easy access, filtering, and comparison.

**Logical Blocks:**

- **1.1 Input Reception:** Trigger the workflow via a user-submitted web form collecting job search parameters.
- **1.2 Input Formatting:** Prepare and format the input parameters specifically for Indeed and LinkedIn API requests.
- **1.3 Indeed Scraping:** Trigger job scraping for Indeed jobs via Bright Data API, check job scraping progress, and fetch results once ready.
- **1.4 LinkedIn Scraping:** Trigger job scraping for LinkedIn jobs via Bright Data API, check job scraping progress, and fetch results once ready.
- **1.5 Data Consolidation and Export:** Merge results from both sources and export combined job data to a designated Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures user input from a web form to start the job scraping process.
- **Nodes Involved:**  
  - Triggers workflow from job search form  
  - Sticky Note1

- **Node Details:**  
  - **Triggers workflow from job search form**  
    - Type: Form Trigger  
    - Role: Entry point capturing user inputs: City (required), Job Title (required), Country (required), and optional Job Type dropdown.  
    - Configuration: Form titled "Job Finder" with required text fields and a dropdown for job types (Full-Time, Part-Time, Remote, WFH, Contract, Internship, Freelance).  
    - Input connections: None (trigger node)  
    - Output connections: Passes data to "Formats form input for Bright Data Indeed API1" and "Triggers job scraping on Indeed via Bright Data"  
    - Edge cases: Missing required fields block form submission; invalid or empty job type handled downstream.  
    - Version: 2.2

- **Sticky Note1**: Annotates that this node triggers on user input with job title, city, country, and job type.

#### 1.2 Input Formatting

- **Overview:** Processes and formats the form input to prepare for the Indeed and LinkedIn Bright Data API requests.
- **Nodes Involved:**  
  - Formats form input for Bright Data Indeed API1  
  - Sticky Note2

- **Node Details:**  
  - **Formats form input for Bright Data Indeed API1**  
    - Type: Code (JavaScript)  
    - Role: Constructs the JSON input payload for Indeed scraping API.  
    - Configuration: Extracts City, Job Title, Country, Job Type from form data; sets domain based on country (fr.indeed.com for France, else indeed.com); sets date_posted to "Last 7 days"; defines custom output fields to be requested from the API.  
    - Key expressions: Uses `$json["City"]`, `$json["Job Title"]`, `$json["Country"]`, `$json["Job_type (Optional)"]`; returns formatted JSON object.  
    - Input connection: From form trigger  
    - Output connection: To "Triggers job scraping on LinkedIn via Bright Data"  
    - Edge cases: Missing city, job title, or country results in empty strings; job type optional, no validation beyond presence.  
    - Version: 2

- **Sticky Note2**: Indicates this node formats input specifically for Indeed scraping.

#### 1.3 Indeed Scraping

- **Overview:** Handles triggering Indeed scraping via Bright Data, monitoring job scraping progress, waiting if necessary, fetching results, and verifying data availability.
- **Nodes Involved:**  
  - Triggers job scraping on Indeed via Bright Data  
  - Checks if Bright Data has completed Indeed scraping  
  - IF node: Is Indeed data ready?  
  - Pauses for 1 minute if data is not yet ready  
  - Checks if any Indeed job records exist  
  - Fetches scraped Indeed data using snapshot ID  
  - Sticky Notes 3, 4, 5, 6, 7, 8

- **Node Details:**  
  - **Triggers job scraping on Indeed via Bright Data**  
    - Type: HTTP Request  
    - Role: Initiates scraping job on Indeed dataset via Bright Data API.  
    - Configuration: POST to Bright Data datasets/v3/trigger endpoint with JSON body containing user input and requested output fields; dataset_id parameter targets Indeed dataset; includes authorization header with Bright Data API Key.  
    - Input: From form trigger (directly uses form fields for parameters)  
    - Output: Contains snapshot_id for progress tracking  
    - Edge cases: API auth failure, rate limits, invalid parameters.

  - **Checks if Bright Data has completed Indeed scraping**  
    - Type: HTTP Request  
    - Role: Polls Bright Data API for scraping job progress status using snapshot_id.  
    - Configuration: GET request with Authorization header, queries status endpoint.  
    - Input: snapshot_id from previous node  
    - Output: JSON containing status (e.g., "ready")  
    - Edge cases: Network errors, invalid snapshot_id, API downtime.

  - **IF node: Is Indeed data ready?**  
    - Type: If node  
    - Role: Branches workflow based on whether Indeed scraping status is "ready".  
    - Input: Status from progress check node  
    - Output: If ready → proceed to check records; else → wait.

  - **Pauses for 1 minute if data is not yet ready**  
    - Type: Wait  
    - Role: Delays workflow for 1 minute before rechecking scraping progress.  
    - Input: From IF node (not ready branch)  
    - Output: Loops back to progress check node.

  - **Checks if any Indeed job records exist**  
    - Type: If node  
    - Role: Validates that scraped data contains at least one job record.  
    - Input: From progress check node output (records count)  
    - Output: If records found → fetch data; else → workflow stops or handles no data case.

  - **Fetches scraped Indeed data using snapshot ID**  
    - Type: HTTP Request  
    - Role: Retrieves scraped job listings from Bright Data using snapshot_id.  
    - Configuration: GET request to Bright Data snapshot endpoint with Authorization header.  
    - Input: snapshot_id from earlier trigger  
    - Output: JSON array of job listings  
    - Edge cases: Data not found, API errors.

- **Sticky Notes 3 to 8**: Annotate steps for Indeed API call, status check, wait/retry cycle, data existence check, and data fetching.

#### 1.4 LinkedIn Scraping

- **Overview:** Mirrors the Indeed scraping process for LinkedIn job listings through Bright Data API, including triggering scrape, status checking, waiting, verifying results, and fetching job data.
- **Nodes Involved:**  
  - Triggers job scraping on LinkedIn via Bright Data  
  - Checks if LinkedIn scraping is completed  
  - IF node: Is LinkedIn data ready?  
  - Waits before rechecking LinkedIn scraping  
  - Checks if any LinkedIn job records exist  
  - Fetches scraped LinkedIn data using snapshot ID  
  - Sticky Notes 9, 10, 11, 12, 13, 14

- **Node Details:**  
  - **Triggers job scraping on LinkedIn via Bright Data**  
    - Type: HTTP Request  
    - Role: Initiates LinkedIn job scraping with formatted input from the Indeed input formatting node.  
    - Configuration: POST to Bright Data API with input JSON and requested output fields; dataset_id corresponds to LinkedIn dataset; authorization included.  
    - Input: From "Formats form input for Bright Data Indeed API1" node output  
    - Output: snapshot_id for progress tracking.

  - **Checks if LinkedIn scraping is completed**  
    - Type: HTTP Request  
    - Role: Polls the status of LinkedIn scraping job using snapshot_id.  
    - Configuration: GET request with Authorization header.  
    - Input: snapshot_id from trigger node  
    - Output: Status JSON.

  - **IF node: Is LinkedIn data ready?**  
    - Type: If node  
    - Role: Branches workflow based on LinkedIn scraping status.  
    - Input: Status from progress check  
    - Output: If ready → check records; else → wait.

  - **Waits before rechecking LinkedIn scraping**  
    - Type: Wait  
    - Role: Delays 1 minute before rechecking LinkedIn scrape progress.  
    - Input: From IF node (not ready branch)  
    - Output: Loops back to progress check.

  - **Checks if any LinkedIn job records exist**  
    - Type: If node  
    - Role: Validates presence of job records in scraped LinkedIn data.  
    - Input: Records count from status node  
    - Output: If records found → fetch data.

  - **Fetches scraped LinkedIn data using snapshot ID**  
    - Type: HTTP Request  
    - Role: Retrieves LinkedIn job listings from Bright Data snapshot endpoint.  
    - Input: snapshot_id  
    - Output: Job data JSON array.

- **Sticky Notes 9 to 14**: Describe API calls, status checks, waiting, data availability checks, and data fetching steps for LinkedIn.

#### 1.5 Data Consolidation and Export

- **Overview:** Combines job listings from Indeed and LinkedIn, then appends them to a Google Sheets document for comparison and further analysis.
- **Nodes Involved:**  
  - Combines Indeed + LinkedIn job results  
  - Saves final job list to Google Sheet  
  - Sticky Notes 15, 16

- **Node Details:**  
  - **Combines Indeed + LinkedIn job results**  
    - Type: Merge  
    - Role: Merges arrays of job listings from both sources into a single dataset.  
    - Configuration: Default merge mode (likely append).  
    - Inputs:  
      - Input 1: Indeed job data from fetch node  
      - Input 2: LinkedIn job data from fetch node  
    - Output: Combined job listings.

  - **Saves final job list to Google Sheet**  
    - Type: Google Sheets  
    - Role: Appends combined job listings to a specific sheet named "Compare".  
    - Configuration:  
      - Document ID and Sheet ID configured to a Google Sheets document accessible via OAuth2 credentials.  
      - Columns mapped with keys such as Salary, Location, Job Title, Job Type, Apply Link, Job Detail, Company Name.  
      - Append operation to add new rows.  
    - Input: Combined job listings from merge node.  
    - Credentials: Uses Google Sheets OAuth2 credentials.  
    - Edge cases: Credential expiration, API limits, data mapping errors.

- **Sticky Notes 15 and 16**: Highlight the merge operation and saving data to the sheet.

---

### 3. Summary Table

| Node Name                                    | Node Type             | Functional Role                         | Input Node(s)                                | Output Node(s)                                    | Sticky Note                                         |
|----------------------------------------------|-----------------------|---------------------------------------|----------------------------------------------|--------------------------------------------------|----------------------------------------------------|
| Triggers workflow from job search form        | Form Trigger           | Entry point capturing search input    | None                                         | Formats form input for Bright Data Indeed API1, Triggers job scraping on Indeed via Bright Data | 📝 Trigger – User fills job title, city, country, job type |
| Sticky Note1                                  | Sticky Note            | Annotation                           | None                                         | None                                             | 📝 Trigger – User fills job title, city, country, job type |
| Formats form input for Bright Data Indeed API1 | Code                   | Formats input for Indeed API          | Triggers workflow from job search form       | Triggers job scraping on LinkedIn via Bright Data | 🧠 Format Input – Prepares input for Indeed scraping |
| Sticky Note2                                  | Sticky Note            | Annotation                           | None                                         | None                                             | 🧠 Format Input – Prepares input for Indeed scraping |
| Triggers job scraping on Indeed via Bright Data | HTTP Request           | Initiates Indeed scraping             | Triggers workflow from job search form       | Checks if Bright Data has completed Indeed scraping | 📤 Indeed API Call – Start job search on Indeed     |
| Sticky Note3                                  | Sticky Note            | Annotation                           | None                                         | None                                             | 📤 Indeed API Call – Start job search on Indeed     |
| Checks if Bright Data has completed Indeed scraping | HTTP Request           | Checks scraping progress status       | Triggers job scraping on Indeed via Bright Data | IF node: Is Indeed data ready?                    | ⏳ Check Indeed Status – Is Indeed data ready?       |
| Sticky Note4                                  | Sticky Note            | Annotation                           | None                                         | None                                             | ⏳ Check Indeed Status – Is Indeed data ready?       |
| IF node: Is Indeed data ready?                 | If                     | Branches on scraping status           | Checks if Bright Data has completed Indeed scraping | Checks if any Indeed job records exist, Pauses for 1 minute if data is not yet ready | ✅ Is Indeed Ready? – Proceed or wait                |
| Sticky Note5                                  | Sticky Note            | Annotation                           | None                                         | None                                             | ✅ Is Indeed Ready? – Proceed or wait                |
| Pauses for 1 minute if data is not yet ready  | Wait                   | Waits before rechecking status        | IF node: Is Indeed data ready?                | Checks if Bright Data has completed Indeed scraping | ⏱️ Wait – Retry check after 1 minute                 |
| Sticky Note6                                  | Sticky Note            | Annotation                           | None                                         | None                                             | ⏱️ Wait – Retry check after 1 minute                 |
| Checks if any Indeed job records exist         | If                     | Validates existence of Indeed data    | IF node: Is Indeed data ready?                 | Fetches scraped Indeed data using snapshot ID     | 📊 Data Found? – Continue only if jobs found on Indeed |
| Sticky Note7                                  | Sticky Note            | Annotation                           | None                                         | None                                             | 📊 Data Found? – Continue only if jobs found on Indeed |
| Fetches scraped Indeed data using snapshot ID | HTTP Request           | Retrieves Indeed job listings         | Checks if any Indeed job records exist        | Combines Indeed + LinkedIn job results             | 📥 Get Indeed Jobs – Fetch results from Bright Data  |
| Sticky Note8                                  | Sticky Note            | Annotation                           | None                                         | None                                             | 📥 Get Indeed Jobs – Fetch results from Bright Data  |
| Triggers job scraping on LinkedIn via Bright Data | HTTP Request           | Initiates LinkedIn scraping           | Formats form input for Bright Data Indeed API1 | Checks if LinkedIn scraping is completed           | 📤 LinkedIn API Call – Start job search on LinkedIn  |
| Sticky Note9                                  | Sticky Note            | Annotation                           | None                                         | None                                             | 📤 LinkedIn API Call – Start job search on LinkedIn  |
| Checks if LinkedIn scraping is completed       | HTTP Request           | Checks LinkedIn scraping status       | Triggers job scraping on LinkedIn via Bright Data | IF node: Is LinkedIn data ready?                    | ⏳ Check LinkedIn Status – Is LinkedIn data ready?   |
| Sticky Note10                                 | Sticky Note            | Annotation                           | None                                         | None                                             | ⏳ Check LinkedIn Status – Is LinkedIn data ready?   |
| IF node: Is LinkedIn data ready?                | If                     | Branches on LinkedIn scraping status | Checks if LinkedIn scraping is completed       | Checks if any LinkedIn job records exist, Waits before rechecking LinkedIn scraping | ✅ Is LinkedIn Ready? – Proceed or wait              |
| Sticky Note11                                 | Sticky Note            | Annotation                           | None                                         | None                                             | ✅ Is LinkedIn Ready? – Proceed or wait              |
| Waits before rechecking LinkedIn scraping       | Wait                   | Waits before retrying LinkedIn status | IF node: Is LinkedIn data ready? (false branch) | Checks if LinkedIn scraping is completed           | ⏱️ Wait – Retry LinkedIn status after 1 min          |
| Sticky Note12                                 | Sticky Note            | Annotation                           | None                                         | None                                             | ⏱️ Wait – Retry LinkedIn status after 1 min          |
| Checks if any LinkedIn job records exist        | If                     | Validates presence of LinkedIn data  | IF node: Is LinkedIn data ready? (true branch) | Fetches scraped LinkedIn data using snapshot ID    | 📊 LinkedIn Data Found? – If yes, continue           |
| Sticky Note13                                 | Sticky Note            | Annotation                           | None                                         | None                                             | 📊 LinkedIn Data Found? – If yes, continue           |
| Fetches scraped LinkedIn data using snapshot ID | HTTP Request           | Retrieves LinkedIn job listings       | Checks if any LinkedIn job records exist       | Combines Indeed + LinkedIn job results             | 📥 Get LinkedIn Jobs – Fetch LinkedIn job listings   |
| Sticky Note14                                 | Sticky Note            | Annotation                           | None                                         | None                                             | 📥 Get LinkedIn Jobs – Fetch LinkedIn job listings   |
| Combines Indeed + LinkedIn job results          | Merge                   | Merges job results from both sources | Fetches scraped Indeed data using snapshot ID, Fetches scraped LinkedIn data using snapshot ID | Saves final job list to Google Sheet                 | 🔗 Merge – Combine both job sources                   |
| Sticky Note15                                 | Sticky Note            | Annotation                           | None                                         | None                                             | 🔗 Merge – Combine both job sources                   |
| Saves final job list to Google Sheet             | Google Sheets           | Appends combined job data to sheet   | Combines Indeed + LinkedIn job results          | None                                             | 📄 Save to Sheet – Add job data to “Compare” sheet  |
| Sticky Note16                                 | Sticky Note            | Annotation                           | None                                         | None                                             | 📄 Save to Sheet – Add job data to “Compare” sheet  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node**  
   - Name: `Triggers workflow from job search form`  
   - Configure Form Title: "Job Finder"  
   - Add required fields:  
     - City (text, required)  
     - Job Title (text, required)  
     - Country (text, required)  
   - Add optional dropdown field:  
     - Job_type (Optional) with options: Full-Time, Part-Time, Remote, WFH, Contract, Internship, Freelance  
   - Save and activate webhook.

3. **Add a Code node for formatting Indeed input**  
   - Name: `Formats form input for Bright Data Indeed API1`  
   - Add JavaScript code to extract form input fields and construct JSON payload with:  
     - country  
     - domain (conditional: "fr.indeed.com" if country is "FR", else "indeed.com")  
     - keyword_search (job title)  
     - location (city)  
     - date_posted: "Last 7 days"  
     - custom_output_fields array specifying all desired fields listed in the original code.  
   - Connect the Form Trigger node's output to this node's input.

4. **Add HTTP Request node to trigger Indeed scraping**  
   - Name: `Triggers job scraping on Indeed via Bright Data`  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Query parameters:  
     - dataset_id: `gd_lpfll7v5hcqtkxl6l`  
     - include_errors: `true`  
     - type: `discover_new`  
     - discover_by: `keyword`  
     - limit_per_input: `2`  
   - Headers:  
     - Authorization: `Bearer BRIGHT_DATA_API_KEY` (replace with actual credential)  
   - Body type: JSON  
   - Body: Construct JSON with `location`, `keyword`, `country`, `time_range` "Past week", and `job_type` if valid from form input.  
   - Connect Form Trigger node output to this node's input as well.

5. **Add HTTP Request node to check Indeed scraping progress**  
   - Name: `Checks if Bright Data has completed Indeed scraping`  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Query: `format=json`  
   - Header: Authorization as above  
   - Connect output of Indeed trigger node to this node.

6. **Add If node to check if Indeed data is ready**  
   - Name: `IF node: Is Indeed data ready?`  
   - Condition: `$json.status === "ready"`  
   - Connect progress check node output to this node.

7. **Add Wait node for retry delay**  
   - Name: `Pauses for 1 minute if data is not yet ready`  
   - Wait for 1 minute  
   - Connect If node's false output to this node.  
   - Connect Wait node output back to the progress check node to create a polling loop.

8. **Add If node to check if Indeed data contains records**  
   - Name: `Checks if any Indeed job records exist`  
   - Condition: `$json.records !== 0`  
   - Connect If node's true branch to next step, false branch can stop or notify no data.

9. **Add HTTP Request node to fetch Indeed job data**  
   - Name: `Fetches scraped Indeed data using snapshot ID`  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query: `format=json`  
   - Header: Authorization as above  
   - Connect If node's true output here.

10. **Add HTTP Request node to trigger LinkedIn scraping**  
    - Name: `Triggers job scraping on LinkedIn via Bright Data`  
    - Method: POST  
    - URL: `https://api.brightdata.com/datasets/v3/trigger`  
    - Query parameters:  
      - dataset_id: `gd_l4dx9j9sscpvs7no2`  
      - include_errors: `true`  
      - type: `discover_new`  
      - discover_by: `keyword`  
      - limit_per_input: `2`  
    - Headers: Authorization as above  
    - Body: Use the JSON output from the Indeed input formatting node for `input` and `custom_output_fields`.  
    - Connect output of formatting node to this node.

11. **Add HTTP Request node to check LinkedIn scraping progress**  
    - Name: `Checks if LinkedIn scraping is completed`  
    - Method: GET  
    - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
    - Query: `format=json`  
    - Header: Authorization as above  
    - Connect LinkedIn trigger node output to this node.

12. **Add If node to check if LinkedIn data is ready**  
    - Name: `IF node: Is LinkedIn data ready?`  
    - Condition: `$json.status === "ready"`  
    - Connect progress check node output to this node.

13. **Add Wait node for LinkedIn retry delay**  
    - Name: `Waits before rechecking LinkedIn scraping`  
    - Wait for 1 minute  
    - Connect If node's false output to this wait node, then loop back to LinkedIn progress check node.

14. **Add If node to check if LinkedIn data contains records**  
    - Name: `Checks if any LinkedIn job records exist`  
    - Condition: `$json.records !== 0`  
    - Connect If node's true branch to next step.

15. **Add HTTP Request node to fetch LinkedIn job data**  
    - Name: `Fetches scraped LinkedIn data using snapshot ID`  
    - Method: GET  
    - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
    - Query: `format=json`  
    - Header: Authorization as above  
    - Connect If node's true output here.

16. **Add Merge node**  
    - Name: `Combines Indeed + LinkedIn job results`  
    - Merge two inputs (Indeed job data and LinkedIn job data) into one dataset.  
    - Connect outputs of Indeed fetch and LinkedIn fetch nodes to this merge node.

17. **Add Google Sheets node**  
    - Name: `Saves final job list to Google Sheet`  
    - Operation: Append  
    - Document ID: `1xkNBckPDGf4YR74bJQN07tAr3qlEoA-70pQc63nBqZ8` (replace with your own)  
    - Sheet Name: `Compare`  
    - Map columns: Salary, Location, Job Title, Job Type, Apply Link, Job Detail, Company Name, etc.  
    - Use OAuth2 credentials connected to your Google account.  
    - Connect Merge node output to this node.

18. **Add Sticky Notes** at appropriate places to annotate workflow logic for clarity.

19. **Set Credentials:**  
    - Configure Bright Data API Key in HTTP Request nodes requiring Authorization header.  
    - Configure Google Sheets OAuth2 credentials for Google Sheets node.

20. **Activate workflow** and test by submitting the form with valid job search parameters.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Make a copy of the Google Sheet used for output: https://docs.google.com/spreadsheets/d/1FmjpyNjus0tdN9hU9e-EsFcKl-W4EdWq1PYfIvGu8Ws/edit#gid=0 | Suggested as base sheet for storing job results.                                                            |
| Bright Data API documentation recommended for parameter details and error handling: https://brightdata.com/docs/api/datasets | Important for maintaining or extending scraping API calls.                                                  |
| OAuth2 credential setup for Google Sheets is required for data export; ensure correct scopes enabled | Critical for Google Sheets node functionality.                                                              |
| Workflow employs polling with 1-minute wait intervals to handle asynchronous scraping data availability | Prevents premature data fetch attempts and respects API limits.                                             |
| Dataset IDs correspond to specific Bright Data datasets for LinkedIn and Indeed; verify validity before use | Dataset IDs `gd_lpfll7v5hcqtkxl6l` (Indeed), `gd_l4dx9j9sscpvs7no2` (LinkedIn) used in HTTP requests.       |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.