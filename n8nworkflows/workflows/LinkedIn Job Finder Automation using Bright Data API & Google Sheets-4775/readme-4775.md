LinkedIn Job Finder Automation using Bright Data API & Google Sheets

https://n8nworkflows.xyz/workflows/linkedin-job-finder-automation-using-bright-data-api---google-sheets-4775


# LinkedIn Job Finder Automation using Bright Data API & Google Sheets

### 1. Workflow Overview

This workflow automates the process of searching for LinkedIn job postings based on user-submitted criteria, retrieving the data via the Bright Data API, and appending the results into a Google Sheets document. It targets use cases where users want to programmatically gather recent LinkedIn job listings filtered by location, job title, country, and optionally job type, without manual scraping.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures job search criteria submitted via a web form.
- **1.2 Job Search Initiation:** Sends the search criteria to Bright Data’s LinkedIn dataset API to trigger a scraping job and obtain a snapshot ID.
- **1.3 Status Polling Loop:** Periodically checks the scraping job status until it is complete.
- **1.4 Data Retrieval:** Downloads the scraped job data using the snapshot ID.
- **1.5 Data Filtering:** Optionally filters the results based on job summary content.
- **1.6 Data Storage:** Appends the filtered job data into a specified Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects user input via a form containing job search parameters such as City, Job Title, Country, and optional Job Type. It serves as the workflow trigger.

**Nodes Involved:**  
- On form submission1  
- Sticky Note (description)

**Node Details:**  

- **On form submission1**  
  - Type: Form Trigger  
  - Role: Starts the workflow when a user submits the form.  
  - Configuration:  
    - Form titled "LinkedIn Job Finder"  
    - Fields: City (required), Job Title (required), Country (required), Job_type (dropdown, optional with predefined options including Full-Time, Part-Time, Remote, WFH, Contract, Internship, Freelance)  
  - Inputs: Webhook trigger on form submission  
  - Outputs: JSON containing form fields  
  - Edge Cases: Missing required fields cause workflow not to trigger; invalid dropdown values possible but limited by dropdown options  
  - Sticky Note attached describes purpose as collecting user search criteria.

---

#### 2.2 Job Search Initiation

**Overview:**  
This block sends the user’s search parameters to the Bright Data API to trigger a LinkedIn job scraping dataset job. It retrieves a snapshot ID representing the asynchronous scraping task.

**Nodes Involved:**  
- Create Snapshot ID  
- Check Snapshot Status  
- Check Final Status (IF Node)  
- Sticky Notes (describing HTTP requests and status check logic)

**Node Details:**  

- **Create Snapshot ID**  
  - Type: HTTP Request (POST)  
  - Role: Initiates scraping job with Bright Data API’s dataset trigger endpoint.  
  - Configuration:  
    - URL: https://api.brightdata.com/datasets/v3/trigger  
    - Method: POST  
    - Body: JSON including search parameters extracted and sanitized from form input: City, Job Title, Country (ISO2 uppercase), Time Range (fixed to "Past week"), Job Type (normalized and validated against a predefined list)  
    - Custom output fields requested include detailed job posting metadata like URL, title, company, location, summary, and error/warning fields for troubleshooting  
    - Query Parameters include dataset_id (placeholder value), type, discover_by, limit_per_input  
    - Header includes Authorization Bearer token (placeholder)  
  - Outputs: JSON containing snapshot_id and initial job trigger response  
  - Edge Cases:  
    - API authentication failure (invalid token)  
    - Invalid or empty parameters leading to empty or error responses  
    - Network issues causing request failure  
  - Sticky Note explains it triggers the LinkedIn job search.

- **Check Snapshot Status**  
  - Type: HTTP Request (GET)  
  - Role: Polls Bright Data API for scraping progress using the snapshot_id.  
  - Configuration:  
    - URL: https://api.brightdata.com/datasets/v3/progress/{{snapshot_id}}  
    - Query: format=json  
    - Header: Authorization Bearer token  
  - Inputs: snapshot_id from Create Snapshot ID node  
  - Outputs: JSON status including possible status values (e.g., “ready”)  
  - Edge Cases:  
    - Snapshot ID invalid or expired  
    - API rate limiting or downtime  
    - Timeout if status is not “ready” for an extended time  
  - Sticky Note describes purpose as checking job status before continuing.

- **Check Final Status (IF Node)**  
  - Type: IF Node  
  - Role: Checks if snapshot status equals “ready”.  
  - Configuration:  
    - Condition: $json.status === "ready"  
  - Inputs: Status JSON from Check Snapshot Status  
  - Outputs:  
    - If True: Continue to data retrieval  
    - If False: Loop back to Wait node for delay and retry  
  - Edge Cases: Status field missing or unexpected values cause logic failures.  
  - Sticky Note explains this node controls the polling loop.

---

#### 2.3 Status Polling Loop

**Overview:**  
This block enforces a 1-minute delay between status checks to avoid API overload and ensures the workflow waits until data is ready.

**Nodes Involved:**  
- Wait 1 minute  
- Connections looping back to Check Snapshot Status  
- Sticky Note describing purpose

**Node Details:**  

- **Wait 1 minute**  
  - Type: Wait  
  - Role: Pauses workflow execution for 1 minute between API status checks.  
  - Configuration:  
    - Unit: minutes  
    - Amount: 1  
  - Inputs: From Check Final Status IF node when status is not “ready”  
  - Outputs: Back to Check Snapshot Status node for next poll  
  - Edge Cases: Long delays if snapshot never becomes ready; workflow timeout limits need consideration.  
  - Sticky Note highlights prevention of frequent polling.

---

#### 2.4 Data Retrieval

**Overview:**  
Once the snapshot is marked ready, this block downloads the scraped LinkedIn job posting data in JSON form.

**Nodes Involved:**  
- Scrape Data from SnapID  
- Sticky Note describing download purpose

**Node Details:**  

- **Scrape Data from SnapID**  
  - Type: HTTP Request (GET)  
  - Role: Fetches the completed job dataset using the snapshot ID.  
  - Configuration:  
    - URL: https://api.brightdata.com/datasets/v3/snapshot/{{ snapshot_id }}  
    - Query: format=json  
    - Header: Authorization Bearer token  
  - Inputs: snapshot_id from previous nodes  
  - Outputs: JSON array of job postings with fields as specified in snapshot trigger  
  - Edge Cases:  
    - Snapshot expired or deleted before retrieval  
    - Partial or malformed JSON response  
    - API authentication failure  
  - Sticky Note: Downloads final scraped data.

---

#### 2.5 Data Filtering

**Overview:**  
Filters the downloaded job postings based on whether the job summary contains a specified short common name (likely a refined keyword or filter).

**Nodes Involved:**  
- Filter

**Node Details:**  

- **Filter**  
  - Type: Filter node  
  - Role: Filters job postings to include only those where the job summary contains the value in `shortCommonName`  
  - Configuration:  
    - Condition: `$json.job_summary` contains `$json.shortCommonName`  
  - Inputs: Job postings JSON from Scrape Data from SnapID node  
  - Outputs: Filtered list of job postings  
  - Edge Cases:  
    - Missing or empty `job_summary` or `shortCommonName` fields causing filter to exclude all entries  
    - Case sensitivity can affect matches (case sensitive as configured)  
  - No sticky note attached to this node specifically.

---

#### 2.6 Data Storage

**Overview:**  
Appends each filtered job posting as a new row into a Google Sheets document, capturing relevant job details for user review.

**Nodes Involved:**  
- Update Job Lists in sheet  
- Sticky Note describing Google Sheets usage

**Node Details:**  

- **Update Job Lists in sheet**  
  - Type: Google Sheets node (Append operation)  
  - Role: Adds rows to a Google Sheet with job postings data.  
  - Configuration:  
    - Operation: Append  
    - Document ID: Placeholder for target Google Sheet ID  
    - Sheet Name: Sheet1 (gid=0)  
    - Mapping: Maps fields from job posting JSON to columns: Location, Job Title, Apply Link, Job Detail, Company URL, Company Name  
    - Credential: Google Sheets OAuth2 credentials (placeholder for actual credential)  
  - Inputs: Filtered job postings from Filter node  
  - Outputs: None (final step)  
  - Edge Cases:  
    - Authentication failure with Google Sheets API  
    - Rate limits or quota exceeded on Google Sheets side  
    - Data mapping errors if fields are missing or malformed  
  - Sticky Note explains purpose as appending job results.

---

### 3. Summary Table

| Node Name              | Node Type              | Functional Role                                      | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                      |
|------------------------|------------------------|-----------------------------------------------------|-------------------------|---------------------------|------------------------------------------------------------------------------------------------|
| On form submission1     | Form Trigger           | Collect user search criteria and trigger workflow   | —                       | Create Snapshot ID        | 🔹 Search by Keyword — Form Trigger<br>Collects user-submitted job search criteria.            |
| Create Snapshot ID     | HTTP Request (POST)    | Trigger Bright Data LinkedIn scraping job           | On form submission1      | Check Snapshot Status     | 🔹 LinkedIn Job Search Fetcher — HTTP Request (POST to Bright Data)<br>Sends search criteria.  |
| Check Snapshot Status  | HTTP Request (GET)     | Poll Bright Data API for scraping status            | Create Snapshot ID       | Check Final Status        | 🔹 Check Delivery Status of Snap ID — HTTP Request (GET)<br>Checks scraping job status.        |
| Check Final Status     | IF Node                | Check if scraping job status is "ready"             | Check Snapshot Status    | If (true), Wait 1 minute (false) | 🔹 Check Final Status — IF Node #1<br>Conditional status check to proceed or wait.           |
| Wait 1 minute          | Wait                   | Pause 1 minute between status polls                  | Check Final Status       | Check Snapshot Status     | 🔹 Wait 1 minute — Wait Node<br>Prevents frequent API polling and reduces errors.              |
| Scrape Data from SnapID| HTTP Request (GET)     | Download scraped LinkedIn job data                    | If                      | Filter                    | 🔹 Decode Snapshot from Response — HTTP Request (GET)<br>Download final scraped dataset.       |
| Filter                 | Filter                 | Filter job postings by presence of shortCommonName  | Scrape Data from SnapID  | Update Job Lists in sheet |                                                                                                |
| Update Job Lists in sheet | Google Sheets          | Append filtered job data rows to Google Sheet        | Filter                   | —                         | 🔹 Google Sheets — Google Sheets (AppendOrUpdate)<br>Appends job results into Google Sheet.    |
| Sticky Note            | Sticky Note            | Descriptions for workflow blocks                     | —                       | —                         | Multiple sticky notes associated with blocks as described above.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Add a Form Trigger node named "On form submission1".  
   - Configure form title as "LinkedIn Job Finder".  
   - Add required fields: City (text), Job Title (text), Country (text).  
   - Add optional dropdown field "Job_type" with options: Full-Time, Part-Time, Remote, WFH, Contract, Internship, Freelance.  
   - Save and activate webhook.

2. **Add HTTP Request Node to Trigger Scraping Job**  
   - Add "Create Snapshot ID" node of type HTTP Request.  
   - Set method to POST.  
   - Set URL to `https://api.brightdata.com/datasets/v3/trigger`.  
   - Set request body (JSON) to include:  
     - `location` from `City` form field trimmed.  
     - `keyword` from `Job Title` trimmed.  
     - `country` from `Country` first two letters uppercase.  
     - `time_range` fixed to "Past week".  
     - `job_type` validated against allowed types (full-time, part-time, internship, contract, temporary).  
     - Include `custom_output_fields` array with detailed job metadata.  
   - Add query parameters:  
     - `dataset_id` (replace with your Bright Data dataset ID).  
     - `include_errors` = true.  
     - `type` = discover_new.  
     - `discover_by` = keyword.  
     - `limit_per_input` = 2.  
   - Add header parameter Authorization with bearer token (your Bright Data API token).  
   - Connect output of form trigger to this node.

3. **Add HTTP Request Node to Check Snapshot Status**  
   - Add "Check Snapshot Status" node of type HTTP Request.  
   - Set method to GET.  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (expression using snapshot_id from previous node).  
   - Query parameter: format=json.  
   - Header: Authorization bearer token (same as above).  
   - Connect output of Create Snapshot ID to this node.

4. **Add IF Node to Check If Snapshot Status is Ready**  
   - Add "Check Final Status" IF node.  
   - Condition: `$json.status === "ready"`.  
   - Connect output of Check Snapshot Status to this node.

5. **Add Wait Node for Delay**  
   - Add "Wait 1 minute" node.  
   - Configure to wait 1 minute.  
   - Connect "Check Final Status" node output "false" branch to Wait node.  
   - Connect Wait node output back to Check Snapshot Status node for polling loop.

6. **Add HTTP Request Node to Download Snapshot Data**  
   - Add "Scrape Data from SnapID" node.  
   - Method: GET.  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`.  
   - Query parameter: format=json.  
   - Header: Authorization with bearer token.  
   - Connect "Check Final Status" node output "true" branch to this node.

7. **Add Filter Node to Filter Job Postings**  
   - Add "Filter" node.  
   - Condition: Check if `$json.job_summary` contains `$json.shortCommonName`.  
   - Connect output of Scrape Data from SnapID to this node.

8. **Add Google Sheets Node to Append Data**  
   - Add "Update Job Lists in sheet" node of type Google Sheets.  
   - Operation: Append.  
   - Document ID: Your Google Sheet ID.  
   - Sheet Name: Use Sheet1 or specify gid=0.  
   - Map columns:  
     - Location ← job_location  
     - Job Title ← job_title  
     - Apply Link ← apply_link  
     - Job Detail ← job_summary  
     - Company URL ← company_url  
     - Company Name ← company_name  
   - Set Google Sheets OAuth2 credentials (create and configure OAuth2 credentials with appropriate scopes).  
   - Connect output of Filter node to this node.

9. **Final Checks**  
   - Review all placeholder tokens and IDs (Bright Data API token, dataset ID, Google Sheets ID, credentials).  
   - Test the workflow end-to-end with sample form submissions.  
   - Add error handling if needed (not included in original workflow).  
   - Activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow uses Bright Data LinkedIn dataset API, requiring a valid dataset ID and API token from Bright Data. | https://brightdata.com/                                                                                   |
| Google Sheets OAuth2 credentials must have permissions to edit the target spreadsheet.                     | https://developers.google.com/sheets/api/guides/authorizing                                               |
| The workflow includes planned polling with 1-minute delay to avoid API throttling and reduce errors.       |                                                                                                          |
| Job type filtering supports a fixed subset of types consistent with LinkedIn categories.                    |                                                                                                          |
| Sticky notes in workflow provide concise descriptions of each logical block's purpose.                      |                                                                                                          |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.