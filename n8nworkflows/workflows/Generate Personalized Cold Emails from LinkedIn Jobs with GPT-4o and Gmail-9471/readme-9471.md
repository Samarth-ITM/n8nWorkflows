Generate Personalized Cold Emails from LinkedIn Jobs with GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/generate-personalized-cold-emails-from-linkedin-jobs-with-gpt-4o-and-gmail-9471


# Generate Personalized Cold Emails from LinkedIn Jobs with GPT-4o and Gmail

### 1. Workflow Overview

This workflow automates the generation and personalized sending of cold outreach emails targeting Belgian companies hiring marketing managers, using LinkedIn job postings as data sources. It scrapes LinkedIn job listings, filters and enriches the data with decision-maker emails, generates tailored email content with GPT-4o, and drafts emails via Gmail.

**Target Use Cases:**  
- Marketing agencies seeking to automate outreach to companies actively hiring marketing managers in Belgium.  
- Sales teams wanting to leverage AI-generated personalized messaging based on real-time job posting data.  
- Lead enrichment and campaign management integrated with Google Sheets and Gmail.

**Logical Blocks:**  
- **1.1 Scheduled Data Gathering:** Periodic triggering and LinkedIn job scraping.  
- **1.2 Data Filtering & Deduplication:** Remove duplicates and filter jobs and companies by criteria.  
- **1.3 Lead Enrichment:** Retrieve decision-maker emails (CEO, sales, marketing).  
- **1.4 Data Processing & Storage:** Extract lead names, remove duplicates, and store enriched leads in Google Sheets.  
- **1.5 AI Content Generation:** Generate personalized cold email scripts and subject lines using GPT-4o.  
- **1.6 Email Drafting:** Create Gmail drafts for review or sending.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Gathering

**Overview:**  
This block schedules daily workflow execution and scrapes LinkedIn job postings from specified URLs via an external scraping API.

**Nodes Involved:**  
- Schedule Trigger  
- Linkedin_jobs_scrapper

**Node Details:**  
- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configured to trigger daily at 08:00 AM.  
  - No inputs; output triggers downstream nodes.  
  - Potential failure: n8n scheduling misconfiguration or instance downtime.

- **Linkedin_jobs_scrapper**  
  - Type: HTTP Request  
  - Performs a POST request to Apify’s LinkedIn jobs scraper API with parameters to scrape 100 job items from two LinkedIn URLs filtered by location and seniority.  
  - Uses HTTP query authentication (token).  
  - Outputs raw scraped job data JSON.  
  - Possible failures: API rate limits, network issues, invalid token, or changes in target URLs/API.

---

#### 1.2 Data Filtering & Deduplication

**Overview:**  
This block removes duplicate companies and filters out irrelevant jobs based on company size, domain, and staffing-related companies to focus on marketing manager roles in Belgium.

**Nodes Involved:**  
- Remove_duplicates  
- Filter_staffing  
- Save_usefull_infos  
- Filter_domain_and_size

**Node Details:**  
- **Remove_duplicates**  
  - Type: Remove Duplicates  
  - Deduplicates on `companyName` field to avoid redundant processing.  
  - Input: Scraped job list  
  - Output: Unique jobs by company.

- **Filter_staffing**  
  - Type: Filter  
  - Filters jobs based on complex condition:  
    - Title contains "marketing manager" or "digital marketing manager" (case-insensitive)  
    - Location includes Belgium or related regions  
    - Seniority level matches associate, mid-senior, director, or executive  
    - Excludes companies whose description contains staffing or recruitment keywords to avoid HR agencies.  
  - Input: Deduplicated companies  
  - Output: Filtered relevant jobs.

- **Save_usefull_infos**  
  - Type: Set  
  - Extracts and saves key fields from raw job JSON into structured fields (e.g., JobUrl, companyWebsite, companySize).  
  - Input: Filtered jobs  
  - Output: Jobs with cleaned and structured data fields.

- **Filter_domain_and_size**  
  - Type: Filter  
  - Filters companies that have a non-empty `companySize`, exclude those with bit.ly URLs, and limit company size to less than 100 employees.  
  - Input: Jobs with useful info  
  - Output: Target small companies with valid domains.

- **Sticky Notes:**  
  - Emphasize filtering by company size and domain quality to target relevant prospects.

---

#### 1.3 Lead Enrichment

**Overview:**  
For each filtered job, the workflow attempts to find key decision-maker emails (CEO, sales, marketing) using the Anymailfinder API.

**Nodes Involved:**  
- Loop Over Items  
- Ceo_email  
- Sales_email  
- Marketing_email  
- Merge  
- If

**Node Details:**  
- **Loop Over Items**  
  - Type: Split in Batches  
  - Processes filtered jobs one-by-one, enabling API calls for each company.  
  - Input: Filtered companies  
  - Output: Individual company data for enrichment.

- **Ceo_email / Sales_email / Marketing_email**  
  - Type: HTTP Request  
  - Query Anymailfinder API for decision-maker emails by category (CEO, sales, marketing).  
  - Uses HTTP header authentication (API key).  
  - Configured to continue on error to avoid workflow halt if no email found.  
  - Input: Single company data  
  - Output: Decision-maker email results.

- **Merge**  
  - Type: Merge  
  - Combines outputs from the three email requests into one stream.  
  - Input: Results from CEO, sales, marketing email nodes  
  - Output: Combined lead data.

- **If**  
  - Type: Conditional  
  - Checks if API did not return 404 and if email is non-empty to filter valid leads.  
  - Input: Merged email data  
  - Output: Valid leads forwarded for next steps, invalid discarded.

- **Sticky Note:**  
  - Describes main decision maker email retrieval.

- **Potential Failures:**  
  - API rate limits or temporary unavailability.  
  - Missing or inaccurate email data.  
  - Network or authentication errors.

---

#### 1.4 Data Processing & Storage

**Overview:**  
This block refines lead data by removing duplicate emails, extracting first and last names, and appending enriched leads to a Google Sheet.

**Nodes Involved:**  
- Split Out  
- Remove_duplicates1  
- Extract_lead_name  
- Add_lead_info

**Node Details:**  
- **Split Out**  
  - Type: Split Out  
  - Extracts specific fields (`result.email`, `result.personFullName`, `result.personLinkedinUrl`) from the merged data for deduplication.  
  - Input: Valid leads from If node.

- **Remove_duplicates1**  
  - Type: Remove Duplicates  
  - Deduplicates lead emails to avoid redundant outreach.  
  - Input: Split lead fields.

- **Extract_lead_name**  
  - Type: Code (JavaScript)  
  - Parses full name into first and last names, capitalizing properly.  
  - Input: Deduplicated leads.

- **Add_lead_info**  
  - Type: Google Sheets  
  - Appends enriched lead data (email, names, job title, company info, verification status, LinkedIn URLs, etc.) into a Google Sheets document for record keeping and campaign management.  
  - Uses Google Sheets OAuth2 credentials.  
  - Input: Leads with parsed names.

- **Sticky Notes:**  
  - Notes on removing duplicates and adding comprehensive lead info.

- **Potential Failures:**  
  - Google Sheets API quota or permission issues.  
  - Data schema mismatch or invalid values.

---

#### 1.5 AI Content Generation

**Overview:**  
Generates personalized cold outreach email body and subject line using GPT-4o, based on enriched lead data.

**Nodes Involved:**  
- Generate custom script  
- Generate custom title

**Node Details:**  
- **Generate custom script**  
  - Type: OpenAI (Langchain)  
  - Uses GPT-4o model with detailed system prompt defining tone, style, structure, and personalization requirements for cold emails targeting Belgian marketing manager roles.  
  - Input: Lead and job data from Google Sheets enrichment step.  
  - Output: Personalized email text with placeholders for manual video insertion.  
  - Includes instructions to output only plain text, no markdown or formatting.

- **Generate custom title**  
  - Type: OpenAI (Langchain)  
  - Generates a concise, casual but professional email subject line (6-8 words) including the prospect’s first name and subtle hiring context to improve open rates.  
  - Input: Output from the previous node and lead data.  
  - Output: Subject line text.

- **Sticky Notes:**  
  - Explains the purpose and constraints of each GPT-based generation node.

- **Potential Failures:**  
  - OpenAI API rate limits or errors.  
  - Improper tokenization or prompt formatting.

---

#### 1.6 Email Drafting

**Overview:**  
Creates Gmail draft emails using generated subject and body text, linked to lead emails as CC for record.

**Nodes Involved:**  
- Send Draft

**Node Details:**  
- **Send Draft**  
  - Type: Gmail  
  - Creates a draft message with body content from the generated script node and CCs the lead’s email.  
  - Uses Gmail OAuth2 credentials.  
  - Input: Generated email script and lead info.  
  - Output: Draft email ready for review/send.  
  - Potential failure: Gmail API quota limits, OAuth token expiry, or invalid email addresses.

---

### 3. Summary Table

| Node Name              | Node Type                     | Functional Role                                     | Input Node(s)                          | Output Node(s)                  | Sticky Note                                                                                 |
|------------------------|-------------------------------|----------------------------------------------------|--------------------------------------|--------------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger              | Triggers workflow daily at 8 AM                     | None                                 | Linkedin_jobs_scrapper          | ## Everyday we scrap jobs from multiple searchs in linkedin jobs                           |
| Linkedin_jobs_scrapper | HTTP Request                 | Scrapes LinkedIn jobs via API                        | Schedule Trigger                     | Remove_duplicates               |                                                                                             |
| Remove_duplicates      | Remove Duplicates            | Deduplicates companies by companyName               | Linkedin_jobs_scrapper               | Filter_staffing                 | ## Filter human ressource or staffing company as well as duplicate and save the remaining infos |
| Filter_staffing        | Filter                      | Filters marketing manager jobs in Belgium and excludes staffing firms | Remove_duplicates               | Save_usefull_infos             | ## Find main decisions maker emails on each relevant company                                |
| Save_usefull_infos     | Set                         | Extracts and saves key job and company info         | Filter_staffing                     | Filter_domain_and_size          |                                                                                             |
| Filter_domain_and_size | Filter                      | Filters companies by domain validity and size < 100| Save_usefull_infos                  | Loop Over Items                | ### Target specific company size                                                            |
| Loop Over Items        | Split in Batches            | Processes each company/job for email enrichment     | Filter_domain_and_size              | Ceo_email, Sales_email, Marketing_email | ## Find main decisions maker emails on each relevant company                                |
| Ceo_email              | HTTP Request                 | Retrieves CEO email from Anymailfinder API          | Loop Over Items                    | Merge                         |                                                                                             |
| Sales_email            | HTTP Request                 | Retrieves Sales email from Anymailfinder API        | Loop Over Items                    | Merge                         |                                                                                             |
| Marketing_email        | HTTP Request                 | Retrieves Marketing email from Anymailfinder API    | Loop Over Items                    | Merge                         |                                                                                             |
| Merge                  | Merge                       | Merges emails from CEO, Sales, and Marketing nodes  | Ceo_email, Sales_email, Marketing_email | If                           |                                                                                             |
| If                     | If                          | Filters valid email results (non-404 and non-empty) | Merge                             | Split Out, Loop Over Items     |                                                                                             |
| Split Out              | Split Out                   | Extracts email, full name, and LinkedIn URL         | If                               | Remove_duplicates1             |                                                                                             |
| Remove_duplicates1     | Remove Duplicates            | Deduplicates leads by email                          | Split Out                        | Extract_lead_name              | ### Retrieve lead name and remove duplicate emails found                                   |
| Extract_lead_name      | Code                        | Parses full name into first and last names          | Remove_duplicates1               | Add_lead_info                 |                                                                                             |
| Add_lead_info          | Google Sheets               | Adds enriched lead info to Google Sheets             | Extract_lead_name               | Generate custom script         | ### Add lead to email campaign with all infos                                              |
| Generate custom script | OpenAI (Langchain)          | Generates personalized cold email body with GPT-4o  | Add_lead_info                   | Generate custom title          | ### Generate custom script (GPT API)                                                       |
| Generate custom title  | OpenAI (Langchain)          | Generates personalized cold email subject line      | Generate custom script          | Send Draft                    | ### Generate custom title (GPT API)                                                        |
| Send Draft             | Gmail                       | Creates Gmail draft emails for review or sending    | Generate custom title           | Wait                         |                                                                                             |
| Wait                   | Wait                        | Wait node used before looping next batch            | Send Draft                     | Loop Over Items               |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 08:00 AM.

2. **Create HTTP Request Node (Linkedin_jobs_scrapper)**  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/curious_coder~linkedin-jobs-scraper/run-sync-get-dataset-items?token=YOUR_TOKEN`  
   - Body (JSON): Include URLs of LinkedIn job search pages filtered for marketing manager roles in Belgium, count=100, scrapeCompany=true.  
   - Authentication: HTTP Query Authentication with your Apify token.

3. **Add Remove Duplicates Node**  
   - Compare field: `companyName`  
   - Connect from Linkedin_jobs_scrapper.

4. **Add Filter Node (Filter_staffing)**  
   - Condition: Title contains "marketing manager" or "digital marketing manager" (case-insensitive), location includes Belgium/Wallonia, seniority level in specified set, exclude staffing companies by keywords in companyDescription.  
   - Connect from Remove Duplicates.

5. **Add Set Node (Save_usefull_infos)**  
   - Map necessary fields (JobUrl, companyWebsite, companySize, companyName, companyDescription, etc.) from raw JSON to named fields.  
   - Connect from Filter_staffing.

6. **Add Filter Node (Filter_domain_and_size)**  
   - Conditions: companySize not empty, companyWebsite exists and does not include "bit.ly", companySize < 100.  
   - Connect from Save_usefull_infos.

7. **Add Split in Batches Node (Loop Over Items)**  
   - Processes each company/job individually for enrichment.  
   - Connect from Filter_domain_and_size.

8. **Add Three HTTP Request Nodes (Ceo_email, Sales_email, Marketing_email)**  
   - Method: POST  
   - URL: `https://api.anymailfinder.com/v5.0/search/decision-maker.json`  
   - Body JSON: Use domain, company_name, and decision_maker_category set to "ceo", "sales", or "marketing" correspondingly.  
   - Authentication: HTTP Header Authentication with your Anymailfinder API key.  
   - Each configured to continue on error.  
   - Connect all three from Loop Over Items.

9. **Add Merge Node**  
   - Number Inputs: 3  
   - Connect from Ceo_email, Sales_email, Marketing_email.

10. **Add If Node**  
    - Condition: `$json.error.status != 404` AND email field is not empty.  
    - Connect from Merge.

11. **Add Split Out Node**  
    - Fields to split: `result.email`, `result.personFullName`, `result.personLinkedinUrl`.  
    - Connect from If (true branch).

12. **Add Remove Duplicates Node (Remove_duplicates1)**  
    - Compare field: `result.email`.  
    - Connect from Split Out.

13. **Add Code Node (Extract_lead_name)**  
    - JavaScript to parse full name into firstName and lastName with capitalization.  
    - Connect from Remove_duplicates1.

14. **Add Google Sheets Node (Add_lead_info)**  
    - Operation: Append  
    - Map fields: email, job url, fullname, job title, first name, companyName, job country, company size, company domain, email verified, company industry, company linkedin, company description, linkedin profile url.  
    - Connect from Extract_lead_name.  
    - Use Google Sheets OAuth2 credentials and specify target spreadsheet and sheet.

15. **Add OpenAI Node (Generate custom script)**  
    - Model: GPT-4o  
    - System and user prompts as per detailed workflow instructions for personalized cold email generation.  
    - Connect from Add_lead_info.  
    - Use OpenAI API credentials.

16. **Add OpenAI Node (Generate custom title)**  
    - Model: GPT-4o  
    - Prompt to generate a short, personalized subject line including first name.  
    - Connect from Generate custom script.  
    - Use OpenAI API credentials.

17. **Add Gmail Node (Send Draft)**  
    - Resource: Draft  
    - Subject: Output from Generate custom title node.  
    - Message: Output from Generate custom script node.  
    - CC: Lead email.  
    - Connect from Generate custom title.  
    - Use Gmail OAuth2 credentials.

18. **Add Wait Node**  
    - Use to pace batch processing if needed.  
    - Connect from Send Draft.  
    - Connect back to Loop Over Items for batch continuation.

19. **Add Sticky Notes** for documentation and clarity as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow scrapes LinkedIn jobs daily at 8 AM targeting Belgian marketing manager roles.       | Workflow overview and scheduling details.                                                                              |
| Avoids staffing and HR agencies in targeting to focus on direct companies.                    | Filtering logic in Filter_staffing node.                                                                                |
| Uses Anymailfinder API for decision-maker email enrichment (CEO, sales, marketing).           | Email enrichment API; requires valid subscription and API key.                                                         |
| GPT-4o used to generate personalized email bodies and icebreaker subject lines.               | OpenAI API integration; prompts carefully crafted for tone and personalization.                                         |
| Emails are drafted in Gmail for manual review or sending; not auto-sent.                      | Gmail OAuth2 integration; drafts created with CC to lead email for traceability.                                       |
| Google Sheets used for lead record keeping and campaign management.                           | Google Sheets OAuth2 credentials required; schema matches lead enrichment fields.                                       |
| Extensive use of expressions and JSON data mapping to maintain data integrity and personalization. | Requires understanding of n8n expressions and node data passing.                                                       |
| Sticky notes provide helpful context on each major block and node groupings.                  | See nodes named Sticky Note, Sticky Note1, etc., for workflow documentation embedded in n8n canvas.                     |
| Potential edge cases include API rate limits, network errors, missing data, and authentication failures. | Proper credential management and error handling (continue on error settings) help maintain workflow stability.          |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.