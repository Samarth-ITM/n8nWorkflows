Scrape Jobs from Indeed, LinkedIn & More with RapidAPI to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-jobs-from-indeed--linkedin---more-with-rapidapi-to-google-sheets-8250


# Scrape Jobs from Indeed, LinkedIn & More with RapidAPI to Google Sheets

### 1. Workflow Overview

This workflow automates the scraping of job listings from multiple major job platforms — Indeed, LinkedIn, ZipRecruiter, and Glassdoor — using the RapidAPI Jobs Search Realtime Data API. It collects user inputs via a form for parameters like location, job type, remote status, and number of results, then queries the API accordingly. The retrieved job data is reformatted and appended into a Google Sheets document for easy access and analysis.

The logic is organized into these functional blocks:

- **1.1 Input Reception:** Captures user job search criteria submitted through a form.
- **1.2 API Query:** Sends a POST request to RapidAPI’s Jobs Search Realtime Data API with user parameters.
- **1.3 Data Transformation:** Extracts and reformats the job listings array from the API response.
- **1.4 Data Storage:** Appends the transformed job listings into a Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures job search inputs from users through a web form, including location, search term, job type, remote option, and desired number of results.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point to capture user input via a web form titled “Active Job Scraper”  
    - Configuration: Fields captured are Location (text), search term (text), job type (text), is remote (dropdown with options true/false), and Result (number, default placeholder 10)  
    - Input/Output: No input; outputs form data as JSON for downstream nodes  
    - Edge cases: Missing required fields, invalid input types, or webhook connectivity issues may cause failures.  
    - Version-specific: Uses webhookId for form trigger (v2.2)  
    - No sub-workflow invoked

#### 2.2 API Query

- **Overview:**  
  Sends a POST request to the RapidAPI Jobs Search Realtime Data API to fetch job listings based on user inputs.

- **Nodes Involved:**  
  - Scarp Active Jobs

- **Node Details:**  
  - **Scarp Active Jobs**  
    - Type: HTTP Request  
    - Role: Query RapidAPI to get job listings from multiple sites  
    - Configuration:  
      - URL: `https://jobs-search-realtime-data.p.rapidapi.com/getjobs`  
      - Method: POST  
      - Body: JSON constructed dynamically using expressions to inject form inputs: location, search_term, results_wanted, job_type, is_remote, and static parameters like sites (indeed, linkedin, zip_recruiter, glassdoor), distance=50, linkedin_fetch_description=true, hours_old=24  
      - Headers: x-rapidapi-host and x-rapidapi-key (API key must be provided by user)  
      - Sends body as JSON with header parameters  
    - Input: JSON from form submission node  
    - Output: API JSON response including job listings  
    - Edge cases: API authentication errors (invalid/missing key), network timeouts, malformed JSON, or empty responses can occur  
    - Version-specific: v4.2 HTTP Request node  
    - No sub-workflow invoked

#### 2.3 Data Transformation

- **Overview:**  
  Extracts the `jobs` array from the API response JSON and outputs it as the main data payload for further processing.

- **Nodes Involved:**  
  - Re Format

- **Node Details:**  
  - **Re Format**  
    - Type: Code (JavaScript)  
    - Role: Extracts the array of jobs for appending to Google Sheets  
    - Configuration: Custom JS code: `return $input.first().json.jobs;` which returns the `jobs` property of the first input JSON object  
    - Input: API response JSON from “Scarp Active Jobs” node  
    - Output: Array of job objects  
    - Edge cases: If `jobs` property is missing or null, the output will be undefined or empty, which could cause downstream failures  
    - Version-specific: Code node v2  
    - No sub-workflow invoked

#### 2.4 Data Storage

- **Overview:**  
  Appends the reformatted job listing data into a specified Google Sheets document for record-keeping and analysis.

- **Nodes Involved:**  
  - Append In Google Sheets

- **Node Details:**  
  - **Append In Google Sheets**  
    - Type: Google Sheets  
    - Role: Append job data rows into Google Sheets  
    - Configuration:  
      - Operation: Append  
      - Sheet Name: Sheet1 (gid=0)  
      - Document ID: User must specify the Google Sheets document ID  
      - Columns: Defines columns success, courses, totalItems, currentPage, totalPages (mapping mode: defineBelow) — note: these columns appear generic and may need to be customized to actual job data fields  
      - Authentication: Service Account credentials configured with Google API access  
    - Input: Array of job objects from “Re Format” node  
    - Output: Confirmation of append operation success  
    - Edge cases: Authentication errors, permission issues on the Google Sheet, invalid document ID, or data schema mismatch could cause failure  
    - Version-specific: Google Sheets node v4.6  
    - No sub-workflow invoked

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                        | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                                                            |
|---------------------|---------------------|-------------------------------------|-----------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger        | Capture user job search inputs       | —                     | Scarp Active Jobs       | Gather user inputs including location, job type, remote status, etc.                                                                                   |
| Scarp Active Jobs    | HTTP Request        | Query RapidAPI for job listings      | On form submission    | Re Format               | Query RapidAPI’s **Jobs Search Realtime Data API** for jobs on **Indeed**, **LinkedIn**, **ZipRecruiter**, and **Glassdoor**.                        |
| Re Format           | Code                | Extract jobs array from API response | Scarp Active Jobs      | Append In Google Sheets | Extract and reshape the jobs array from the API response.                                                                                            |
| Append In Google Sheets | Google Sheets      | Append job data to Google Sheets     | Re Format              | —                       | Append the scraped job listings to your Google Sheets document.                                                                                       |
| Sticky Note         | Sticky Note         | Documentation and description        | —                     | —                       | # Active Job Scraper Workflow<br>Automate your job search data collection with this powerful **Active Job Scraper** workflow built in **n8n**. <br>See details in workflow description. |
| Sticky Note1        | Sticky Note         | Form input explanation                | —                     | —                       | Gather user inputs including location, job type, remote status, etc.                                                                                   |
| Sticky Note2        | Sticky Note         | API query explanation                 | —                     | —                       | Query RapidAPI’s **Jobs Search Realtime Data API** for jobs on **Indeed**, **LinkedIn**, **ZipRecruiter**, and **Glassdoor**.                        |
| Sticky Note3        | Sticky Note         | Data extraction explanation           | —                     | —                       | Extract and reshape the jobs array from the API response.                                                                                            |
| Sticky Note4        | Sticky Note         | Google Sheets append explanation      | —                     | —                       | Append the scraped job listings to your Google Sheets document.                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a **Form Trigger** node named “On form submission”.  
   - Set the form title to “Active Job Scraper”.  
   - Add form fields (all required):  
     - Location (text)  
     - search term (text)  
     - job type (text)  
     - is remote (dropdown with options “true” and “false”)  
     - Result (number, placeholder 10)  
   - Save to generate the webhook URL which will be used to trigger the workflow.

2. **Add HTTP Request Node**  
   - Add an **HTTP Request** node named “Scarp Active Jobs”.  
   - Method: POST  
   - URL: `https://jobs-search-realtime-data.p.rapidapi.com/getjobs`  
   - Set **Body Content Type** to JSON.  
   - In the JSON Body, insert expressions to dynamically populate fields from the form data:  
     ```json
     {
       "location": "={{ $json['Location'] }}",
       "search_term": "={{ $json['search term'] }}",
       "results_wanted": {{ $json.Result }},
       "site_name": ["indeed","linkedin","zip_recruiter","glassdoor"],
       "distance": 50,
       "job_type": "={{ $json['job type'] }}",
       "is_remote": {{ $json['is remote'] }},
       "linkedin_fetch_description": true,
       "hours_old": 24
     }
     ```  
   - Add HTTP headers:  
     - `x-rapidapi-host` = `jobs-search-realtime-data.p.rapidapi.com`  
     - `x-rapidapi-key` = **your RapidAPI key** (credential must be added).  
   - Connect “On form submission” node output to this HTTP Request node input.

3. **Add Code Node for Data Formatting**  
   - Add a **Code** node named “Re Format”.  
   - Use JavaScript code:  
     ```javascript
     return $input.first().json.jobs;
     ```  
   - Connect output of “Scarp Active Jobs” to this node.

4. **Add Google Sheets Node**  
   - Add a **Google Sheets** node named “Append In Google Sheets”.  
   - Operation: Append  
   - Document ID: Enter the Google Sheets document ID where you want to append job data.  
   - Sheet Name: `Sheet1` (or specify the correct sheet by gid=0 or name)  
   - Define columns to map job data fields appropriately: currently set to columns like success, courses, totalItems, currentPage, totalPages but should be adjusted to actual fields from jobs array (e.g., job title, company, location, description, URL).  
   - Authentication: Use a Service Account credential configured with Google Sheets API enabled.  
   - Connect output of “Re Format” node to this node.

5. **Connect Nodes in Order:**  
   - On form submission → Scarp Active Jobs → Re Format → Append In Google Sheets.

6. **Credential Setup:**  
   - Configure RapidAPI key in the HTTP Request node headers or via n8n credentials if preferred.  
   - Create and configure Google API Service Account credentials with permissions to edit the target Google Sheets document.

7. **Testing:**  
   - Submit the form via the generated webhook URL with sample job search parameters.  
   - Verify the job listings are scraped, reformatted, and appended into the Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow uses RapidAPI’s [Jobs Search Realtime Data API](https://rapidapi.com/skdeveloper/api/jobs-search-realtime-data) to fetch jobs from Indeed, LinkedIn, ZipRecruiter, and Glassdoor.                                        | RapidAPI API Documentation                                                                                            |
| The Google Sheets node requires a valid Service Account credential with appropriate scopes to modify the target spreadsheet.                                                                                                       | Google Cloud Console - Service Account & Sheets API                                                                    |
| The form webhook URL is the entry point and must be triggered with all required fields for the workflow to execute correctly.                                                                                                       | n8n Form Trigger Node documentation                                                                                     |
| Be mindful of API rate limits and authentication errors with RapidAPI. Ensure your API key is valid and has appropriate quota.                                                                                                     | RapidAPI usage policies                                                                                                 |
| The current column mapping in the Google Sheets node appears generic; customize columns to match the actual job data properties you want to record (e.g., job title, company, location, etc.).                                       | Customize Google Sheets node columns accordingly                                                                        |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.