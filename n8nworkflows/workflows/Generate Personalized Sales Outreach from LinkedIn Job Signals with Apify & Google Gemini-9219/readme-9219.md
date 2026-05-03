Generate Personalized Sales Outreach from LinkedIn Job Signals with Apify & Google Gemini

https://n8nworkflows.xyz/workflows/generate-personalized-sales-outreach-from-linkedin-job-signals-with-apify---google-gemini-9219


# Generate Personalized Sales Outreach from LinkedIn Job Signals with Apify & Google Gemini

### 1. Workflow Overview

This n8n workflow automates the generation of personalized sales outreach emails targeting companies actively hiring tech roles on LinkedIn. It integrates data scraping, enrichment, filtering, and AI-generated email composition to streamline lead generation and outreach for sales teams or recruiters.

The workflow is logically divided into these blocks:

- **1.1 Input Trigger:** Manual start to initiate the workflow.
- **1.2 LinkedIn Job Scraping & Data Collection:** Uses Apify actor to scrape LinkedIn job postings and retrieves dataset items.
- **1.3 Data Filtering and Preparation:** Filters companies by size, removes duplicates and HR-related industries, prepares and sanitizes company details.
- **1.4 Domain Verification & Limiting:** Checks domain presence and limits the number of companies for focused processing.
- **1.5 Apollo.io Data Enrichment:** Searches Apollo.io for targeted decision-makers, sanitizes person data, and finds verified emails.
- **1.6 Data Merging:** Combines company, person, and email data into complete lead profiles.
- **1.7 Lead Storage:** Saves the enriched leads to Google Sheets.
- **1.8 Lead Fetching & Validation:** Reads leads back from the sheet and filters those needing email generation.
- **1.9 AI-based Email Generation:** Uses Google Gemini AI model via LangChain nodes to generate personalized cold emails.
- **1.10 Update Sheet with AI Output:** Parses AI output and updates the Google Sheet rows with generated email subject and body.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:** Manually triggers the entire lead generation and outreach process.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`
- **Node Details:**  
  - Type: Manual Trigger node  
  - Configuration: Default manual trigger, no parameters  
  - Inputs: None  
  - Outputs: Initiates the workflow chain to LinkedIn scraper  
  - Edge cases: User must manually start; no automatic scheduling implemented

---

#### 2.2 LinkedIn Job Scraping & Data Collection

- **Overview:** Scrapes LinkedIn job postings using Apify and retrieves the scraped dataset.
- **Nodes Involved:**  
  - `Run the LinkedIn Job Scraper`  
  - `Get dataset Items`
- **Node Details:**

  - **Run the LinkedIn Job Scraper**  
    - Type: Apify node  
    - Configuration: Runs Apify actor `Linkedin Jobs Scraper - PPR` with input to scrape 100 jobs for "ML Engineer" within 25 miles of geoId 103644278 (US region)  
    - Inputs: Trigger from manual node  
    - Outputs: Runs actor, outputs dataset ID for next node  
    - Edge cases: Apify actor errors, quota limits, timeout

  - **Get dataset Items**  
    - Type: Apify node  
    - Configuration: Retrieves up to 110 items from dataset returned by previous node  
    - Inputs: Output from scraper node (dataset ID)  
    - Outputs: Job and company data records  
    - Edge cases: Empty datasets, API limits

---

#### 2.3 Data Filtering and Preparation

- **Overview:** Filters companies smaller than 250 employees, removes duplicates and HR-related industries, prepares company data with clean domains.
- **Nodes Involved:**  
  - `Checks Company Size < 250`  
  - `Remove Duplicates`  
  - `Removes HR Related Industry`  
  - `Prepare Final Company Details`
- **Node Details:**

  - **Checks Company Size < 250**  
    - Type: If node  
    - Configuration: Passes only companies with `companyEmployeesCount` < 250  
    - Inputs: Dataset items  
    - Outputs: Filtered companies  
    - Edge cases: Missing or invalid employee count data

  - **Remove Duplicates**  
    - Type: Remove Duplicates node  
    - Configuration: Deduplicates on `companyName` field  
    - Inputs: Filtered companies  
    - Outputs: Unique company records  
    - Edge cases: Case sensitivity, missing companyName

  - **Removes HR Related Industry**  
    - Type: If node  
    - Configuration: Filters out companies whose `industries` contain “Human Resources” or “Staffing and Recruiting”  
    - Inputs: Deduplicated companies  
    - Outputs: Non-HR companies  
    - Edge cases: Industry field format variance

  - **Prepare Final Company Details**  
    - Type: Code node  
    - Configuration: Extracts and wraps company data, cleans website URL to get domain, normalizes fields, outputs wrapped in `company` object  
    - Inputs: Filtered companies  
    - Outputs: Cleaned company objects with parsed domain  
    - Edge cases: Malformed URLs, missing fields, URL parsing errors handled with fallback

---

#### 2.4 Domain Verification & Limiting

- **Overview:** Ensures companies have valid domains and limits the number of companies processed to 5.
- **Nodes Involved:**  
  - `Checks Domain Existence`  
  - `Limit Companies Search`
- **Node Details:**

  - **Checks Domain Existence**  
    - Type: If node  
    - Configuration: Checks existence of `company.domain` field  
    - Inputs: Prepared company details  
    - Outputs: Companies with valid domains only  
    - Edge cases: Empty or null domains

  - **Limit Companies Search**  
    - Type: Limit node  
    - Configuration: Limits max items to 5 for downstream processing  
    - Inputs: Companies with domains  
    - Outputs: Up to 5 companies  
    - Edge cases: Less than 5 companies available

---

#### 2.5 Apollo.io Data Enrichment

- **Overview:** Uses Apollo.io API to find key personnel, sanitize person data, and find verified emails.
- **Nodes Involved:**  
  - `Apollo Get Targeted Personnel`  
  - `Sanitising Person Details`  
  - `Apollo Email Finder`  
  - `Sanitising Email Details`
- **Node Details:**

  - **Apollo Get Targeted Personnel**  
    - Type: HTTP Request  
    - Configuration: POST to Apollo API people search endpoint with domain filter, seniorities, job titles, pagination (2 per company)  
    - Inputs: Company domain from limit node  
    - Outputs: List of people for each company  
    - Credentials: Requires Apollo.io API key configured as HTTP Header Auth  
    - Edge cases: API response errors, rate limits, empty results

  - **Sanitising Person Details**  
    - Type: Code node  
    - Configuration: Flattens and deduplicates person data by unique key (person.id + companyName), extracts relevant person and company fields, normalizes domain extraction  
    - Inputs: Apollo personnel data  
    - Outputs: Clean person records wrapped in `person` object  
    - Edge cases: Missing person or company fields, URL parsing errors

  - **Apollo Email Finder**  
    - Type: HTTP Request  
    - Configuration: POST to Apollo API match endpoint to find verified emails for each person by personId  
    - Inputs: Sanitized person data  
    - Outputs: Email info for each person  
    - Credentials: Apollo.io API key as HTTP Header Auth  
    - Edge cases: Email not found, API errors

  - **Sanitising Email Details**  
    - Type: Code node  
    - Configuration: Extracts and wraps email id and email address into simplified json for merging  
    - Inputs: Apollo email finder responses  
    - Outputs: Clean email objects  
    - Edge cases: Missing email data

---

#### 2.6 Data Merging

- **Overview:** Combines company, person, and email data into unified lead profiles.
- **Nodes Involved:**  
  - `Merge Data`  
  - `Structuring Complete Details of Person`
- **Node Details:**

  - **Merge Data**  
    - Type: Merge node  
    - Configuration: Merges three input streams (company, person, email) using “Merge by index” (default)  
    - Inputs:  
      - Companies (limited)  
      - Sanitized person data  
      - Sanitized email data  
    - Outputs: Combined raw data for further merging logic  
    - Edge cases: Mismatched array lengths or missing data

  - **Structuring Complete Details of Person**  
    - Type: Code node  
    - Configuration:  
      - Builds lookup maps for email by personId and company details by companyName  
      - Merges person data with their email and associated company info  
      - Outputs complete lead records with all info fields (email, companyDescription, jobTitle, employeeCount) flattened  
    - Inputs: Merged data from previous node  
    - Outputs: Complete lead profiles ready for storage  
    - Edge cases: Missing keys, incomplete merges

---

#### 2.7 Lead Storage

- **Overview:** Saves complete lead profiles as rows in a Google Sheet.
- **Nodes Involved:**  
  - `Adding Leads to Sheets`
- **Node Details:**

  - **Adding Leads to Sheets**  
    - Type: Google Sheets node  
    - Configuration: Appends rows with mapped lead fields (e.g., Email, Domain, Company, Location, Job Title, LinkedIn URLs, Employee Count, Company Description) into a master spreadsheet  
    - Inputs: Complete lead profiles  
    - Outputs: Confirmation of append operation  
    - Credentials: Google Sheets OAuth2 credentials configured  
    - Edge cases: API quota, write errors, schema mismatch

---

#### 2.8 Lead Fetching & Validation

- **Overview:** Reads leads from the spreadsheet and filters those with valid emails but missing generated email content.
- **Nodes Involved:**  
  - `Fetching Leads Data`  
  - `Validates Email Id and Email Content`  
  - `Filter for Batching`
- **Node Details:**

  - **Fetching Leads Data**  
    - Type: Google Sheets node  
    - Configuration: Reads all rows from the master sheet  
    - Inputs: None (triggered from prior node)  
    - Outputs: Raw lead data  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Read errors, empty sheet

  - **Validates Email Id and Email Content**  
    - Type: If node  
    - Configuration: Passes leads only if Email contains “@” and both Subject and Email Body fields are empty (indicating email not yet generated)  
    - Inputs: Lead data  
    - Outputs: Leads needing email generation  
    - Edge cases: Malformed emails, incomplete rows

  - **Filter for Batching**  
    - Type: Filter node  
    - Configuration: Filters leads where `row_number` > 83 (example batching or testing filter)  
    - Inputs: Validated leads  
    - Outputs: Batched leads to process by AI  
    - Edge cases: Row number missing or non-numeric

---

#### 2.9 AI-based Email Generation

- **Overview:** Uses Google Gemini AI model to generate a personalized cold email subject and body for each lead.
- **Nodes Involved:**  
  - `Lead Email Generator`  
  - `Google Gemini Chat Model`  
  - `Structured Output Parser`
- **Node Details:**

  - **Lead Email Generator**  
    - Type: LangChain Chain LLM node  
    - Configuration: Uses a detailed prompt instructing the AI to write a personalized, professional cold email based on lead data (Company, Job Title, Occupation), with constraints on length and tone  
    - Inputs: Batched lead data  
    - Outputs: Raw AI-generated text including subject and email body  
    - Edge cases: API limits, prompt failures, rate limiting

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model node  
    - Configuration: Executes the AI model call as requested by the Chain LLM node  
    - Inputs: Prompt and lead data  
    - Outputs: Raw AI response text  
    - Credentials: Google Gemini API credentials required  
    - Edge cases: Authentication errors, request timeouts

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Configuration: Parses AI text output into JSON with two fields: `subject` and `email_body`  
    - Inputs: Raw AI text  
    - Outputs: Parsed JSON with email content fields  
    - Edge cases: Parsing errors, malformed AI response

---

#### 2.10 Update Sheet with AI Output

- **Overview:** Updates the Google Sheet rows with the generated email subject and body for each lead.
- **Nodes Involved:**  
  - `Update Sheet with Email`
- **Node Details:**

  - **Update Sheet with Email**  
    - Type: Google Sheets node  
    - Configuration: Updates existing rows by matching the Email column, setting the Subject and Email Body columns with AI-generated content  
    - Inputs: Parsed AI output combined with lead email key for matching  
    - Outputs: Confirmation of update  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Row not found, concurrency conflicts

---

### 3. Summary Table

| Node Name                | Node Type                            | Functional Role                            | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                  |
|--------------------------|------------------------------------|-------------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                     | Initiates workflow                         | None                          | Run the LinkedIn Job Scraper | ▶️ Start Workflow: Manually triggers the entire lead generation process.                     |
| Run the LinkedIn Job Scraper | Apify                             | Scrape LinkedIn job postings               | When clicking ‘Execute workflow’ | Get dataset Items            | Run the LinkedIn Job Scraper from Apify: Scrape LinkedIn Job Postings based on search URL.    |
| Get dataset Items          | Apify                             | Retrieves scraped data                      | Run the LinkedIn Job Scraper  | Checks Company Size < 250    | Get dataset Items: Collect scraped job posting and company data.                              |
| Checks Company Size < 250  | If                                | Filters companies by employee count < 250 | Get dataset Items             | Remove Duplicates            | Checks Company Size < 250: Filter by Company Size.                                           |
| Remove Duplicates          | Remove Duplicates                  | Removes duplicate company entries          | Checks Company Size < 250     | Removes HR Related Industry  | Rmeove Duplicate Entries                                                                     |
| Removes HR Related Industry| If                                | Excludes HR and staffing related companies | Remove Duplicates             | Prepare Final Company Details| Rmeove HR Related Companies                                                                  |
| Prepare Final Company Details | Code                             | Cleans and structures company info         | Removes HR Related Industry   | Checks Domain Existence      | Prepare Final Company Details                                                                |
| Checks Domain Existence    | If                                | Filters companies with valid domain         | Prepare Final Company Details | Limit Companies Search       | Check domain Existence                                                                       |
| Limit Companies Search     | Limit                             | Limits to 5 companies for further processing| Checks Domain Existence       | Merge Data                  | Limit Company Search                                                                         |
| Apollo Get Targeted Personnel | HTTP Request                    | Apollo.io API call to find key personnel   | Limit Companies Search        | Sanitising Person Details    | Apollo Get Targeted Personnel: Searches Apollo.io for decision-makers.                       |
| Sanitising Person Details  | Code                              | Deduplicates and cleans person data        | Apollo Get Targeted Personnel | Apollo Email Finder, Merge Data | Sanitising Person Details: Clean Person Data.                                                |
| Apollo Email Finder        | HTTP Request                      | Finds verified emails via Apollo.io         | Sanitising Person Details     | Sanitising Email Details     | Apollo Email Finder: Finds verified emails for contacts.                                    |
| Sanitising Email Details   | Code                              | Extracts and formats email details          | Apollo Email Finder           | Merge Data                  | Sanitising Email Details: Isolate Email Data.                                               |
| Merge Data                 | Merge                             | Combines company, person, and email data   | Limit Companies Search, Sanitising Person Details, Sanitising Email Details | Structuring Complete Details of Person | Merge Data: Combine Data Streams.                                                            |
| Structuring Complete Details of Person | Code                     | Merges all data into final lead profile    | Merge Data                   | Adding Leads to Sheets       | Structuring Complete Details of Person: Create Final Lead Profile.                          |
| Adding Leads to Sheets     | Google Sheets                    | Appends lead profiles to Google Sheet      | Structuring Complete Details of Person | Fetching Leads Data          | Save Leads to Google Sheets                                                                  |
| Fetching Leads Data        | Google Sheets                    | Reads leads from Google Sheet               | Adding Leads to Sheets         | Validates Email Id and Email Content | Fetching Leads Data: Read Leads for AI.                                                      |
| Validates Email Id and Email Content | If                      | Filters leads missing generated emails      | Fetching Leads Data           | Filter for Batching          | Validates Email Id and Email Content: Filters leads for valid email and missing content.     |
| Filter for Batching        | Filter                           | Batches leads based on row_number filter   | Validates Email Id and Email Content | Lead Email Generator         | Apply Final Filter: Applies final condition before AI model.                                |
| Lead Email Generator       | LangChain Chain LLM              | Generates personalized email subject/body  | Filter for Batching           | Update Sheet with Email      | Lead Email Generator: AI Prompt Engine.                                                     |
| Google Gemini Chat Model   | LangChain Google Gemini          | AI model call to generate email content    | Lead Email Generator (llmChatGoogleGemini) | Lead Email Generator (outputParser) | Google Gemini Chat Model: The AI model that writes emails.                                  |
| Structured Output Parser   | LangChain Structured Output Parser | Parses AI output into JSON structured data | Google Gemini Chat Model      | Lead Email Generator         | Structured Output Parser: Format AI Output.                                                |
| Update Sheet with Email    | Google Sheets                    | Updates sheet rows with AI-generated emails| Lead Email Generator          | None                        | Update Sheet with Email: Saves AI-generated email content back to Google Sheets.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger (default settings)  

2. **Add Apify Node to Scrape LinkedIn Jobs:**  
   - Name: `Run the LinkedIn Job Scraper`  
   - Type: Apify  
   - Actor ID: `hKByXkMQaC5Qt9UMN` (Linkedin Jobs Scraper - PPR)  
   - Memory: 4096 MB  
   - Input JSON:  
     ```json
     {
       "count": 100,
       "scrapeCompany": true,
       "urls": [
         "https://www.linkedin.com/jobs/search-results/?distance=25&f_TPR=r86400&geoId=103644278&keywords=ML%20Engineer&origin=SEMANTIC_SEARCH_HISTORY"
       ]
     }
     ```
   - Connect output of manual trigger to this node.

3. **Add Apify Node to Get Dataset Items:**  
   - Name: `Get dataset Items`  
   - Type: Apify  
   - Resource: Datasets  
   - Dataset ID: Use the output `defaultDatasetId` from previous node  
   - Limit: 110 items  
   - Connect output from scraper to this node.

4. **Add If Node to Filter Companies by Employee Count:**  
   - Name: `Checks Company Size < 250`  
   - Condition: Number `companyEmployeesCount` < 250  
   - Connect output from dataset node to this node.

5. **Add Remove Duplicates Node:**  
   - Name: `Remove Duplicates`  
   - Fields to compare: `companyName`  
   - Connect output from above If node.

6. **Add If Node to Remove HR-related Industries:**  
   - Name: `Removes HR Related Industry`  
   - Condition: `industries` does NOT contain "Human Resources" OR "Staffing and Recruiting"  
   - Connect output from Remove Duplicates node.

7. **Add Code Node to Prepare Final Company Details:**  
   - Name: `Prepare Final Company Details`  
   - Paste provided JavaScript code to clean and wrap company data, extract domain  
   - Connect output from above If node.

8. **Add If Node to Check Domain Existence:**  
   - Name: `Checks Domain Existence`  
   - Condition: `company.domain` exists (non-empty)  
   - Connect output from Prepare Final Company Details.

9. **Add Limit Node to Limit Companies:**  
   - Name: `Limit Companies Search`  
   - Max Items: 5  
   - Connect output from domain check node.

10. **Add HTTP Request Node for Apollo People Search:**  
    - Name: `Apollo Get Targeted Personnel`  
    - Method: POST  
    - URL: `https://api.apollo.io/api/v1/people/search`  
    - Authentication: HTTP Header Auth with Apollo.io API key credential  
    - Body (JSON):  
      ```json
      {
        "q_organization_domains_list": ["{{ $json.company.domain }}"],
        "person_seniorities": ["vp", "head", "director", "founder", "c-suite", "lead"],
        "person_titles": [
          "engineering",
          "technology",
          "product",
          "operations",
          "infrastructure",
          "devops",
          "data science",
          "machine learning",
          "cloud"
        ],
        "pagination": { "page": 1, "per_page": 2 }
      }
      ```
    - Connect output from Limit Companies Search node.

11. **Add Code Node to Sanitize Person Details:**  
    - Name: `Sanitising Person Details`  
    - Paste provided JS code to flatten and dedupe person data  
    - Connect output from Apollo personnel node.

12. **Add HTTP Request Node to Find Emails via Apollo:**  
    - Name: `Apollo Email Finder`  
    - Method: POST  
    - URL: `https://api.apollo.io/v1/people/match`  
    - Authentication: HTTP Header Auth with Apollo.io API key  
    - Body (JSON): `{ "id": "{{ $json.person.personId }}" }`  
    - Connect output from Sanitising Person Details node.

13. **Add Code Node to Sanitize Email Details:**  
    - Name: `Sanitising Email Details`  
    - Paste provided JS code to extract email info  
    - Connect output from Apollo Email Finder.

14. **Add Merge Node to Combine Company, Person, and Email Data:**  
    - Name: `Merge Data`  
    - Number of Inputs: 3  
    - Connect outputs:  
      - Input 1: Limit Companies Search output (companies)  
      - Input 2: Sanitising Person Details output (persons)  
      - Input 3: Sanitising Email Details output (emails)

15. **Add Code Node to Structure Complete Lead Details:**  
    - Name: `Structuring Complete Details of Person`  
    - Paste provided JS code to merge all data into final lead profile  
    - Connect output from Merge Data.

16. **Add Google Sheets Node to Append Leads:**  
    - Name: `Adding Leads to Sheets`  
    - Operation: Append  
    - Google Sheets OAuth2 credential  
    - Document ID: Spreadsheet ID for master leads sheet  
    - Sheet Name: `gid=0` or as appropriate  
    - Map fields as per node config (Email, Domain, Company, etc.)  
    - Connect output from Structuring Complete Details of Person.

17. **Add Google Sheets Node to Fetch Leads:**  
    - Name: `Fetching Leads Data`  
    - Operation: Read rows  
    - Use same credential and spreadsheet as above  
    - Connect output from Adding Leads to Sheets.

18. **Add If Node to Validate Email and Missing Content:**  
    - Name: `Validates Email Id and Email Content`  
    - Condition: Email contains “@”, AND Subject is empty, AND Email Body is empty  
    - Connect output from Fetching Leads Data.

19. **Add Filter Node for Batching:**  
    - Name: `Filter for Batching`  
    - Condition: `row_number` > 83 (adjust as needed)  
    - Connect output from validation node.

20. **Add LangChain Chain LLM Node to Generate Emails:**  
    - Name: `Lead Email Generator`  
    - Prompt: Use detailed prompt text provided to generate personalized cold email and subject  
    - Enable Output Parser  
    - Connect output from Filter for Batching.

21. **Add LangChain Google Gemini Chat Model Node:**  
    - Name: `Google Gemini Chat Model`  
    - Connect as AI language model node to Lead Email Generator.

22. **Add LangChain Structured Output Parser Node:**  
    - Name: `Structured Output Parser`  
    - JSON Schema Example:  
      ```json
      {
        "subject": "A short, catchy, question-based subject line",
        "email_body": "The full email body, 100-120 words, without any salutation or sign-off."
      }
      ```
    - Connect as AI output parser node to Lead Email Generator.

23. **Add Google Sheets Node to Update Sheet with Emails:**  
    - Name: `Update Sheet with Email`  
    - Operation: Update rows by matching Email column  
    - Columns to update: Subject, Email Body mapped from AI output  
    - Connect output from Lead Email Generator.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses Apify actor “Linkedin Jobs Scraper - PPR” for LinkedIn job scraping: https://console.apify.com/actors/hKByXkMQaC5Qt9UMN                                                                                                | Apify LinkedIn Job Scraper Actor                                                                |
| Apollo.io API requires HTTP Header Auth with API key for people search and email finder endpoints                                                                                                                               | Apollo.io API Documentation: https://apollo.io/api/docs                                       |
| Google Gemini AI model integration via LangChain nodes; credentials for Google Gemini API required                                                                                                                              | Google Gemini and LangChain integration docs                                                   |
| Prompt engineering carefully constrains email tone, length, and personalization for effective cold outreach                                                                                                                    | Prompt text included in Lead Email Generator node                                              |
| Google Sheets used as both input and output datastore; schema includes company, person, and outreach email fields                                                                                                               | Google Sheets API OAuth2 credentials required                                                  |
| For assistance with workflow customization or development, contact Intuz: Email getstarted@intuz.com, Website https://www.intuz.com/                                                                                              | Contact for custom automation solutions                                                        |

---

This document fully describes the workflow’s structure, logic, and configuration, enabling users or AI agents to understand, reproduce, or modify the sales outreach automation end-to-end.