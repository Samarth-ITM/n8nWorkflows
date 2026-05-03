Scrape Upwork Job Listings & Generate Daily Email Reports with Apify & Google Sheets

https://n8nworkflows.xyz/workflows/scrape-upwork-job-listings---generate-daily-email-reports-with-apify---google-sheets-6028


# Scrape Upwork Job Listings & Generate Daily Email Reports with Apify & Google Sheets

### 1. Workflow Overview

This workflow automates the process of scraping Upwork job listings using Apify, processing and cleaning the job data, updating Google Sheets with the results, generating daily keyword summary statistics, and finally sending an email report with the aggregated data. It targets users who want to monitor Upwork job opportunities daily, grouped by specific keywords, and receive an organized report for informed decision-making or lead generation.

The workflow is logically divided into the following blocks:

- **1.1 Job Scraping & Initial Processing**: Starts with manual trigger, fetches keywords, scrapes Upwork jobs via Apify API, waits for completion, retrieves raw data, filters recent jobs, and appends results to a daily Google Sheet.

- **1.2 Data Cleaning & Deduplication**: Loads the scraped daily jobs, removes duplicates by title and description, saves cleaned data, clears old or duplicate rows from the sheet, and reloads the clean dataset.

- **1.3 Daily Summary & Email Report**: Generates keyword-based job counts, updates summary sheets, fetches final aggregated data, builds an HTML email body with statistics, and sends a daily email report to the stakeholders.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Job Scraping & Initial Processing

**Overview:**  
This block handles scraping Upwork job listings by iterating over keywords from a Google Sheet, triggering Apify actors to perform the scrape, waiting for completion, fetching the dataset, filtering jobs posted within the last 24 hours, and saving the filtered jobs to a daily Google Sheet.

**Nodes Involved:**  
- Trigger Manual Run  
- Fetch Keywords from Google Sheet  
- Loop Through Keywords  
- Trigger Apify Scraper  
- Wait for Apify Completion  
- Delay Before Dataset Read  
- Fetch Scraped Job Dataset  
- Process Raw Job Data  
- Save Jobs to Daily Sheet  
- Update Keyword Job Count  
- Delete rows or columns from sheet  
- Load Today’s Daily Jobs  

**Node Details:**

- **Trigger Manual Run**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually  
  - Configuration: No parameters  
  - Inputs: None  
  - Outputs: Connects to Fetch Keywords from Google Sheet  
  - Failure cases: None typical; manual start only  

- **Fetch Keywords from Google Sheet**  
  - Type: Google Sheets node  
  - Role: Reads the list of keywords from the “All Keywords combined” sheet in a specific Google Spreadsheet  
  - Configuration: Uses Service Account for authentication, reads all rows in the specified sheet  
  - Inputs: Trigger Manual Run  
  - Outputs: Connects to Loop Through Keywords  
  - Failure cases: Google Sheets API auth errors, sheet access issues  

- **Loop Through Keywords**  
  - Type: Split In Batches  
  - Role: Iterates over each keyword record to handle scraping one keyword at a time  
  - Configuration: Default batch options (likely batch size of 1)  
  - Inputs: Fetch Keywords from Google Sheet  
  - Outputs: Two parallel paths: Load Today’s Daily Jobs and Trigger Apify Scraper  
  - Failure cases: Batch iteration errors  

- **Trigger Apify Scraper**  
  - Type: HTTP Request  
  - Role: Starts a new Apify run for scraping Upwork jobs matching the current keyword  
  - Configuration: POST request to Apify API /runs endpoint with JSON body including filters (clientHistory, experienceLevel, maxJobAge, paymentVerified false, query from current keyword, paging, sorting)  
  - Headers: Content-Type application/json  
  - Authentication: HTTP Header Auth with API key  
  - Inputs: Loop Through Keywords  
  - Outputs: Connects to Wait for Apify Completion  
  - Failure cases: HTTP errors, API key invalid, rate limits, malformed JSON  

- **Wait for Apify Completion**  
  - Type: HTTP Request  
  - Role: Polls Apify API until the actor run finishes or timeout (waitForFinish=150 seconds)  
  - Configuration: GET request to /actor-runs/{runId} endpoint with waitForFinish query param  
  - Authentication: Same HTTP Header Auth  
  - Inputs: Trigger Apify Scraper  
  - Outputs: Connects to Delay Before Dataset Read  
  - Failure cases: Timeout, API unavailability  

- **Delay Before Dataset Read**  
  - Type: Wait node  
  - Role: Waits 20 seconds to ensure dataset availability after actor completes  
  - Inputs: Wait for Apify Completion  
  - Outputs: Connects to Fetch Scraped Job Dataset  
  - Failure cases: None typical  

- **Fetch Scraped Job Dataset**  
  - Type: HTTP Request  
  - Role: Retrieves the scraped job listings dataset from Apify using the dataset ID from actor run  
  - Configuration: GET request to /datasets/{datasetId}/items endpoint  
  - Authentication: HTTP Header Auth  
  - Inputs: Delay Before Dataset Read  
  - Outputs: Connects to Process Raw Job Data  
  - Failure cases: Dataset not found, API errors  

- **Process Raw Job Data**  
  - Type: Code (JavaScript)  
  - Role: Filters job listings to only include those posted within the last 23 hours (relative to current time)  
  - Key Logic: Compares job posting absoluteDate to threshold time (now - 23 hours)  
  - Inputs: Fetch Scraped Job Dataset  
  - Outputs: Connects to Save Jobs to Daily Sheet  
  - Failure cases: Date parsing errors, empty datasets  

- **Save Jobs to Daily Sheet**  
  - Type: Google Sheets node  
  - Role: Appends the filtered job data to the daily job sheet (named with date, e.g., “15 July 2025”)  
  - Configuration: Maps job data fields to columns (title, description, url, budget, jobType, dates, clientLocation, experienceLevel, paymentVerified, tags, allowedApplicantCountries, and the keyword used)  
  - Authentication: Service Account  
  - Inputs: Process Raw Job Data  
  - Outputs: Connects to Update Keyword Job Count  
  - Failure cases: Google Sheets quota, API errors, data type mismatches  

- **Update Keyword Job Count**  
  - Type: Google Sheets node  
  - Role: Updates the "All Keywords combined" sheet with the total job count for the current keyword for today’s date  
  - Configuration: Append or update row matching keyword, with total count from processed job data  
  - Authentication: Service Account  
  - Inputs: Save Jobs to Daily Sheet  
  - Outputs: Connects back to Loop Through Keywords (to iterate next keyword)  
  - Failure cases: Sheet update conflicts, API errors  

- **Delete rows or columns from sheet**  
  - Type: Google Sheets node  
  - Role: Deletes rows in the daily jobs sheet equal to the number of jobs loaded (clearing old data before saving clean data)  
  - Configuration: Delete operation on daily jobs sheet, number to delete equals the length of “Load Today’s Daily Jobs” output  
  - Authentication: Service Account  
  - Inputs: Save Clean Job Data (see next block)  
  - Outputs: Connects to Reload Clean Job Data  
  - Failure cases: Row index errors, sheet permissions  

- **Load Today’s Daily Jobs**  
  - Type: Google Sheets node  
  - Role: Loads all jobs currently saved in the daily sheet (used for deduplication and cleaning)  
  - Configuration: Read all rows from daily jobs sheet  
  - Authentication: Service Account  
  - Inputs: Loop Through Keywords (parallel branch)  
  - Outputs: Connects to Remove Duplicates by Title/Desc  
  - Failure cases: Sheet read errors  

---

#### 2.2 Data Cleaning & Deduplication

**Overview:**  
This block ensures the daily job data is clean and unique by removing duplicate entries based on title and description, saving the clean data, clearing old entries, and reloading the cleaned dataset for further processing.

**Nodes Involved:**  
- Load Today’s Daily Jobs  
- Remove Duplicates by Title/Desc  
- Save Clean Job Data  
- Delete rows or columns from sheet  
- Reload Clean Job Data  

**Node Details:**

- **Remove Duplicates by Title/Desc**  
  - Type: Code (JavaScript)  
  - Role: Eliminates duplicate job entries by comparing lowercased, trimmed title and description combinations; ignores placeholder entries (“No records found in last 24 hours”)  
  - Inputs: Load Today’s Daily Jobs  
  - Outputs: Save Clean Job Data  
  - Failure cases: Edge cases if titles or descriptions are missing or malformed  

- **Save Clean Job Data**  
  - Type: Google Sheets node  
  - Role: Appends the deduplicated job listings back to the daily sheet  
  - Configuration: Same column mapping as Save Jobs to Daily Sheet  
  - Authentication: Service Account  
  - Inputs: Remove Duplicates by Title/Desc  
  - Outputs: Delete rows or columns from sheet  
  - Failure cases: Google Sheets API limits, write conflicts  

- **Delete rows or columns from sheet**  
  - (Described above; deletes old rows before reload)  

- **Reload Clean Job Data**  
  - Type: Google Sheets node  
  - Role: Reads the cleaned daily job data back from the sheet for further summarization  
  - Inputs: Delete rows or columns from sheet  
  - Outputs: Generate Keyword Summary Stats  
  - Failure cases: Sheet read errors  

---

#### 2.3 Daily Summary & Email Report

**Overview:**  
This block generates aggregated statistics of job counts per keyword, updates the summary Google Sheet, fetches the final summary data, constructs an HTML email with key metrics, and sends a daily report email to specified recipients.

**Nodes Involved:**  
- Reload Clean Job Data  
- Generate Keyword Summary Stats  
- Update Summary Sheet  
- Fetch Final Summary Data  
- Build Email Body  
- Send Daily Report Email  

**Node Details:**

- **Generate Keyword Summary Stats**  
  - Type: Code (JavaScript)  
  - Role: Counts the number of jobs per keyword by matching keywords with the "Keyword Title" in the cleaned job data, excluding placeholder titles  
  - Inputs: Reload Clean Job Data  
  - Outputs: Update Summary Sheet  
  - Failure cases: Empty or inconsistent data, string matching issues  

- **Update Summary Sheet**  
  - Type: Google Sheets node  
  - Role: Updates the “All Keywords combined” summary sheet with the total job count per keyword for the current date  
  - Operation: Append or update (matching on Keywords column)  
  - Inputs: Generate Keyword Summary Stats  
  - Outputs: Fetch Final Summary Data  
  - Failure cases: Sheet update race conditions, API errors  

- **Fetch Final Summary Data**  
  - Type: Google Sheets node  
  - Role: Reads the updated summary sheet data for use in email report generation  
  - Inputs: Update Summary Sheet  
  - Outputs: Build Email Body  
  - Failure cases: Sheet access errors  

- **Build Email Body**  
  - Type: Code (JavaScript)  
  - Role: Constructs an object containing the scraping date and counts by keyword type for insertion into the email template  
  - Inputs: Fetch Final Summary Data  
  - Outputs: Send Daily Report Email  
  - Failure cases: Missing fields, parsing errors  

- **Send Daily Report Email**  
  - Type: Gmail node  
  - Role: Sends an HTML formatted email daily report to a fixed list of recipients with scraping date, total records count, counts by type, and spreadsheet link  
  - Configuration: Uses OAuth2 credentials, static recipient list, dynamic subject and body with data placeholders  
  - Inputs: Build Email Body  
  - Outputs: None  
  - Failure cases: SMTP or OAuth errors, sending limits  

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                           | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                                                                                |
|--------------------------------|---------------------|-----------------------------------------|-----------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger Manual Run              | Manual Trigger      | Starts the workflow                      | None                        | Fetch Keywords from Google Sheet   | Job Scraping & Initial Processing                                                                                                         |
| Fetch Keywords from Google Sheet| Google Sheets       | Reads list of keywords                   | Trigger Manual Run           | Loop Through Keywords             | Job Scraping & Initial Processing                                                                                                         |
| Loop Through Keywords           | Split In Batches    | Iterates keywords for scraping           | Fetch Keywords from Google Sheet | Load Today’s Daily Jobs, Trigger Apify Scraper | Job Scraping & Initial Processing                                                                                                         |
| Trigger Apify Scraper           | HTTP Request       | Initiates Apify scraping for a keyword   | Loop Through Keywords        | Wait for Apify Completion         | Job Scraping & Initial Processing                                                                                                         |
| Wait for Apify Completion       | HTTP Request       | Waits for Apify actor run to finish      | Trigger Apify Scraper        | Delay Before Dataset Read         | Job Scraping & Initial Processing                                                                                                         |
| Delay Before Dataset Read       | Wait                | Waits 20 seconds before reading dataset  | Wait for Apify Completion    | Fetch Scraped Job Dataset         | Job Scraping & Initial Processing                                                                                                         |
| Fetch Scraped Job Dataset       | HTTP Request       | Retrieves scraped job data from Apify    | Delay Before Dataset Read    | Process Raw Job Data              | Job Scraping & Initial Processing                                                                                                         |
| Process Raw Job Data            | Code (JavaScript)  | Filters jobs posted in last 23 hours     | Fetch Scraped Job Dataset    | Save Jobs to Daily Sheet          | Job Scraping & Initial Processing                                                                                                         |
| Save Jobs to Daily Sheet        | Google Sheets       | Appends filtered jobs to daily sheet     | Process Raw Job Data         | Update Keyword Job Count          | Job Scraping & Initial Processing                                                                                                         |
| Update Keyword Job Count        | Google Sheets       | Updates job count per keyword             | Save Jobs to Daily Sheet     | Loop Through Keywords             | Job Scraping & Initial Processing                                                                                                         |
| Load Today’s Daily Jobs         | Google Sheets       | Loads current daily jobs data             | Loop Through Keywords        | Remove Duplicates by Title/Desc   | Data Cleaning & Deduplication                                                                                                             |
| Remove Duplicates by Title/Desc | Code (JavaScript)  | Removes duplicate jobs by title & desc   | Load Today’s Daily Jobs      | Save Clean Job Data               | Data Cleaning & Deduplication                                                                                                             |
| Save Clean Job Data             | Google Sheets       | Saves cleaned unique job data             | Remove Duplicates by Title/Desc | Delete rows or columns from sheet | Data Cleaning & Deduplication                                                                                                             |
| Delete rows or columns from sheet | Google Sheets     | Deletes old rows in daily sheet           | Save Clean Job Data          | Reload Clean Job Data             | Data Cleaning & Deduplication                                                                                                             |
| Reload Clean Job Data           | Google Sheets       | Reloads cleaned job data                   | Delete rows or columns from sheet | Generate Keyword Summary Stats   | Data Cleaning & Deduplication                                                                                                             |
| Generate Keyword Summary Stats  | Code (JavaScript)  | Counts jobs per keyword                    | Reload Clean Job Data        | Update Summary Sheet             | Daily Summary & Email Report                                                                                                              |
| Update Summary Sheet            | Google Sheets       | Updates summary sheet with keyword counts | Generate Keyword Summary Stats | Fetch Final Summary Data          | Daily Summary & Email Report                                                                                                              |
| Fetch Final Summary Data        | Google Sheets       | Reads summary data for reporting          | Update Summary Sheet         | Build Email Body                 | Daily Summary & Email Report                                                                                                              |
| Build Email Body               | Code (JavaScript)  | Builds email content with stats            | Fetch Final Summary Data     | Send Daily Report Email           | Daily Summary & Email Report                                                                                                              |
| Send Daily Report Email         | Gmail               | Sends the daily summary email              | Build Email Body             | None                            | Daily Summary & Email Report                                                                                                              |
| Delete rows or columns from sheet | Google Sheets     | Deletes rows from daily sheet (old data)  | Save Clean Job Data          | Reload Clean Job Data             | Data Cleaning & Deduplication                                                                                                             |
| Sticky Note                    | Sticky Note         | Description of Job Scraping & Initial Processing block | None                        | None                            | See note content                                                                                                                          |
| Sticky Note1                   | Sticky Note         | Description of Data Cleaning & Deduplication block | None                        | None                            | See note content                                                                                                                          |
| Sticky Note2                   | Sticky Note         | Description of Daily Summary & Email Report block | None                        | None                            | See note content                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: Trigger Manual Run  
   - Type: Manual Trigger  
   - No parameters  

2. **Create Google Sheets Node to Fetch Keywords**  
   - Name: Fetch Keywords from Google Sheet  
   - Operation: Read rows from sheet  
   - Google Sheet Document ID: (Your Google Sheet ID)  
   - Sheet Name: “All Keywords combined” (or equivalent)  
   - Authentication: Service Account with appropriate permissions  
   - Connect Trigger Manual Run → Fetch Keywords from Google Sheet  

3. **Create Split In Batches Node**  
   - Name: Loop Through Keywords  
   - Purpose: Iterate over each keyword (default batch size 1)  
   - Connect Fetch Keywords from Google Sheet → Loop Through Keywords  

4. **Create Google Sheets Node to Load Today’s Daily Jobs**  
   - Name: Load Today’s Daily Jobs  
   - Operation: Read rows from sheet  
   - Document ID and Sheet Name: Daily job sheet (e.g., “15 July 2025”)  
   - Authentication: Service Account  
   - Connect Loop Through Keywords → Load Today’s Daily Jobs  

5. **Create HTTP Request Node to Trigger Apify Scraper**  
   - Name: Trigger Apify Scraper  
   - Method: POST  
   - URL: https://api.apify.com/v2/acts/XYTgO05GT5qAoSlxy/runs  
   - Headers: Content-Type: application/json  
   - Authentication: HTTP Header Auth with Apify API key  
   - Body (JSON):  
     ```json
     {
       "clientHistory": ["noHires", "1to9Hires", "10+Hires"],
       "experienceLevel": ["entry", "intermediate", "expert"],
       "maxJobAge": {"value":23,"unit":"hours"},
       "paymentVerified": false,
       "query": "{{ $json.Keywords }}",
       "page": 1,
       "perPage": 100,
       "sort": "newest"
     }
     ```  
   - Connect Loop Through Keywords → Trigger Apify Scraper  

6. **Create HTTP Request Node to Wait for Apify Completion**  
   - Name: Wait for Apify Completion  
   - Method: GET  
   - URL: https://api.apify.com/v2/actor-runs/{{ $json.data.id }}  
   - Query Parameter: waitForFinish=150  
   - Authentication: HTTP Header Auth with Apify API key  
   - Connect Trigger Apify Scraper → Wait for Apify Completion  

7. **Create Wait Node**  
   - Name: Delay Before Dataset Read  
   - Duration: 20 seconds  
   - Connect Wait for Apify Completion → Delay Before Dataset Read  

8. **Create HTTP Request Node to Fetch Scraped Dataset**  
   - Name: Fetch Scraped Job Dataset  
   - Method: GET  
   - URL: https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items  
   - Authentication: HTTP Header Auth with Apify API key  
   - Connect Delay Before Dataset Read → Fetch Scraped Job Dataset  

9. **Create Code Node to Process Raw Job Data**  
   - Name: Process Raw Job Data  
   - JavaScript:
     ```js
     const now = new Date();
     const hoursThreshold = 23;
     const thresholdTime = new Date(now.getTime() - hoursThreshold * 60 * 60 * 1000);

     const filteredJobs = $input.all().filter(item => {
       const jobDate = new Date(item.json.absoluteDate);
       return jobDate > thresholdTime;
     });

     return filteredJobs;
     ```
   - Connect Fetch Scraped Job Dataset → Process Raw Job Data  

10. **Create Google Sheets Node to Save Jobs to Daily Sheet**  
    - Name: Save Jobs to Daily Sheet  
    - Operation: Append rows  
    - Document ID and Sheet Name: Daily job sheet (e.g., “15 July 2025”)  
    - Map job fields (title, description, url, budget, jobType, absoluteDate, relativeDate, Keyword Title, clientLocation, experienceLevel, paymentVerified, tags, allowedApplicantCountries)  
    - Authentication: Service Account  
    - Connect Process Raw Job Data → Save Jobs to Daily Sheet  

11. **Create Google Sheets Node to Update Keyword Job Count**  
    - Name: Update Keyword Job Count  
    - Operation: Append or Update (match on Keywords column)  
    - Document ID and Sheet Name: “All Keywords combined”  
    - Map: Keywords, Total Count for today (count of jobs processed)  
    - Authentication: Service Account  
    - Connect Save Jobs to Daily Sheet → Update Keyword Job Count  

12. **Connect Update Keyword Job Count → Loop Through Keywords** (to process next keyword)  

13. **Create Code Node to Remove Duplicates by Title/Desc**  
    - Name: Remove Duplicates by Title/Desc  
    - JavaScript:
      ```js
      const seen = new Set();
      const uniqueItems = [];

      for (const item of $input.all()) {
        const title = item.json.title?.trim().toLowerCase() || '';
        const description = item.json.description?.trim().toLowerCase() || '';
        const key = `${title}|||${description}`;

        if(title !== "no records found in last 24 hours") {
          if (!seen.has(key)) {
            seen.add(key);
            uniqueItems.push(item);
          }
        }
      }

      return uniqueItems;
      ```
    - Connect Load Today’s Daily Jobs → Remove Duplicates by Title/Desc  

14. **Create Google Sheets Node to Save Clean Job Data**  
    - Name: Save Clean Job Data  
    - Operation: Append rows  
    - Document ID and Sheet Name: Daily job sheet  
    - Map all relevant job fields  
    - Authentication: Service Account  
    - Connect Remove Duplicates by Title/Desc → Save Clean Job Data  

15. **Create Google Sheets Node to Delete Rows or Columns from Sheet**  
    - Name: Delete rows or columns from sheet  
    - Operation: Delete rows  
    - Document ID and Sheet Name: Daily job sheet  
    - Number to delete: Use expression to delete as many rows as loaded in “Load Today’s Daily Jobs”  
    - Authentication: Service Account  
    - Connect Save Clean Job Data → Delete rows or columns from sheet  

16. **Create Google Sheets Node to Reload Clean Job Data**  
    - Name: Reload Clean Job Data  
    - Operation: Read rows  
    - Document ID and Sheet Name: Daily job sheet  
    - Authentication: Service Account  
    - Connect Delete rows or columns from sheet → Reload Clean Job Data  

17. **Create Code Node to Generate Keyword Summary Stats**  
    - Name: Generate Keyword Summary Stats  
    - JavaScript:
      ```js
      const keywordsInput = $('Reload Clean Job Data').all().map(item => item.json.Keywords);
      const counts = [];

      for (const keyword of keywordsInput) {
        let count = 0;

        for (const job of $('Save Clean Job Data').all()) {
          const content = job.json['Keyword Title'];
          if (content.includes(keyword)) {
            if (job.json.title !== "No records found in last 24 hours") {
              count++;
            }
          }
        }

        counts.push({
          json: {
            keyword: keyword,
            count: count,
            date: '15-07-2025' // adjust dynamically as needed
          }
        });
      }

      return counts;
      ```
    - Connect Reload Clean Job Data → Generate Keyword Summary Stats  

18. **Create Google Sheets Node to Update Summary Sheet**  
    - Name: Update Summary Sheet  
    - Operation: Append or Update (match on Keywords)  
    - Document ID and Sheet Name: “All Keywords combined” summary sheet  
    - Map: Keywords, Total Count for today  
    - Authentication: Service Account  
    - Connect Generate Keyword Summary Stats → Update Summary Sheet  

19. **Create Google Sheets Node to Fetch Final Summary Data**  
    - Name: Fetch Final Summary Data  
    - Operation: Read rows  
    - Document ID and Sheet Name: “All Keywords combined” summary sheet  
    - Authentication: Service Account  
    - Connect Update Summary Sheet → Fetch Final Summary Data  

20. **Create Code Node to Build Email Body**  
    - Name: Build Email Body  
    - JavaScript:
      ```js
      const rows = $input.all();
      const totals = {};

      for (const row of rows) {
        const type = row.json["Keyword Type"];
        const count = parseInt(row.json["Total Count 15-07-2025"]) || 0;

        totals[type] = (totals[type] || 0) + count;
      }

      return [{
        json: {
          date: "15-07-2025",  // Adjust dynamically
          counts_by_type: totals
        }
      }];
      ```
    - Connect Fetch Final Summary Data → Build Email Body  

21. **Create Gmail Node to Send Daily Report Email**  
    - Name: Send Daily Report Email  
    - Operation: Send Email  
    - Recipients: bd@itoneclick.com, james@itoneclick.com, nikunj@itoneclick.com, drashti@itoneclick.com, sahil@itoneclick.com  
    - Subject: Apify data details - 15 July 2025 (dynamic date)  
    - Message (HTML): Includes greeting, scraping date, total records, counts by type, and link to Google Sheet  
    - Authentication: OAuth2 Gmail credentials  
    - Connect Build Email Body → Send Daily Report Email  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                                            |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| This workflow uses Apify API to scrape Upwork job listings by keyword with filters on client history, experience level, and job age.                            | Apify API documentation: https://docs.apify.com/api/v2                                                                 |
| Google Sheets operations use a service account for authentication to read/write data securely.                                                                  | Google Sheets API and Service Account setup documentation                                                                |
| Email sending uses Gmail OAuth2 for secure SMTP access; ensure correct OAuth2 credentials and scopes are configured.                                             | Gmail OAuth2 setup: https://developers.google.com/identity/protocols/oauth2                                               |
| The workflow includes manual trigger to allow controlled execution and testing.                                                                                  |                                                                                                                            |
| Deduplication logic compares title and description lowercase trimmed strings to remove near-exact duplicates.                                                   |                                                                                                                            |
| The workflow design includes wait and delay nodes to handle asynchronous Apify API completion and dataset readiness, avoiding race conditions.                  |                                                                                                                            |
| Spreadsheet links in emails are dynamic to allow quick access to updated job data.                                                                               |                                                                                                                            |
| Sticky notes in the original workflow provide block-level explanations and can be used as in-app documentation for maintenance and onboarding.                   |                                                                                                                            |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. The content strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.