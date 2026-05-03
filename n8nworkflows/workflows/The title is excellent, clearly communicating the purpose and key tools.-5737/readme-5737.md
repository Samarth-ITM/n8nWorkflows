The title is excellent, clearly communicating the purpose and key tools.

https://n8nworkflows.xyz/workflows/the-title-is-excellent--clearly-communicating-the-purpose-and-key-tools--5737


# The title is excellent, clearly communicating the purpose and key tools.

### 1. Workflow Overview

This workflow automates the process of scraping recent job listings from the Y Combinator jobs page, cleans and structures the scraped data, and appends it to a Google Sheets document for tracking and analysis. It is designed to run every 6 hours, ensuring fresh job data is continually updated.

The workflow is logically divided into three main blocks:

- **1.1 Scheduled Trigger and Data Scraping:** Initiates the workflow on a fixed time interval and performs web crawling using Scrapeless to retrieve job listings.
- **1.2 Data Cleaning and Transformation:** Processes the raw scraped markdown data to extract meaningful job information, including introductory text and individual job titles with links.
- **1.3 Data Storage:** Saves the cleaned and structured job data into a specified Google Sheets spreadsheet for persistent storage and easy access.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Data Scraping

**Overview:**  
This block periodically triggers the workflow every 6 hours and scrapes job listings from the Y Combinator jobs webpage using the Scrapeless crawler service.

**Nodes Involved:**  
- Schedule Trigger  
- Scrapeless  

**Node Details:**

- **Schedule Trigger**  
  - *Type & Role:* Time-based trigger node; initiates the workflow every 6 hours.  
  - *Configuration:* Set to trigger at 6-hour intervals using the built-in scheduling options.  
  - *Key Parameters:* Interval field specifying hours = 6.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connects to Scrapeless node to start scraping.  
  - *Edge Cases:* Potential failure if the n8n instance is down or scheduling misconfiguration occurs.  
  - *Version Requirements:* Compatible with n8n versions supporting v1.2 of the schedule trigger node.

- **Scrapeless**  
  - *Type & Role:* Web crawler node using Scrapeless API to fetch web content.  
  - *Configuration:*  
    - URL: `https://www.ycombinator.com/jobs`  
    - Operation: `crawl` resource to scrape pages.  
    - Limit: Crawls up to 2 pages to limit volume and runtime.  
    - Credential: Uses Scrapeless API credentials for authentication.  
  - *Key Parameters:* URL target, page crawl limit, API credential.  
  - *Inputs:* Triggered by Schedule Trigger.  
  - *Outputs:* Raw scraped data passed as markdown to the next node.  
  - *Edge Cases:*  
    - API authentication failure.  
    - Network or site unavailability.  
    - Rate limits or crawl restrictions.  
  - *Version Requirements:* Requires Scrapeless node support (version 1 or later).

---

#### 1.2 Data Cleaning and Transformation

**Overview:**  
This block refines the raw markdown data scraped from the site. It extracts an introductory text section and a list of job postings with their titles and hyperlinks, transforming the data into a structured format suitable for storage.

**Nodes Involved:**  
- Code  
- Code1  
- Code2  
- Sticky Note1 (provides descriptive context)

**Node Details:**

- **Code**  
  - *Type & Role:* JavaScript code node that extracts the markdown content from the Scrapeless output.  
  - *Configuration:*  
    - Extracts the `markdown` field from each scraped item and returns it as a new item array.  
  - *Key Expressions:* Accesses `items[0].json` and maps to output `markdown` field.  
  - *Inputs:* Raw Scrapeless data.  
  - *Outputs:* Markdown content for further processing.  
  - *Edge Cases:* Empty or malformed data may cause no output or empty arrays.  
  - *Version Requirements:* Uses v2 code node syntax.

- **Code1**  
  - *Type & Role:* JavaScript node that parses the markdown to separate the introduction and individual job entries.  
  - *Configuration:*  
    - Uses regex to locate the "jobs added recently" header dynamically (case-insensitive).  
    - Splits the markdown into intro and jobs sections.  
    - Cleans the intro text by removing headings.  
    - Extracts jobs as objects with `title` and `link` ignoring image-only links.  
  - *Key Expressions:* Regex for splitting and matching job list markdown links.  
  - *Inputs:* Markdown output from the previous node.  
  - *Outputs:* JSON objects with `intro` and `jobs` array.  
  - *Edge Cases:*  
    - Missing or renamed header may cause parsing failure.  
    - Unexpected markdown formatting affecting extraction.  
  - *Version Requirements:* v2 code node.

- **Code2**  
  - *Type & Role:* JavaScript node that flattens the jobs array, producing one item per job with the intro text repeated.  
  - *Configuration:*  
    - Iterates through each job entry, creating a new item with `intro`, `jobTitle`, and `jobLink`.  
  - *Inputs:* JSON objects with intro and jobs arrays.  
  - *Outputs:* Flat list of job entries suitable for tabular storage.  
  - *Edge Cases:* Empty jobs array results in no output items.  
  - *Version Requirements:* v2 code node.

- **Sticky Note1**  
  - *Role:* Provides a descriptive label for this block indicating it handles data cleaning and labeling based on patterns and keywords.

---

#### 1.3 Data Storage

**Overview:**  
This block appends the cleaned job data into a Google Sheets document, enabling easy access and ongoing tracking.

**Nodes Involved:**  
- Google Sheets  
- Sticky Note2 (contextual description)

**Node Details:**

- **Google Sheets**  
  - *Type & Role:* Google Sheets node appends rows to a spreadsheet.  
  - *Configuration:*  
    - Operation: Append rows to a specific sheet.  
    - Document ID: Hardcoded Google Sheets document ID.  
    - Sheet Name: Uses sheet with GID 0 (Sheet1).  
    - Columns mapped: `"Job Title"` and `"Job Link"` mapped from item fields `jobTitle` and `jobLink`.  
    - Matching Columns: Uses `Job Title` for potential deduplication or matching (but operation is append, so matching may be redundant).  
    - Credential: Uses Google Sheets OAuth2 credential for authentication.  
  - *Inputs:* Structured job entries from Code2 node.  
  - *Outputs:* None (end of workflow).  
  - *Edge Cases:*  
    - Credential expiration or permission errors.  
    - Spreadsheet not found or access revoked.  
    - API rate limits or quota exceeded.  
  - *Version Requirements:* Supports n8n node version 4.6 or later.

- **Sticky Note2**  
  - *Role:* Provides a label emphasizing the role of saving structured data to Google Sheets for analysis.

---

### 3. Summary Table

| Node Name       | Node Type                   | Functional Role                         | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                         |
|-----------------|-----------------------------|---------------------------------------|---------------------|---------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger| scheduleTrigger              | Initiates workflow every 6 hours      | None                | Scrapeless          | ##  Automated Job Scraper (Every 6 Hours) ⏰ Schedule Trigger: Runs every 6 hours to fetch fresh data. 🕸️ Crawl Step: Scrapes job listings from a specific website using Scrapeless. |
| Scrapeless      | scrapeless                  | Scrapes job listings from target URL | Schedule Trigger    | Code                | ##  Automated Job Scraper (Every 6 Hours) ⏰ Schedule Trigger: Runs every 6 hours to fetch fresh data. 🕸️ Crawl Step: Scrapes job listings from a specific website using Scrapeless. |
| Code            | code                        | Extracts raw markdown from scrape     | Scrapeless          | Code1               |                                                                                                   |
| Code1           | code                        | Parses markdown, extracts intro & jobs| Code               | Code2                | ## Data Cleaning & Labeling Functions These functions clean the raw job data and assign appropriate labels  based on keywords and patterns. |
| Code2           | code                        | Flattens jobs array to individual items| Code1              | Google Sheets        | ## Data Cleaning & Labeling Functions These functions clean the raw job data and assign appropriate labels  based on keywords and patterns. |
| Google Sheets   | googleSheets                | Appends cleaned job data to spreadsheet| Code2               | None                 | ## Save to Google Sheets After cleaning and labeling, the final structured job data is stored in Google Sheets for easy access and analysis. |
| Sticky Note     | stickyNote                  | Explains schedule and crawl steps     | None                | None                 | ##  Automated Job Scraper (Every 6 Hours) ⏰ Schedule Trigger: Runs every 6 hours to fetch fresh data. 🕸️ Crawl Step: Scrapes job listings from a specific website using Scrapeless. |
| Sticky Note1    | stickyNote                  | Describes data cleaning & labeling    | None                | None                 | ## Data Cleaning & Labeling Functions These functions clean the raw job data and assign appropriate labels  based on keywords and patterns. |
| Sticky Note2    | stickyNote                  | Describes Google Sheets storage       | None                | None                 | ## Save to Google Sheets After cleaning and labeling, the final structured job data is stored in Google Sheets for easy access and analysis. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set the interval to run every 6 hours (hoursInterval = 6).  
   - This node will start the workflow on the specified schedule.

2. **Add a Scrapeless Node**  
   - Type: Scrapeless  
   - Configure to crawl the URL: `https://www.ycombinator.com/jobs`.  
   - Set resource to `crawler` and operation to `crawl`.  
   - Limit the crawl to 2 pages to control the data volume.  
   - Set Scrapeless API credentials (create or import the appropriate API credential for Scrapeless).  
   - Connect the output of the Schedule Trigger node to this Scrapeless node.

3. **Add a Code Node (“Code”) for Markdown Extraction**  
   - Type: Code (JavaScript)  
   - Paste the following script to extract markdown from the Scrapeless output:
     ```javascript
     const raw = items[0].json;             
     const output = raw.map(obj => ({
       json: {
         markdown: obj.markdown,
       }
     }));
     return output;
     ```  
   - Connect the Scrapeless node output to this Code node.

4. **Add a Code Node (“Code1”) for Parsing Markdown**  
   - Type: Code (JavaScript)  
   - Paste the following script to parse the intro and jobs list:
     ```javascript
     return items.map(item => {
       const md = item.json.markdown;
       const splitRegex = /^#{1,3}\s*.+jobs added recently\s*$/im;
       const parts = md.split(splitRegex);
       const introSectionRaw = parts[0] || '';
       const jobsSectionRaw = parts.slice(1).join('') || '';

       const intro = introSectionRaw
         .replace(/^#+\s*/gm, '')
         .trim();

       const jobs = [];
       const re = /\-\s*\[(?!\!)([^\]]+)\]\((https?:\/\/[^\)]+)\)/g;
       let match;
       while ((match = re.exec(jobsSectionRaw))) {
         jobs.push({
           title: match[1].trim(),
           link:  match[2].trim(),
         });
       }

       return {
         json: {
           intro,
           jobs,
         },
       };
     });
     ```  
   - Connect the output of the previous Code node to this node.

5. **Add a Code Node (“Code2”) to Flatten Job Entries**  
   - Type: Code (JavaScript)  
   - Paste the following script to create one item per job listing:
     ```javascript
     const output = [];

     items.forEach(item => {
       const intro = item.json.intro;
       const jobs  = item.json.jobs || [];

       jobs.forEach(job => {
         output.push({
           json: {
             intro,             
             jobTitle: job.title,
             jobLink:  job.link
           }
         });
       });
     });

     return output;
     ```  
   - Connect the output of the previous Code1 node to this node.

6. **Add a Google Sheets Node**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use your Google Sheets document ID for storing job data.  
   - Sheet Name: Use the target sheet (e.g., "Sheet1" or GID=0).  
   - Set columns mapping for:  
     - `Job Title` mapped from `jobTitle` field.  
     - `Job Link` mapped from `jobLink` field.  
   - Matching Columns: Set to `Job Title` (optional, based on your deduplication needs).  
   - Configure Google Sheets OAuth2 credentials with sufficient permissions to append data.  
   - Connect the output of the Code2 node to this Google Sheets node.

7. **(Optional) Add Sticky Note Nodes for Documentation**  
   - Create Sticky Note nodes to document each logical block for clarity and maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow is configured to run every 6 hours to balance data freshness and API usage limits.      | Scheduling best practices for job scraping workflows.                                                    |
| Scrapeless crawler is used to avoid complex manual scraping setup; ensure Scrapeless API credentials are kept current. | [Scrapeless API Documentation](https://scrapeless.com/docs)                                             |
| Google Sheets is used as a simple, accessible data store for job listings; ensure the sheet ID and permissions are correct. | Google Sheets API Authentication and Permissions [Google Developers](https://developers.google.com/sheets/api) |
| Regex parsing is sensitive to markdown format; if the target site changes structure, adjust regex accordingly. | Regular expression debugging tools like regex101.com help in adapting patterns.                           |
| Consider adding error handling nodes or notifications for failures in scraping, parsing, or Google Sheets appending. | n8n error workflow documentation.                                                                         |

---

**Disclaimer:**  
The text provided above is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and publicly accessible.