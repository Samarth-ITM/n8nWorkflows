Enrich LinkedIn Profiles in NocoDB CRM with Apify Scraper

https://n8nworkflows.xyz/workflows/enrich-linkedin-profiles-in-nocodb-crm-with-apify-scraper-5258


# Enrich LinkedIn Profiles in NocoDB CRM with Apify Scraper

# Reference Document for n8n Workflow  
**Title:** Enrich LinkedIn Profiles in NocoDB CRM with Apify Scraper

---

### 1. Workflow Overview

This workflow automates the enrichment of LinkedIn profile data for guests stored in a NocoDB CRM database by leveraging the Apify LinkedIn profile scraper. It targets use cases where basic LinkedIn URLs are available in the CRM, but detailed profile information is missing. The enriched data enhances CRM records with comprehensive LinkedIn insights such as names, headlines, emails, bios, skills, experiences, and more.

The workflow includes the following logical blocks:

- **1.1 Input Reception:** Triggering the workflow manually or via schedule, then retrieving guest records with LinkedIn URLs needing enrichment.
- **1.2 LinkedIn Profile Scraping:** Submitting profile URLs to Apify scraper, waiting for scraper completion, and checking run status.
- **1.3 Data Retrieval and Transformation:** Fetching detailed profile data from Apify, transforming it into the CRM schema.
- **1.4 CRM Update on Success:** Updating NocoDB guest records with enriched LinkedIn data.
- **1.5 Handling Invalid URLs (404):** Detecting inaccessible profiles and clearing broken LinkedIn URLs in CRM.
- **1.6 Error Handling:** Capturing scrape errors or timeouts, and updating CRM records with error statuses.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception  
**Overview:**  
The workflow starts either manually or on a schedule to fetch CRM records with LinkedIn URLs that lack full profile data.

**Nodes Involved:**  
- Manual Trigger  
- Schedule Trigger  
- Get Guests with LinkedIn

**Node Details:**  

- **Manual Trigger**  
  - Type: Trigger (manual)  
  - Role: Entry point for manual workflow execution  
  - Configuration: Default, no parameters  
  - Connections: Outputs to "Get Guests with LinkedIn"  
  - Failures: None expected; manual invocation only

- **Schedule Trigger**  
  - Type: Trigger (scheduled)  
  - Role: Periodic automated start for the workflow  
  - Configuration: Default interval (empty array indicates manual or customized schedule)  
  - Connections: Outputs to "Get Guests with LinkedIn"  
  - Failures: None expected; ensure schedule is properly set in production

- **Get Guests with LinkedIn**  
  - Type: NocoDB node (Get All)  
  - Role: Retrieves guest records that have a LinkedIn URL but missing headline (indicating unenriched profiles)  
  - Configuration:  
    - Table: CRM table ID (placeholder "NOCODB_TABLEID")  
    - Filter: Where LinkedIn field is not null AND linkedin_headline is null  
    - Limit: 15 records per run  
    - Authentication: NocoDB API token  
  - Connections: Outputs to "Run Apify LinkedIn Scraper"  
  - Failures: Possible auth errors or API failures; handle rate limits and token expiry  

---

#### 1.2 LinkedIn Profile Scraping  
**Overview:**  
Sends LinkedIn URLs to Apify scraper, waits for run completion, then evaluates the success of the scraping.

**Nodes Involved:**  
- Run Apify LinkedIn Scraper  
- Wait for Completion  
- Check Run Status

**Node Details:**  

- **Run Apify LinkedIn Scraper**  
  - Type: HTTP Request (POST)  
  - Role: Initiates scraping by sending LinkedIn URL to Apify LinkedIn profile scraper actor  
  - Configuration:  
    - URL: Apify API endpoint for LinkedIn scraper runs  
    - Body: JSON with profileUrls array containing the LinkedIn URL from the CRM record (`{{$json.LinkedIn}}`)  
    - Authentication: HTTP Query Auth with Apify token  
  - Connections: Outputs to "Wait for Completion"  
  - Failures: Auth errors, network timeouts, malformed URL input

- **Wait for Completion**  
  - Type: HTTP Request (GET)  
  - Role: Polls the Apify run endpoint to wait up to 240 seconds for the scraping task to finish  
  - Configuration:  
    - URL: Constructed using run ID from previous node’s response  
    - Query parameter: waitForFinish=240 (timeout 4 minutes)  
    - Timeout: 300000 ms (5 minutes)  
    - Authentication: Apify token  
  - Connections: Outputs to "Check Run Status"  
  - Failures: Timeout if scraper is slow; network errors; unexpected status codes

- **Check Run Status**  
  - Type: If node  
  - Role: Checks if the scraper run status equals "SUCCEEDED" to branch success vs failure flows  
  - Configuration:  
    - Condition: `$json.data.status === "SUCCEEDED"`  
  - Connections:  
    - True branch: To "Get Scraper Results"  
    - False branch: To "Handle Scraper Error"  
  - Failures: Expression errors if response data is malformed or missing

---

#### 1.3 Data Retrieval and Transformation  
**Overview:**  
Retrieves detailed LinkedIn profile data from Apify dataset, validates it, and transforms it to match the NocoDB CRM schema.

**Nodes Involved:**  
- Get Scraper Results  
- Transform Data

**Node Details:**  

- **Get Scraper Results**  
  - Type: Code node (JavaScript)  
  - Role: Fetches dataset items from Apify using the dataset ID, handles empty or invalid data, returns wrapped LinkedIn data  
  - Configuration:  
    - Runs once per item  
    - Uses internal HTTP request helper with Apify token (must be replaced with actual token)  
    - Error handling: Throws on empty dataset or invalid profile data; continues error output on HTTP 404 dataset not found  
  - Connections:  
    - Success: To "Transform Data"  
    - Failure: To "Clear Broken LinkedIn URL" (handles 404 profiles)  
  - Failures: HTTP errors, invalid JSON, missing fields

- **Transform Data**  
  - Type: Code node (JavaScript)  
  - Role: Extracts first LinkedIn profile from dataset array and maps fields to CRM schema with timestamps  
  - Configuration:  
    - Runs once per item  
    - Reads guest ID from original NocoDB data  
    - Constructs object with rich LinkedIn fields: full name, headline, email, bio, profile pic URLs, current role/company, country, skills, experiences (stringified arrays), websites, publications, scrape status and timestamps  
  - Connections: Outputs to "Update Guest Success"  
  - Failures: Missing guest ID, empty LinkedIn data, JSON serialization errors

---

#### 1.4 CRM Update on Success  
**Overview:**  
Updates the guest record in NocoDB with the enriched LinkedIn profile data.

**Nodes Involved:**  
- Update Guest Success

**Node Details:**  

- **Update Guest Success**  
  - Type: NocoDB node (Update)  
  - Role: Updates guest record fields with transformed LinkedIn data  
  - Configuration:  
    - Table: CRM table ID  
    - ID: From transformed data’s `Id` field  
    - Data to send: Automatically maps all input fields  
    - Authentication: NocoDB API token  
  - Connections: None (terminal node on success flow)  
  - Failures: Auth errors, invalid record ID, API failures

---

#### 1.5 Handling Invalid URLs (404)  
**Overview:**  
When a profile dataset is not found (404), clears the broken LinkedIn URL from the guest record and marks status accordingly.

**Nodes Involved:**  
- Clear Broken LinkedIn URL  
- Update Guest - Clear URL

**Node Details:**  

- **Clear Broken LinkedIn URL**  
  - Type: Code node (JavaScript)  
  - Role: Prepares data to clear the LinkedIn URL field and set error status "invalid_url" with reason  
  - Configuration:  
    - Runs once per item  
    - Reads guest ID and LinkedIn URL from original data  
    - Returns object clearing LinkedIn field, updating scrape status, error reason, and timestamp  
  - Connections: Outputs to "Update Guest - Clear URL"  
  - Failures: Missing guest ID, unexpected data structure

- **Update Guest - Clear URL**  
  - Type: NocoDB node (Update)  
  - Role: Updates the guest record to clear invalid LinkedIn URL and set error metadata  
  - Configuration:  
    - Table: CRM table ID  
    - ID: From input data  
    - Data to send: Auto mapped  
    - Authentication: NocoDB API token  
  - Connections: None (terminal node)  
  - Failures: API errors, invalid ID

---

#### 1.6 Error Handling  
**Overview:**  
Handles scraper failure or timeout cases by capturing error details and updating the CRM record with error status and message.

**Nodes Involved:**  
- Handle Scraper Error  
- Update Guest - Error Status

**Node Details:**  

- **Handle Scraper Error**  
  - Type: Code node (JavaScript)  
  - Role: Extracts error messages from failed scraper run or from node error output, prepares error update data  
  - Configuration:  
    - Runs once per item  
    - Reads guest ID and error info from JSON  
    - Returns object with scrape_status “error”, error_reason message, and timestamp  
  - Connections: Outputs to "Update Guest - Error Status"  
  - Failures: Missing guest ID, unexpected error format

- **Update Guest - Error Status**  
  - Type: NocoDB node (Update)  
  - Role: Updates guest record with error status and message  
  - Configuration:  
    - Table: CRM table ID  
    - ID: From input data  
    - Data to send: Auto mapped  
    - Authentication: NocoDB API token  
  - Connections: None (terminal node)  
  - Failures: API or auth errors

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                         | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                                               |
|---------------------------|------------------------|---------------------------------------|--------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger            | Trigger (Manual)       | Manual workflow start                  |                          | Get Guests with LinkedIn       |                                                                                                                           |
| Schedule Trigger          | Trigger (Schedule)     | Scheduled workflow start               |                          | Get Guests with LinkedIn       |                                                                                                                           |
| Get Guests with LinkedIn  | NocoDB (GetAll)        | Retrieve guests with LinkedIn URLs    | Manual Trigger, Schedule | Run Apify LinkedIn Scraper     | ## Get LinkedIn URL                                                                                                        |
| Run Apify LinkedIn Scraper| HTTP Request (POST)    | Start LinkedIn scraping via Apify     | Get Guests with LinkedIn  | Wait for Completion            | ## Get LinkedIn profile data                                                                                               |
| Wait for Completion       | HTTP Request (GET)     | Wait for Apify scraper run completion | Run Apify LinkedIn Scraper| Check Run Status              | ## Get LinkedIn profile data                                                                                               |
| Check Run Status          | If                     | Check if scraper run succeeded        | Wait for Completion      | Get Scraper Results (true) / Handle Scraper Error (false) |                                                                                                                           |
| Get Scraper Results       | Code                   | Fetch scraper dataset and validate    | Check Run Status (true)  | Transform Data (success) / Clear Broken LinkedIn URL (failure) |                                                                                                                           |
| Transform Data            | Code                   | Map LinkedIn data to CRM schema       | Get Scraper Results (success) | Update Guest Success       |                                                                                                                           |
| Update Guest Success      | NocoDB (Update)        | Update CRM guest with enriched data   | Transform Data           |                                |                                                                                                                           |
| Clear Broken LinkedIn URL | Code                   | Prepare data to clear invalid LinkedIn URL | Get Scraper Results (failure) | Update Guest - Clear URL   | ## Remove 404 linkedin URL                                                                                                |
| Update Guest - Clear URL  | NocoDB (Update)        | Clear broken LinkedIn URL in CRM      | Clear Broken LinkedIn URL|                                | ## Remove 404 linkedin URL                                                                                                |
| Handle Scraper Error      | Code                   | Prepare error data for scraper failure| Check Run Status (false) | Update Guest - Error Status    | ## Apify error/timeout                                                                                                     |
| Update Guest - Error Status| NocoDB (Update)       | Update CRM guest with scrape error    | Handle Scraper Error     |                                | ## Apify error/timeout                                                                                                     |
| Sticky Note               | Sticky Note            | Overview comment: Data enrichment flow|                          |                                | ## Data enrichment flow                                                                                                   |
| Sticky Note1              | Sticky Note            | Comment on 404 URL clearing           |                          |                                | ## Remove 404 linkedin URL                                                                                                |
| Sticky Note2              | Sticky Note            | Comment on Apify errors and timeouts |                          |                                | ## Apify error/timeout                                                                                                     |
| Sticky Note3              | Sticky Note            | Required NocoDB fields documentation  |                          |                                | ## Required fields in NocoDB table: Input LinkedIn field, multiple output fields listed                                     |
| Sticky Note4              | Sticky Note            | Comment on getting LinkedIn URL       |                          |                                | ## Get LinkedIn URL                                                                                                        |
| Sticky Note5              | Sticky Note            | (Empty)                              |                          |                                |                                                                                                                           |
| Sticky Note6              | Sticky Note            | (Empty)                              |                          |                                |                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - Purpose: Allow manual execution  
   - No parameters needed

2. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure schedule interval as needed (default empty array is placeholder)  

3. **Create NocoDB node "Get Guests with LinkedIn"**  
   - Operation: Get All  
   - Table: Specify your NocoDB table ID  
   - Filter: `(LinkedIn,isnot,null) AND (linkedin_headline,is,null)`  
   - Limit: 15 records  
   - Authentication: Set up NocoDB API token credentials with valid token  
   - Connect outputs from Manual Trigger and Schedule Trigger nodes to this node  

4. **Create HTTP Request node "Run Apify LinkedIn Scraper"**  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/dev_fusion~linkedin-profile-scraper/runs`  
   - Body Type: JSON  
   - Body Content: `{"profileUrls": ["{{$json.LinkedIn}}"]}`  
   - Authentication: Create and use HTTP Query Auth credentials with your Apify API token  
   - Connect output from "Get Guests with LinkedIn" node  

5. **Create HTTP Request node "Wait for Completion"**  
   - Method: GET  
   - URL: Expression: `https://api.apify.com/v2/acts/dev_fusion~linkedin-profile-scraper/runs/{{$json.data.id}}`  
   - Query Parameter: `waitForFinish=240` (string)  
   - Timeout: 300000 ms (5 minutes)  
   - Authentication: Use same Apify HTTP Query Auth credentials  
   - Connect output from "Run Apify LinkedIn Scraper"  

6. **Create If node "Check Run Status"**  
   - Condition: `$json.data.status === "SUCCEEDED"` (string equality, case sensitive)  
   - Connect output from "Wait for Completion"  

7. **Create Code node "Get Scraper Results"**  
   - Mode: Run Once For Each Item  
   - JavaScript code:  
     - Extract `datasetId` from `$json.data.defaultDatasetId`  
     - Use `this.helpers.httpRequest` to GET `https://api.apify.com/v2/datasets/${datasetId}/items?token=YOUR_APIFY_TOKEN`  
     - Validate data presence and structure; throw errors on missing or invalid data  
     - Return object `{ linkedinData: response }`  
   - On error: set to continue error output  
   - Connect If node True branch output  

8. **Create Code node "Transform Data"**  
   - Mode: Run Once For Each Item  
   - JavaScript code:  
     - Extract first profile from `linkedinData` array  
     - Read guest ID from "Get Guests with LinkedIn" node’s original item JSON  
     - Map all LinkedIn profile fields to CRM schema fields, including stringified arrays and timestamps  
     - Return transformed data object  
   - Connect output from "Get Scraper Results" node (success output)  

9. **Create NocoDB node "Update Guest Success"**  
   - Operation: Update  
   - Table: Your NocoDB table ID  
   - ID: Expression `={{$json.Id}}`  
   - Data to send: Auto map input data  
   - Authentication: NocoDB API token credentials  
   - Connect output from "Transform Data"  

10. **Create Code node "Clear Broken LinkedIn URL"**  
    - Mode: Run Once For Each Item  
    - JavaScript code:  
      - Extract guest ID and LinkedIn URL from "Get Guests with LinkedIn" node’s item JSON  
      - Return object clearing LinkedIn URL, setting `linkedin_scrape_status` to `'invalid_url'`, error reason, and timestamp  
    - Connect output from "Get Scraper Results" node (error output)  

11. **Create NocoDB node "Update Guest - Clear URL"**  
    - Operation: Update  
    - Table: Your NocoDB table ID  
    - ID: Expression `={{$json.Id}}`  
    - Data to send: Auto map input data  
    - Authentication: NocoDB API token  
    - Connect output from "Clear Broken LinkedIn URL"  

12. **Create Code node "Handle Scraper Error"**  
    - Mode: Run Once For Each Item  
    - JavaScript code:  
      - Extract guest ID from "Get Guests with LinkedIn" node’s JSON  
      - Extract error message from `$json.error.message` or fallback messages  
      - Return object with `linkedin_scrape_status` `'error'`, error reason, and timestamp  
    - Connect If node False branch output  

13. **Create NocoDB node "Update Guest - Error Status"**  
    - Operation: Update  
    - Table: Your NocoDB table ID  
    - ID: Expression `={{$json.Id}}`  
    - Data to send: Auto map input data  
    - Authentication: NocoDB API token  
    - Connect output from "Handle Scraper Error"  

14. **Connect nodes as per logical flow:**  
    - Manual Trigger and Schedule Trigger → Get Guests with LinkedIn → Run Apify LinkedIn Scraper → Wait for Completion → Check Run Status  
    - Check Run Status true → Get Scraper Results → Transform Data → Update Guest Success  
    - Check Run Status false → Handle Scraper Error → Update Guest - Error Status  
    - Get Scraper Results error output → Clear Broken LinkedIn URL → Update Guest - Clear URL  

15. **Configure all credentials:**  
    - NocoDB API token with read/write permissions for the CRM table  
    - Apify API token for LinkedIn scraper access  

16. **Replace placeholders:**  
    - `NOCODB_TABLEID` with your actual NocoDB table ID  
    - `NOCODB_PROJECTID` with your NocoDB project ID  
    - `NOCODB_APITOKENID` with your stored NocoDB API token credential ID  
    - `[YOUR APIFY TOKEN]` in code nodes with your actual Apify API token (or manage securely)  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow requires the NocoDB CRM table to have specific fields for LinkedIn enrichment, including input and output fields as documented in Sticky Note7. | Ensures schema compatibility and successful data mapping.                                          |
| Apify LinkedIn scraper API is used with a 4-minute wait timeout; longer profiles may cause timeout errors. Handle accordingly. | https://apify.com/docs/api/v2#/reference/acts/run-an-act/run-an-act                                  |
| The workflow includes robust error handling for 404 profiles (clears broken URLs) and other scraper errors (marks status). | Ensures data cleanliness and error transparency in CRM records.                                    |
| Manual and scheduled triggers enable flexibility in execution strategy depending on use case.                        |                                                                                                    |
| Video and blog tutorials on using Apify LinkedIn scraper and NocoDB API can assist further in customization.          | Apify Blog: https://apify.com/blog/linkedin-scraper/ ; NocoDB docs: https://docs.nocodb.com/       |

---

_Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected content. All handled data are legal and public._