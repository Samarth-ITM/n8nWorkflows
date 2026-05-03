Upwork Lead Generation: Extract Client Emails with LinkedIn Scraping and AI

https://n8nworkflows.xyz/workflows/upwork-lead-generation--extract-client-emails-with-linkedin-scraping-and-ai-4794


# Upwork Lead Generation: Extract Client Emails with LinkedIn Scraping and AI

### 1. Workflow Overview

This workflow automates lead generation by extracting client emails from Upwork job postings using LinkedIn scraping and AI. It is designed to periodically fetch new Upwork jobs, extract client or company names via AI, classify them as person or company, search corresponding LinkedIn profiles through Phantombuster APIs, scrape detailed LinkedIn data, find emails via Hunter.io, and finally store enriched leads in Google Sheets. The workflow is structured into these logical blocks:

- **1.1 Trigger & Data Collection:** Periodically fetch latest Upwork jobs using Apify API.  
- **1.2 AI Extraction & Classification:** Use OpenAI and Langchain agents to extract client/company name and classify as person or company.  
- **1.3 Conditional Routing:** Branch workflow based on person vs company classification.  
- **1.4 Company Path:** Search LinkedIn for company profiles, scrape company data, find company emails via Hunter.io.  
- **1.5 Person Path:** Search LinkedIn for individual profiles, scrape person data, find personal emails via Hunter.io.  
- **1.6 Storage:** Append all enriched data to Google Sheets for tracking and outreach.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Collection

- **Overview:**  
  This block triggers the workflow on a schedule and fetches the latest Upwork job postings via the Apify API.

- **Nodes Involved:**  
  - Run Every X Hours  
  - Fetch Latest Upwork Jobs (Apify)

- **Node Details:**

  - **Run Every X Hours**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow periodically (default hourly interval)  
    - Config: Runs every X hours as defined in the schedule parameter  
    - Inputs: None  
    - Outputs: Triggers next node  
    - Failures: None typical; workflow won’t run if n8n service is down  

  - **Fetch Latest Upwork Jobs (Apify)**  
    - Type: HTTP Request  
    - Role: Fetches newest Upwork jobs as JSON from Apify Upwork actor API  
    - Config: HTTP GET request with API token and task ID embedded in URL  
    - Inputs: Trigger from schedule  
    - Outputs: JSON array of job postings (title, description, client info, URL, etc.)  
    - Failures: API errors, rate limits, invalid token, malformed response  

#### 2.2 AI Extraction & Classification

- **Overview:**  
  Uses AI agents to extract the client's or company’s name from job details and classify the extracted name as a person or company.

- **Nodes Involved:**  
  - Extract Company or Person Name from Job (Langchain Agent)  
  - Name is Found? (IF)  
  - No Name Found - End Early (NoOp)  
  - Is It a Person or Company? (Langchain Agent)  
  - Person or Company? (IF)

- **Node Details:**

  - **Extract Company or Person Name from Job**  
    - Type: Langchain AI Agent (OpenAI Chat model)  
    - Role: Parses job title + description to extract client name or company name only  
    - Config: Prompt instructs to return only the name or "null" if none found  
    - Inputs: JSON from Upwork jobs  
    - Outputs: Single string field with extracted name or "null"  
    - Failures: GPT API errors, ambiguous or missing data, empty output  

  - **Name is Found?**  
    - Type: IF  
    - Role: Checks if extracted name is not "null"  
    - Config: Condition `output != "null"`  
    - Inputs: Output of AI extraction  
    - Outputs: If true, continue; else route to No Name Found node  
    - Failures: Expression errors if output missing  

  - **No Name Found - End Early**  
    - Type: No Operation (NoOp)  
    - Role: Terminates processing for jobs without extracted names  
    - Inputs: From IF false branch  
    - Outputs: None  
    - Failures: None  

  - **Is It a Person or Company?**  
    - Type: Langchain AI Agent  
    - Role: Classifies extracted name as "person" or "company" via prompt  
    - Config: Returns string "person" or "company" only  
    - Inputs: Extracted name string  
    - Outputs: Classification string  
    - Failures: API errors, ambiguous classification  

  - **Person or Company?**  
    - Type: IF  
    - Role: Routes flow based on classification: if "company" follow company path, else person path  
    - Config: Condition equals "company"  
    - Inputs: Classification output  
    - Outputs: Two branches: company or person  
    - Failures: Expression evaluation errors  

#### 2.3 Company Path

- **Overview:**  
  Searches LinkedIn for company profiles using Phantombuster, scrapes company info, finds company emails via Hunter.io.

- **Nodes Involved:**  
  - Search LinkedIn for Company (Phantombuster)  
  - Scrape LinkedIn Company Profile  
  - Find Company Email (Hunter.io)

- **Node Details:**

  - **Search LinkedIn for Company (Phantombuster)**  
    - Type: HTTP Request  
    - Role: Launches Phantombuster LinkedIn company search agent with company name and session cookie  
    - Config: POST request with JSON body providing search term, profile count, and LinkedIn session cookie; API key in headers  
    - Inputs: Company name string  
    - Outputs: LinkedIn company profile URLs and metadata  
    - Failures: Invalid API key, session expired, rate limiting, network failure  

  - **Scrape LinkedIn Company Profile**  
    - Type: HTTP Request  
    - Role: Launches Phantombuster agent to scrape detailed company profile data using spreadsheet URL from previous node  
    - Config: POST request with spreadsheet URL containing LinkedIn URLs, session cookie, and API key header  
    - Inputs: Company LinkedIn profile URL(s)  
    - Outputs: Company details: size, website, industry, location, description, specialties  
    - Failures: Session cookie invalid, data extraction errors, missing URLs, API errors  

  - **Find Company Email (Hunter.io)**  
    - Type: HTTP Request  
    - Role: Queries Hunter.io domain search API to find company-level emails by domain extracted from company website  
    - Config: GET request with domain parameter and API key  
    - Inputs: Company domain string  
    - Outputs: List of emails with type, confidence scores  
    - Failures: API quota exceeded, invalid domain, malformed request  

#### 2.4 Person Path

- **Overview:**  
  Searches LinkedIn for individual profiles via Phantombuster, scrapes person data, and finds personal emails through Hunter.io.

- **Nodes Involved:**  
  - Search LinkedIn for Person (Phantombuster)  
  - Scrape LinkedIn Person Profile  
  - Find Personal Email (Hunter.io)

- **Node Details:**

  - **Search LinkedIn for Person (Phantombuster)**  
    - Type: HTTP Request  
    - Role: Launches Phantombuster LinkedIn person search agent with person name and session cookie  
    - Config: POST request with JSON body (search name, profile count, session cookie), API key in header  
    - Inputs: Person name string  
    - Outputs: LinkedIn profile URLs and metadata  
    - Failures: API key invalid, session cookie expired, no profiles found  

  - **Scrape LinkedIn Person Profile**  
    - Type: HTTP Request  
    - Role: Launches Phantombuster agent to scrape detailed individual profile data using spreadsheet URL  
    - Config: POST request with spreadsheet URL, session cookie, API key in header  
    - Inputs: LinkedIn profile URL(s)  
    - Outputs: Person details including skills, summary, job titles, education, location  
    - Failures: Invalid session, incomplete data, API errors  

  - **Find Personal Email (Hunter.io)**  
    - Type: HTTP Request  
    - Role: Uses Hunter.io email finder endpoint to locate personal emails by full name and domain  
    - Config: GET request with full name and domain parameters, API key  
    - Inputs: Person full name and company domain  
    - Outputs: Email address and confidence score if available  
    - Failures: API limits, ambiguous results, no email found  

#### 2.5 Storage

- **Overview:**  
  Collects all enriched data and appends a new row in Google Sheets for tracking and outreach.

- **Nodes Involved:**  
  - Store Results in Google Sheet

- **Node Details:**

  - **Store Results in Google Sheet**  
    - Type: Google Sheets (Append Row)  
    - Role: Saves job title, description, client name, type (person/company), LinkedIn profile URL, scraped details, and emails in a Google Sheet  
    - Config: Document ID and sheet name provided via resource list or variable  
    - Inputs: Data from either company or person email finder nodes  
    - Outputs: Confirmation of row append  
    - Failures: Permission issues, wrong document ID, API quota exceeded  

---

### 3. Summary Table

| Node Name                          | Node Type                             | Functional Role                                  | Input Node(s)                       | Output Node(s)                                   | Sticky Note                                                             |
|-----------------------------------|-------------------------------------|-------------------------------------------------|-----------------------------------|-------------------------------------------------|-------------------------------------------------------------------------|
| Run Every X Hours                 | Schedule Trigger                    | Periodic trigger to start workflow              | None                              | Fetch Latest Upwork Jobs (Apify)                 | Trigger & Data Collection explanation                                  |
| Fetch Latest Upwork Jobs (Apify)  | HTTP Request                       | Fetch latest Upwork jobs via Apify API          | Run Every X Hours                 | Extract Company or Person Name from Job          | Trigger & Data Collection explanation                                  |
| Extract Company or Person Name from Job | Langchain AI Agent (OpenAI Chat)  | Extract client or company name from job          | Fetch Latest Upwork Jobs (Apify) | Name is Found?                                   | AI Name Extraction explanation                                        |
| Name is Found?                    | IF                                 | Check if name was extracted                       | Extract Company or Person Name from Job | Is It a Person or Company?, No Name Found - End Early | AI Name Extraction explanation                                        |
| No Name Found - End Early         | No Operation (NoOp)                | Stops processing if no name found                 | Name is Found?                   | None                                            | AI Name Extraction explanation                                        |
| Is It a Person or Company?        | Langchain AI Agent                 | Classify extracted name as person or company      | Name is Found?                   | Person or Company?                               | AI Classification explanation                                        |
| Person or Company?                | IF                                 | Route flow to person or company path              | Is It a Person or Company?       | Search LinkedIn for Company (Phantombuster), Search LinkedIn for Person (Phantombuster) | AI Classification explanation                                        |
| Search LinkedIn for Company (Phantombuster) | HTTP Request                       | Search LinkedIn for company profiles              | Person or Company? (company branch) | Scrape LinkedIn Company Profile                   | Company Path explanation                                             |
| Scrape LinkedIn Company Profile   | HTTP Request                       | Scrape detailed company LinkedIn data             | Search LinkedIn for Company       | Find Company Email (Hunter.io)                   | Company Path explanation                                             |
| Find Company Email (Hunter.io)    | HTTP Request                       | Find company emails using Hunter.io               | Scrape LinkedIn Company Profile   | Store Results in Google Sheet                     | Company Path explanation                                             |
| Search LinkedIn for Person (Phantombuster) | HTTP Request                       | Search LinkedIn for person profiles                | Person or Company? (person branch) | Scrape LinkedIn Person Profile                     | Person Path explanation                                              |
| Scrape LinkedIn Person Profile    | HTTP Request                       | Scrape detailed individual LinkedIn data          | Search LinkedIn for Person         | Find Personal Email (Hunter.io)                   | Person Path explanation                                              |
| Find Personal Email (Hunter.io)   | HTTP Request                       | Find personal emails using Hunter.io               | Scrape LinkedIn Person Profile    | Store Results in Google Sheet                     | Person Path explanation                                              |
| Store Results in Google Sheet     | Google Sheets (Append Row)         | Store enriched lead data                            | Find Company Email (Hunter.io), Find Personal Email (Hunter.io) | None                                            | Google Sheets storage explanation                                   |
| Sticky Note9                     | Sticky Note                       | Workflow assistance and overall explanation        | None                            | None                                            | General workflow instructions and support contacts                  |
| Sticky Note4                     | Sticky Note                       | High-level workflow description                     | None                            | None                                            | Detailed step-by-step block explanation                             |
| Sticky Note1                     | Sticky Note                       | Company path detailed explanation                    | None                            | None                                            | Company Path detailed description                                  |
| Sticky Note3                     | Sticky Note                       | Person path detailed explanation                     | None                            | None                                            | Person Path detailed description                                   |
| Sticky Note5                     | Sticky Note                       | Google Sheets storage explanation                    | None                            | None                                            | Storage block explanation                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Run Every X Hours`  
   - Type: Schedule Trigger  
   - Setup: Set interval to run every X hours (e.g., 1 hour or desired frequency).

2. **Create HTTP Request Node for Upwork Jobs**  
   - Name: `Fetch Latest Upwork Jobs (Apify)`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.apify.com/v2/actor-tasks/<TASK_ID>/run-sync-get-dataset-items?token=<API_TOKEN>` (replace placeholders)  
   - Connect input from: `Run Every X Hours`

3. **Create Langchain AI Agent Node for Name Extraction**  
   - Name: `Extract Company or Person Name from Job`  
   - Type: Langchain AI Agent (OpenAI Chat)  
   - Model: `gpt-4o-mini` or other available GPT-4 variant  
   - Prompt: Input text as "Title: {{ $json.title }}\nDescription: {{ $json.description }}"  
   - System message: "Extract the client's name or company name if it is mentioned. Return only the name or 'null' if not found."  
   - Connect input from: `Fetch Latest Upwork Jobs (Apify)`

4. **Create IF Node to Check Name Presence**  
   - Name: `Name is Found?`  
   - Type: IF  
   - Condition: Check if extracted output `{{ $json.output }}` is not equal to `"null"`  
   - Connect input from: `Extract Company or Person Name from Job`  
   - True branch: Continue workflow  
   - False branch: Connect to `No Name Found - End Early`

5. **Create No Operation Node**  
   - Name: `No Name Found - End Early`  
   - Type: NoOp  
   - Connect input from: False branch of `Name is Found?`

6. **Create Langchain AI Agent Node for Person/Company Classification**  
   - Name: `Is It a Person or Company?`  
   - Type: Langchain AI Agent  
   - Input text: `{{ $json.output }}` (extracted name)  
   - System message: "Extract the client's name or company name. Return only 'person' if looks like a person's name else return 'company'."  
   - Connect input from: True branch of `Name is Found?`

7. **Create IF Node to Route by Classification**  
   - Name: `Person or Company?`  
   - Type: IF  
   - Condition: Output equals `"company"`  
   - Connect input from: `Is It a Person or Company?`  
   - True branch: Company path nodes  
   - False branch: Person path nodes

8. **Company Path Setup:**  
   - **HTTP Request Node: `Search LinkedIn for Company (Phantombuster)`**  
     - Method: POST  
     - URL: `https://api.phantombuster.com/api/v2/agents/<AGENT_ID>/launch` (replace with your agent ID)  
     - JSON Body:  
       ```json
       {
         "argument": {
           "search": "{{ $json.output }}",
           "numberOfProfiles": 5,
           "sessionCookie": "<LINKEDIN_SESSION_COOKIE>"
         },
         "saveArguments": true
       }
       ```
     - Headers: `X-Phantombuster-Key-1: <YOUR_API_KEY>`  
     - Connect input from: True branch of `Person or Company?`

   - **HTTP Request Node: `Scrape LinkedIn Company Profile`**  
     - Method: POST  
     - URL: same Phantombuster agent launch URL for scraping  
     - JSON Body:  
       ```json
       {
         "argument": {
           "spreadsheetUrl": "{{ $json.profileUrl }}",
           "sessionCookie": "<LINKEDIN_SESSION_COOKIE>"
         },
         "saveArguments": true
       }
       ```
     - Headers: `X-Phantombuster-Key-1: <YOUR_API_KEY>`  
     - Connect input from: `Search LinkedIn for Company (Phantombuster)`

   - **HTTP Request Node: `Find Company Email (Hunter.io)`**  
     - Method: GET  
     - URL Template:  
       `https://api.hunter.io/v2/domain-search?domain={{ $json.website }}&api_key=<YOUR_API_KEY>`  
     - Connect input from: `Scrape LinkedIn Company Profile`

9. **Person Path Setup:**  
   - **HTTP Request Node: `Search LinkedIn for Person (Phantombuster)`**  
     - Same setup as company search but with person name and fewer profiles (3)  
     - JSON Body:  
       ```json
       {
         "argument": {
           "search": "{{ $json.output }}",
           "numberOfProfiles": 3,
           "sessionCookie": "<LINKEDIN_SESSION_COOKIE>"
         },
         "saveArguments": true
       }
       ```
     - Headers: `X-Phantombuster-Key-1: <YOUR_API_KEY>`  
     - Connect input from: False branch of `Person or Company?`

   - **HTTP Request Node: `Scrape LinkedIn Person Profile`**  
     - Same as company profile scraping but fetching personal details  
     - Connect input from: `Search LinkedIn for Person (Phantombuster)`

   - **HTTP Request Node: `Find Personal Email (Hunter.io)`**  
     - Method: GET  
     - URL Template:  
       `https://api.hunter.io/v2/email-finder?full_name={{ $json.fullName }}&domain={{ $json.domain }}&api_key=<YOUR_API_KEY>`  
     - Connect input from: `Scrape LinkedIn Person Profile`

10. **Google Sheets Node to Store Results**  
    - Name: `Store Results in Google Sheet`  
    - Type: Google Sheets (Append Row)  
    - Parameters:  
      - Operation: Append  
      - Document ID and Sheet Name: fill from your Google Sheets credentials  
    - Connect input from: Both `Find Company Email (Hunter.io)` and `Find Personal Email (Hunter.io)` nodes  

11. **Credential Setup:**  
    - Configure OpenAI API credentials with valid API key  
    - Configure Phantombuster API key and LinkedIn session cookie securely  
    - Configure Hunter.io API key  
    - Configure Google Sheets OAuth2 credentials  

12. **Testing & Validation:**  
    - Test with dummy Upwork job data to validate AI extraction and routing  
    - Monitor rate limits and API quotas  
    - Add error handling or retries as needed (not included by default)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| For any questions or support, contact Yaron at Yaron@nofluff.online                                                                                                  | Support contact                                                                                                      |
| Explore more automation tips and tutorials on Yaron’s YouTube channel: https://www.youtube.com/@YaronBeen/videos                                                     | Video tutorials                                                                                                      |
| LinkedIn profile of Yaron Been for professional insights: https://www.linkedin.com/in/yaronbeen/                                                                      | Professional networking                                                                                                |
| This workflow uses Apify Upwork actor API, Phantombuster LinkedIn scraping agents, Hunter.io email finder, OpenAI GPT models, and Google Sheets integration            | Key integrations overview                                                                                            |
| Secure all API keys and session cookies using environment variables or n8n credentials to prevent exposure or unauthorized use                                        | Security best practice                                                                                               |
| Be mindful of API rate limits and quotas for Phantombuster, Hunter.io, and OpenAI to avoid service interruptions                                                     | Operational caution                                                                                                  |
| Consider adding error handling, retry logic, and rate-limiting nodes in n8n for production-ready robustness                                                          | Reliability improvements                                                                                             |
| Use n8n execution logs to debug and trace problems step-by-step                                                                                                      | Debugging tip                                                                                                        |

---

**Disclaimer:** This document represents an automated n8n workflow for legal, publicly available data processing, respecting all content policies and privacy standards.