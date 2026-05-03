Scrape LinkedIn Job Listings with Phantombuster & Save to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-linkedin-job-listings-with-phantombuster---save-to-google-sheets-8144


# Scrape LinkedIn Job Listings with Phantombuster & Save to Google Sheets

### 1. Workflow Overview

This workflow automates the process of scraping LinkedIn job listings for a set of companies and saving the results into a Google Sheet for competitive hiring intelligence. It targets recruiters and HR analysts who want to track job openings across multiple companies regularly.

The workflow is logically divided into three main blocks:

- **1.1 Trigger & Input Retrieval:** Periodically triggers the workflow and reads a list of companies (with LinkedIn URLs) marked as "Pending" from a Google Sheet.
- **1.2 Scraping Process (Phantombuster Integration):** Calls the Phantombuster API to launch a LinkedIn job scraping agent for each company URL, waits for the scrape to complete, and fetches the resulting CSV data link.
- **1.3 Data Processing & Storage:** Formats the scraped job data into a standardized structure and appends it to a dedicated Google Sheet for job results, including metadata like scrape date.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Retrieval

**Overview:**  
This block schedules the workflow to run weekly on Mondays at 9 AM, then reads a Google Sheet containing companies to scrape. It filters companies whose status is "Pending" to ensure only relevant entries trigger the scraper.

**Nodes Involved:**  
- ⏰Schedule Trigger  
- 📄Read Companies Sheet

**Node Details:**

- **⏰Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow weekly at a fixed time  
  - Configuration: Cron expression set to trigger at 09:00 AM every Monday (`0 9 * * 1`)  
  - Inputs: None (start node)  
  - Outputs: Connects to "📄Read Companies Sheet"  
  - Edge Cases: Workflow will not run outside scheduled times; ensure server timezone matches intended schedule.

- **📄Read Companies Sheet**  
  - Type: Google Sheets  
  - Role: Reads company data from a Google Sheet to get LinkedIn profile URLs  
  - Configuration: Reads from sheet named "Sheet1" (gid=0) in a specified Google Sheet document ID; filters rows where the `Status` column equals "Pending" and returns only the first matching row.  
  - Inputs: From Schedule Trigger  
  - Outputs: Connects to "🚀 Trigger Phantombuster Scraper"  
  - Credentials: Requires OAuth2 credentials for Google Sheets  
  - Edge Cases:  
    - If no rows match "Pending," downstream nodes may receive empty data causing failures.  
    - Network or credential issues may prevent sheet access.

---

#### 2.2 Scraping Process (Phantombuster Integration)

**Overview:**  
This block triggers the Phantombuster agent to scrape LinkedIn job listings for the provided company URL, waits for 3 minutes for the scrape to complete, and then fetches the output CSV container link.

**Nodes Involved:**  
- 🚀 Trigger Phantombuster Scraper  
- ⏳ Wait for Scraper to Finish  
- 📦 Fetch Scraped CSV Link

**Node Details:**

- **🚀 Trigger Phantombuster Scraper**  
  - Type: HTTP Request  
  - Role: Sends a POST request to start the Phantombuster scraping agent  
  - Configuration:  
    - URL: `https://api.phantombuster.com/api/v2/agents/launch`  
    - Method: POST  
    - Headers: Includes `X-Phantombuster-Key-1` with API key and `Content-Type: application/json`  
    - Body: JSON with agent ID and argument containing an array of LinkedIn profile URLs extracted dynamically from the previous node (`{{ $json.LinkedIn }}`)  
  - Inputs: Receives company row with LinkedIn URL  
  - Outputs: Connects to "⏳ Wait for Scraper to Finish"  
  - Edge Cases:  
    - Invalid or missing API key leads to auth errors  
    - Incorrect agent ID or malformed URL can cause request failures  
    - Empty or malformed LinkedIn URL input can cause scraping to fail or produce no data.

- **⏳ Wait for Scraper to Finish**  
  - Type: Wait  
  - Role: Pauses the workflow for 3 minutes to allow the scraping job to complete  
  - Configuration: Wait set to 3 minutes  
  - Inputs: From HTTP Request node  
  - Outputs: Connects to "📦 Fetch Scraped CSV Link"  
  - Edge Cases:  
    - Fixed wait time may be insufficient if scraping takes longer, causing incomplete data fetch  
    - Excessive wait time delays workflow execution unnecessarily.

- **📦 Fetch Scraped CSV Link**  
  - Type: HTTP Request  
  - Role: Retrieves the output container from Phantombuster to fetch the scraped CSV data link  
  - Configuration:  
    - URL dynamically constructed using container ID from the scraper response (`{{ $('🚀 Trigger Phantombuster Scraper').item.json.containerId }}`)  
    - Method: GET (default)  
    - Headers: `X-Phantombuster-Key-1` API key included  
  - Inputs: From Wait node  
  - Outputs: Connects to "🛠️ Format Job Data"  
  - Edge Cases:  
    - If containerId is missing or invalid, request will fail  
    - Network issues or API quota limits may cause errors.

---

#### 2.3 Data Processing & Storage

**Overview:**  
Formats the raw scraped job data into a defined schema and appends it to a Google Sheet for job results, including the current scrape date for tracking.

**Nodes Involved:**  
- 🛠️ Format Job Data  
- 📊 Write to Results Sheet

**Node Details:**

- **🛠️ Format Job Data**  
  - Type: Set (data transformation)  
  - Role: Maps and structures raw scraped data into specific fields: Company Name, Job Title, Job Description, Job Link, Date Posted, Location, Employment Type  
  - Configuration: Defines fields explicitly without values here (presumably populated dynamically from previous node data)  
  - Inputs: From CSV fetch node  
  - Outputs: Connects to "📊 Write to Results Sheet"  
  - Edge Cases:  
    - If input data is missing expected fields, mapping will fail or produce incomplete records  
    - Data type inconsistencies may cause formatting errors.

- **📊 Write to Results Sheet**  
  - Type: Google Sheets  
  - Role: Appends the formatted job data to a "Job Results" sheet in a specified Google Sheet document  
  - Configuration:  
    - Mapping mode defines each column explicitly with expressions extracting data from the JSON input  
    - Adds a "Scraped Date" column with current date (`new Date().toISOString().split('T')[0]`) for tracking  
    - Operation: Append (adds new rows)  
  - Inputs: From Format Job Data  
  - Credentials: Uses Google Sheets OAuth2 credentials  
  - Outputs: None (end node)  
  - Edge Cases:  
    - Failure to authenticate or access the sheet will cause errors  
    - If sheet structure changes, mapping may fail or misalign data  
    - Large data volumes may hit API rate limits.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                              | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                                                          |
|-------------------------------|--------------------|----------------------------------------------|-------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| ⏰Schedule Trigger            | Schedule Trigger   | Starts workflow every Monday at 9 AM         | None                          | 📄Read Companies Sheet           | Runs the workflow at 9:00 AM every Monday.                                                                                           |
| 📄Read Companies Sheet        | Google Sheets      | Reads companies with status "Pending"        | ⏰Schedule Trigger             | 🚀 Trigger Phantombuster Scraper | Reads data from a specified Google Sheet containing company details, filtering only those with Status = Pending.                     |
| 🚀 Trigger Phantombuster Scraper | HTTP Request      | Launches Phantombuster agent for scraping    | 📄Read Companies Sheet         | ⏳ Wait for Scraper to Finish     | Sends a POST request to the Phantombuster API to start a LinkedIn job scrape, passing the LinkedIn profile URL from the sheet as input. |
| ⏳ Wait for Scraper to Finish | Wait               | Pauses workflow to allow scraping to complete| 🚀 Trigger Phantombuster Scraper | 📦 Fetch Scraped CSV Link         | Pauses execution for 3 minutes to allow Phantombuster to complete the scrape.                                                        |
| 📦 Fetch Scraped CSV Link     | HTTP Request       | Retrieves scraped CSV output link             | ⏳ Wait for Scraper to Finish  | 🛠️ Format Job Data               | Fetches the output CSV file link from the Phantombuster container.                                                                   |
| 🛠️ Format Job Data           | Set                | Formats scraped data into predefined fields   | 📦 Fetch Scraped CSV Link      | 📊 Write to Results Sheet         | Structures the scraped job data into predefined fields — Company Name, Job Title, Job Description, Job Link, Date Posted, Location, and Employment Type. |
| 📊 Write to Results Sheet     | Google Sheets      | Appends formatted job data to results sheet  | 🛠️ Format Job Data            | None                            | Appends the formatted job data into a separate Google Sheet (Job Results), including the current scrape date for tracking.            |
| Sticky Note                  | Sticky Note        | Explains Trigger & Input Retrieval block      | None                          | None                            | Ensures the scraper runs on fresh, relevant company entries at a scheduled interval.                                                  |
| Sticky Note1                 | Sticky Note        | Explains Scraping Process block                | None                          | None                            | Automates the job scraping process without manual API checks or re-runs.                                                              |
| Sticky Note2                 | Sticky Note        | Explains Data Processing & Storage block      | None                          | None                            | Converts raw scrape output into a clean, standardized dataset and saves it for easy review.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set Cron expression to `0 9 * * 1` (every Monday at 9 AM)  
   - Connect output to next node.

2. **Create a Google Sheets node ("Read Companies Sheet")**  
   - Operation: Read rows  
   - Document ID: Set your Google Sheet ID containing companies  
   - Sheet Name: Set to "Sheet1" or appropriate tab (gid=0)  
   - Filters: Add filter where `Status` column equals "Pending"  
   - Return only the first matching row (or all matches if desired)  
   - Connect input from Schedule Trigger node  
   - Configure Google Sheets OAuth2 credentials.

3. **Create an HTTP Request node ("Trigger Phantombuster Scraper")**  
   - Method: POST  
   - URL: `https://api.phantombuster.com/api/v2/agents/launch`  
   - Headers:  
     - `X-Phantombuster-Key-1`: Your Phantombuster API key  
     - `Content-Type`: `application/json`  
   - Body (JSON):  
     ```json
     {
       "id": "YOUR_PHANTOMBUSTER_AGENT_ID",
       "argument": {
         "profileUrls": ["{{$json.LinkedIn}}"]
       }
     }
     ```  
   - Send body as JSON  
   - Connect input from Google Sheets node.

4. **Create a Wait node ("Wait for Scraper to Finish")**  
   - Set unit to minutes  
   - Set amount to 3  
   - Connect input from HTTP Request node.

5. **Create an HTTP Request node ("Fetch Scraped CSV Link")**  
   - Method: GET  
   - URL:  
     `https://api.phantombuster.com/api/v2/containers/fetch-output?id={{$('Trigger Phantombuster Scraper').item.json.containerId}}`  
   - Headers:  
     - `X-Phantombuster-Key-1`: Your Phantombuster API key  
   - Connect input from Wait node.

6. **Create a Set node ("Format Job Data")**  
   - Add fields:  
     - Company Name  
     - Job Title  
     - Job Description  
     - Job Link  
     - Date Posted  
     - Location  
     - Employment Type  
   - Map these fields according to the fetched CSV data structure from the previous node (use expressions to extract data)  
   - Connect input from Fetch Scraped CSV Link node.

7. **Create a Google Sheets node ("Write to Results Sheet")**  
   - Operation: Append row  
   - Document ID: Your Google Sheet ID for storing results  
   - Sheet Name: "Job Results"  
   - Map columns to match the fields in the Set node, including:  
     - Scraped Date: `={{new Date().toISOString().split('T')[0]}}`  
   - Connect input from Format Job Data node  
   - Configure Google Sheets OAuth2 credentials.

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                                         |
|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow enables automated weekly scraping of LinkedIn job listings using Phantombuster API.      | Phantombuster API documentation: https://phantombuster.com/api/v2/                                                     |
| Google Sheets OAuth2 credentials must have appropriate permissions for reading and writing sheets.| https://developers.google.com/sheets/api/guides/authorizing                                                             |
| Ensure the LinkedIn URLs in the companies sheet are valid and publicly accessible for scraping.   | LinkedIn profiles or company pages should be accessible without login or limited by Phantombuster agent capabilities.    |
| Consider increasing wait time if scraping frequently fails due to incomplete data availability.   | Adjust wait node duration as per average job scrape completion times.                                                   |
| Monitor API quota limits for both Phantombuster and Google Sheets to avoid workflow failures.     | Quotas and limits: https://developers.google.com/sheets/api/limits and Phantombuster API usage policies                  |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.