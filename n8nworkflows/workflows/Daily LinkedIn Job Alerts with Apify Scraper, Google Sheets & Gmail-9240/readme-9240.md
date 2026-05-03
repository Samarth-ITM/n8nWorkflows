Daily LinkedIn Job Alerts with Apify Scraper, Google Sheets & Gmail

https://n8nworkflows.xyz/workflows/daily-linkedin-job-alerts-with-apify-scraper--google-sheets---gmail-9240


# Daily LinkedIn Job Alerts with Apify Scraper, Google Sheets & Gmail

### 1. Workflow Overview

This workflow automates the daily process of scraping LinkedIn job listings based on predefined criteria, storing the results in a Google Sheet, and emailing a formatted summary to a user. It targets professionals or recruiters who want to receive daily updates on relevant job postings without manual searching.

Logical blocks include:

- **1.1 Scheduled Start:** Trigger the workflow every day at noon.
- **1.2 Data Scraping:** Use the Apify LinkedIn Jobs Scraper actor to fetch recent job listings with specified filters.
- **1.3 Data Formatting & Storage:** Format each job listing into HTML for email and simultaneously append raw job data to a Google Sheet for archival and review.
- **1.4 Data Aggregation:** Combine all individual job entries into a single list for email content generation.
- **1.5 Email Delivery:** Send a styled email summarizing the day’s job listings via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Start

- **Overview:**  
  Triggers the workflow automatically every day at 12:00 PM to ensure daily job alerts.

- **Nodes Involved:**  
  - Every day at noon...

- **Node Details:**  
  - **Node:** Every day at noon...  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger once daily at hour 12 (noon).  
    - Inputs: None  
    - Outputs: Triggers the "Get LinkedIn jobs" node  
    - Edge cases: Scheduler misfire or time zone mismatch may cause delayed or missed runs.  
    - Version: 1.2  

#### 1.2 Data Scraping

- **Overview:**  
  Runs a scraping actor on Apify to extract recent LinkedIn job postings matching the keyword, location, and other filters.

- **Nodes Involved:**  
  - Get LinkedIn jobs  
  - Sticky Note (Get LinkedIn jobs description)

- **Node Details:**  
  - **Node:** Get LinkedIn jobs  
    - Type: Apify Community Node (Actor Runner)  
    - Configuration:  
      - Actor ID: KE649tixwpoRnZtJJ (LinkedIn Jobs Scraper - No Cookies)  
      - Operation: Run actor and retrieve dataset  
      - Input JSON:  
        ```json
        {
          "date_posted": "day",
          "keywords": "SEO manager",
          "limit": 10,
          "location": "United States",
          "sort": "recent"
        }
        ```  
      - Credentials: Apify API key  
    - Inputs: Trigger from schedule  
    - Outputs: Array of job items with fields like company, job_title, job_url, location, work_type, salary  
    - Edge cases:  
      - API limits or errors from Apify service  
      - Scraper downtime or LinkedIn structure changes causing empty or malformed data  
      - Network timeouts  
    - Version: 1  
    - Sticky Note content:  
      Explains usage of Apify LinkedIn Jobs Scraper actor and suggests HTTP node alternative if community node unavailable.

#### 1.3 Data Formatting & Storage

- **Overview:**  
  Transforms each job item into HTML format for emails and appends job data to a Google Sheet for record keeping.

- **Nodes Involved:**  
  - Set job data & HTML  
  - Add jobs to Google Sheet  
  - Sticky Note1 (Set job data and Google Sheets)

- **Node Details:**  
  - **Node:** Set job data & HTML  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Mode: Run once for each input item (process each job separately)  
      - Script summarizes job info into an HTML snippet with fallback for missing salary or work type:  
        ```js
        const company = $input.item.json.company;
        const job_title = $input.item.json.job_title;
        const job_url = $input.item.json.job_url;
        const location = $input.item.json.location;
        const work_type = $input.item.json.work_type ? $input.item.json.work_type : 'Not specified';
        const salary = $input.item.json.salary ? $input.item.json.salary : "Not specified";

        const html_item = `<b><a href="${job_url}">${job_title} - ${company}</a></b><br />Salary: ${salary} <br />Location: ${location} <br />Remote or on-site: ${work_type} <br />`;

        return {
          json: {
            html: html_item,
          },
        };
        ```  
    - Inputs: Job data from Apify node  
    - Outputs: JSON object with `html` property for each job  
    - Edge cases: Null or missing fields handled by fallback strings; risk if input JSON schema changes drastically  
    - Version: 2  

  - **Node:** Add jobs to Google Sheet  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append rows  
      - Sheet: Specified Google Sheet by document ID and sheet GID  
      - Columns mapped: Date (formatted from posted_at), salary, company, job url, location, job title, remote or on-site  
      - Uses OAuth2 credentials for Google Sheets API  
    - Inputs: Job data from Apify node (parallel to code node)  
    - Outputs: Confirmation of appended rows  
    - Edge cases:  
      - Google Sheets API quota or permission errors  
      - Incorrect sheet or document ID causing failures  
      - Data type mismatches or formatting errors  
    - Version: 4.7  
    - Sticky Note content: Explains adding jobs to Google Sheets for future reference and formatting HTML email content.  

#### 1.4 Data Aggregation

- **Overview:**  
  Aggregates all individual formatted HTML job entries into a single list for email body construction.

- **Nodes Involved:**  
  - Combine items into one list  
  - Sticky Note2 (Combine items description)

- **Node Details:**  
  - **Node:** Combine items into one list  
    - Type: Aggregate node  
    - Configuration: Aggregate all item data into a single list  
    - Inputs: Multiple items from the "Set job data & HTML" node  
    - Outputs: One consolidated item containing an array of all HTML snippets  
    - Edge cases: Empty input array results in empty output; no special error handling needed  
    - Version: 1  

#### 1.5 Email Delivery

- **Overview:**  
  Sends a formatted email via Gmail containing the combined daily job listings with a date-specific subject line.

- **Nodes Involved:**  
  - Send a message  
  - Sticky Note3 (Email sending description)

- **Node Details:**  
  - **Node:** Send a message  
    - Type: Gmail node  
    - Configuration:  
      - Subject: Dynamic, e.g., "Job summary for 15 June 2024" using expression `={{ $now.format('dd MMMM yyyy')}}`  
      - Message body: HTML composed by mapping over the aggregated job HTML snippets joined by `<br/>` tags  
      - Options: Attribution disabled  
      - OAuth2 credentials for Gmail API  
    - Inputs: Aggregated list from previous node  
    - Outputs: Email sent confirmation  
    - Edge cases:  
      - Gmail API quota or permission issues  
      - Malformed HTML could affect email rendering  
      - Network issues during sending  
    - Version: 2.1  

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                       | Input Node(s)           | Output Node(s)          | Sticky Note                                                                           |
|---------------------------|---------------------------|-------------------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------|
| Every day at noon...       | Schedule Trigger          | Trigger workflow daily at noon      | -                      | Get LinkedIn jobs        | ## Scheduled trigger<br>Run this workflow every day at noon                          |
| Get LinkedIn jobs         | Apify Community Node       | Scrape LinkedIn jobs via Apify      | Every day at noon...    | Set job data & HTML, Add jobs to Google Sheet | ## Get LinkedIn jobs<br>Use Apify to scrape LinkedIn jobs. This workflow uses the [`LinkedIn Jobs Scraper - No Cookies`](https://apify.com/apimaestro/linkedin-jobs-scraper-api) actor on Apify. Note that you can use a HTTP node instead of the Apify community node. |
| Set job data & HTML       | Code Node (JavaScript)     | Format job data into HTML snippet    | Get LinkedIn jobs       | Combine items into one list | ## Set job data and HTML and add to Google Sheets<br>Add the jobs to a Google Sheet so we can reference them at a later date. At the same time, we format the email HTML. |
| Add jobs to Google Sheet  | Google Sheets Node         | Append job data to Google Sheet      | Get LinkedIn jobs       | -                        | ## Set job data and HTML and add to Google Sheets<br>Add the jobs to a Google Sheet so we can reference them at a later date. |
| Combine items into one list | Aggregate Node            | Combine all job HTML snippets        | Set job data & HTML     | Send a message           | ## Combine items into one list<br>We combine all the different items into one big list |
| Send a message            | Gmail Node                 | Send email with job listings         | Combine items into one list | -                        | ## Send email<br>We send an email with a list of all the jobs titles, salaries, and more |
| Sticky Note               | Sticky Note                | Documentation and instructions       | -                      | -                        | Various notes explaining each block and setup instructions                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Scheduled Trigger**  
   - Add a **Schedule Trigger** node.  
   - Set it to trigger every day at 12:00 PM (hour = 12).  
   - Name it "Every day at noon...".

2. **Add Apify Node to Scrape LinkedIn Jobs**  
   - Add **Apify** community node.  
   - Set operation to "Run actor and get dataset".  
   - Use actor ID `KE649tixwpoRnZtJJ` (LinkedIn Jobs Scraper - No Cookies).  
   - Configure input JSON:  
     ```json
     {
       "date_posted": "day",
       "keywords": "SEO manager",
       "limit": 10,
       "location": "United States",
       "sort": "recent"
     }
     ```  
   - Connect "Every day at noon..." node to this node.  
   - Add Apify API credentials.

3. **Add Code Node to Format Each Job Item**  
   - Add **Code** node, set mode to "Run once for each item".  
   - Paste JavaScript code to create an HTML snippet with job title, company, salary, location, and work type with fallbacks.  
   - Connect output of Apify node to this node.

4. **Add Google Sheets Node to Store Job Data**  
   - Add **Google Sheets** node, operation "Append".  
   - Set document ID and sheet name (gid=0).  
   - Map columns to job fields: Date (formatted from posted_at), salary, company, job url, location, job title, remote or on-site.  
   - Connect output of Apify node to this node (parallel to Code node).  
   - Configure Google Sheets OAuth2 credentials.

5. **Add Aggregate Node to Combine HTML Snippets**  
   - Add **Aggregate** node.  
   - Set aggregation to combine all items into a single array.  
   - Connect output of Code node to this node.

6. **Add Gmail Node to Send Email**  
   - Add **Gmail** node.  
   - Configure subject as `"Job summary for {{ $now.format('dd MMMM yyyy') }}"`.  
   - Configure message body as HTML combining all job HTML snippets joined by `<br/>`.  
   - Disable attribution.  
   - Connect output of Aggregate node to this node.  
   - Configure Gmail OAuth2 credentials.

7. **Connect Nodes in Order**:  
   - Schedule Trigger → Apify → (parallel) Code & Google Sheets → Code → Aggregate → Gmail Send.

8. **Add Optional Sticky Notes** for documentation at each block to help future maintenance.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Every day, this workflow scrapes LinkedIn jobs based on your keywords, saves them in a Google Sheet, and sends them by email. | Full workflow overview and setup details included in sticky note. |
| [LinkedIn Jobs Scraper - No Cookies](https://apify.com/apimaestro/linkedin-jobs-scraper-api) | Apify actor used for scraping LinkedIn jobs. |
| To use HTTP node instead of Apify community node, check Apify API docs: https://docs.apify.com/api/v2 | Alternative integration approach. |
| Requirements: Apify account, Google Sheets and Gmail API credentials with OAuth2 set up in n8n | Setup prerequisites. |
| Possible customizations: add full job descriptions, generate tailored CVs, use AI to extract key skills from job descriptions | Suggested enhancements for advanced users. |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, adhering strictly to current content policies without containing illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.