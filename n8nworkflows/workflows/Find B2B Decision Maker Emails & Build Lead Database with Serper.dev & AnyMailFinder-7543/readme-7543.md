Find B2B Decision Maker Emails & Build Lead Database with Serper.dev & AnyMailFinder

https://n8nworkflows.xyz/workflows/find-b2b-decision-maker-emails---build-lead-database-with-serper-dev---anymailfinder-7543


# Find B2B Decision Maker Emails & Build Lead Database with Serper.dev & AnyMailFinder

### 1. Workflow Overview

This workflow automates the process of finding B2B decision maker emails and building a lead database by combining data from company records, web search results, AI analysis, and email finder APIs. It is designed for sales and marketing teams seeking to enrich company data with verified email contacts of key decision makers (CEO, sales, marketing).

The core logical blocks are:

- **1.1 Input Reception & Company Data Retrieval**: Trigger and fetch company data from a NocoDB table.
- **1.2 Domain Discovery via Serper.dev and AI**: Search Google using Serper.dev API, analyze results with OpenAI to identify official company domains, and update company records.
- **1.3 Decision Maker Email Retrieval via Anymailfinder**: Query Anymailfinder API to find emails of CEO, sales, and marketing decision makers based on company domain or name.
- **1.4 Data Extraction & Contact Creation**: Extract relevant fields from API responses, remove duplicates, create contacts in NocoDB.
- **1.5 Email Status Aggregation and Company Status Update**: Aggregate email validity statuses by company, update company records accordingly.
- **1.6 Bulk Company Email Search & Update**: For companies lacking valid emails, perform a broader email search with Anymailfinder and update records.
- **1.7 Control & Scheduling**: Manual and scheduled triggers to start the workflow, batching and waiting mechanisms to handle rate limits and partial failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Company Data Retrieval

- **Overview:**  
Starts the workflow manually or via schedule and fetches all company entries from the NocoDB "companies" table to process.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger2 (Scheduled monthly trigger)  
  - Get many rows (NocoDB data retrieval)  
  - Loop Over Items (Batch processing)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual start of workflow for testing or on-demand runs.  
    - Connections: Outputs to "Get many rows".  
    - Edge Cases: No input parameters; manual start only.

  - **Schedule Trigger2**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow every 12 months to refresh data.  
    - Configuration: Interval set to 12 months.  
    - Connections: Outputs to "Get many rows".  
    - Notes: Enables automation without manual intervention.

  - **Get many rows**  
    - Type: NocoDB node (getAll operation)  
    - Role: Retrieves all company records from table "m4v2qbu9q4yewh4" in NocoDB project "p3iac4hmm93iief".  
    - Configuration: Returns all rows, no filtering. Uses API token authentication.  
    - Connections: Outputs to "Loop Over Items".  
    - Edge Cases: API token failure, empty table, large dataset handled by batching.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes company records in batches of 500 to avoid rate limits or timeouts.  
    - Connections: On batch output to "Filter1", on batch completion to "Wait".  
    - Edge Cases: Batch size too large may cause timeouts; small batch size increases runtime.

#### 2.2 Domain Discovery via Serper.dev and AI

- **Overview:**  
Uses Serper.dev to perform Google search queries for company domains, then uses OpenAI to analyze search results and extract official domain info, then updates company records.

- **Nodes Involved:**  
  - Filter1 (Filter companies without domain status)  
  - serper search domains (HTTP Request to Serper.dev)  
  - extract url & domain (OpenAI node)  
  - Update Companies Domains (NocoDB update)

- **Node Details:**

  - **Filter1**  
    - Type: Filter  
    - Role: Passes only companies with empty "status" field (i.e., domains not found yet).  
    - Condition: `status` field is empty.  
    - Connections: To "serper search domains".  
    - Edge Cases: Misconfigured field names or missing status field.

  - **serper search domains**  
    - Type: HTTP Request  
    - Role: Calls Serper.dev Google search API with company name and location as query parameters.  
    - Configuration: POST request to https://google.serper.dev/search with JSON body containing `q` (company_name or fallback "a") and `location`. Authentication via HTTP header API key.  
    - Connections: To "extract url & domain".  
    - Edge Cases: API rate limits, network errors, malformed JSON responses. On error, continues regular output to avoid workflow halt.

  - **extract url & domain**  
    - Type: OpenAI (Langchain)  
    - Role: Processes Serper.dev results to select official company website URL and domain using a detailed prompt and logic.  
    - Configuration: GPT-4.1-mini model, low temperature (0.2) for deterministic output, outputs JSON object with company_name, location, url, domain, explanation.  
    - Connections: To "Update Companies Domains".  
    - Edge Cases: OpenAI API failures, unexpected response format, invalid or missing input data.

  - **Update Companies Domains**  
    - Type: NocoDB update  
    - Role: Updates company record with discovered domain, URL, and status ("domain found" or "domain not found").  
    - Uses data from AI node message content.  
    - Connections: To three email finder nodes: "Get Sales Decision Maker Email", "Get CEO Email", "Get Marketing Email".  
    - Edge Cases: Update failure, concurrent updates, missing Id field.

#### 2.3 Decision Maker Email Retrieval via Anymailfinder

- **Overview:**  
Queries Anymailfinder API for emails of decision makers in categories: sales, CEO, and marketing using discovered domain or company name.

- **Nodes Involved:**  
  - Get Sales Decision Maker Email  
  - Get CEO Email  
  - Get Marketing Email  
  - Extract Data5 (Sales), Extract Data3 (CEO), Extract Data4 (Marketing)  
  - Merge1 (merges extracted data)

- **Node Details:**

  - **Get Sales Decision Maker Email**  
    - Type: HTTP Request  
    - Role: POST to Anymailfinder API endpoint for sales decision maker emails.  
    - Parameters: Uses domain if available, else first company name split by comma. Category "sales".  
    - Authentication: HTTP header API key.  
    - Connections: To "Extract Data5".  
    - Edge Cases: API errors, quota limits, invalid domain or company name.

  - **Get CEO Email**  
    - Same as above but category "ceo".  
    - Connections: To "Extract Data3".

  - **Get Marketing Email**  
    - Same as above but category "marketing".  
    - Connections: To "Extract Data4".

  - **Extract Data5 / Extract Data3 / Extract Data4**  
    - Type: Set  
    - Role: Extracts and normalizes relevant fields from Anymailfinder response such as company_name, person_full_name, linkedin url, job title, email, email_status, company_id.  
    - Uses expressions referencing "Get many rows" and "Update Companies Domains" for company info and IDs.  
    - Connections: To "Merge1" inputs 0, 1, 2 respectively.  
    - Edge Cases: Missing fields, empty responses, data type mismatches.

  - **Merge1**  
    - Type: Merge  
    - Role: Merges the three decision maker categories data arrays into single stream.  
    - Number of inputs: 3 (Sales, CEO, Marketing).  
    - Connections: To "Filter".  
    - Edge Cases: Handling empty inputs, duplicate records.

#### 2.4 Data Extraction & Contact Creation

- **Overview:**  
Filters merged email data, removes duplicates, creates contacts in NocoDB for valid emails.

- **Nodes Involved:**  
  - Filter (filters items that have non-empty email)  
  - Remove Duplicates (removes duplicate emails)  
  - Create Contacts (NocoDB create operation)  
  - Determine Email Status by Company (code node for aggregation)

- **Node Details:**

  - **Filter**  
    - Type: Filter  
    - Role: Allows only items where `email` field exists and is not empty.  
    - Connections: To "Remove Duplicates".  
    - Edge Cases: Missing email field.

  - **Remove Duplicates**  
    - Type: RemoveDuplicates  
    - Role: Removes duplicate entries based on the `email` field to avoid redundancy.  
    - Connections: To "Create Contacts".  
    - Edge Cases: Case sensitivity in emails.

  - **Create Contacts**  
    - Type: NocoDB create  
    - Role: Creates new contact records in NocoDB table "m1yxldkvwib7f98" with extracted person info and email status.  
    - Connections: To "Determine Email Status by Company".  
    - Edge Cases: Duplicates in database, insertion errors.

  - **Determine Email Status by Company**  
    - Type: Code (JavaScript)  
    - Role: Aggregates email statuses per company; if all emails are "risky", status is "risky", else "valid".  
    - Output: Array of objects with `companies_id` and aggregated `email_status`.  
    - Connections: To "Update Company Status".  
    - Edge Cases: Missing or inconsistent email statuses.

#### 2.5 Email Status Aggregation and Company Status Update

- **Overview:**  
Updates each company's email status in NocoDB based on aggregated email validity.

- **Nodes Involved:**  
  - Update Company Status (NocoDB update)  
  - Get All Company Status (NocoDB getAll)  
  - If Email Found1 (IF node)  
  - If Only Risky Emails1 (Filter node)  
  - Merge (merge node combining branches)  
  - Get All Company Emails (Anymailfinder broad search)  
  - Update Company Emails (NocoDB update)

- **Node Details:**

  - **Update Company Status**  
    - Type: NocoDB update  
    - Role: Updates company record with email status string "Email Found: [valid|risky]".  
    - Connections: None (end of chain).  
    - Edge Cases: Update failures.

  - **Get All Company Status**  
    - Type: NocoDB getAll  
    - Role: Retrieves all companies to check current statuses. Used for decision making.  
    - Connections: To "If Email Found1".  
    - ExecuteOnce: true for performance.

  - **If Email Found1**  
    - Type: If  
    - Role: Checks if company's status contains "Email Found".  
    - True branch: To "If Only Risky Emails1".  
    - False branch: To "Merge" (to trigger broad email search).  
    - Edge Cases: String matching errors.

  - **If Only Risky Emails1**  
    - Type: Filter  
    - Role: Checks if company status contains "risky" (only risky emails found).  
    - True branch: To "Merge" (broad search needed).  
    - Edge Cases: Case sensitivity.

  - **Merge**  
    - Type: Merge  
    - Role: Combines control flow branches before triggering broad email search.  
    - Connections: To "Get All Company Emails".

  - **Get All Company Emails**  
    - Type: HTTP Request  
    - Role: Calls Anymailfinder API to get up to 20 emails per company domain or name for companies lacking valid emails.  
    - Batch size: 20 emails per request.  
    - Connections: To "Update Company Emails".  
    - Edge Cases: API quota, large companies may cause partial data.

  - **Update Company Emails**  
    - Type: NocoDB update  
    - Role: Updates company record with all emails found and their status.  
    - Connections: Back to "Loop Over Items" for continued processing.  
    - Edge Cases: Data length limits.

#### 2.6 Control & Scheduling

- **Overview:**  
Manages overall flow control, batching, waiting to respect API limits, and scheduling.

- **Nodes Involved:**  
  - Wait (Pause between batches)  
  - Loop Over Items (Batch splitting)  
  - Schedule Trigger2 (Scheduled monthly run)  
  - When clicking ‘Execute workflow’ (Manual trigger)

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 1 minute between batches to avoid API rate limit issues.  
    - Connections: To "Filter1" for next batch processing.  
    - Edge Cases: Too short wait may cause rate limits.

  - All other nodes as described in prior blocks.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                                  | Input Node(s)                          | Output Node(s)                                    | Sticky Note                                                                                                         |
|-------------------------------|---------------------------|-------------------------------------------------|--------------------------------------|--------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual start trigger                             | None                                 | Get many rows                                    |                                                                                                                     |
| Schedule Trigger2              | Schedule Trigger          | Scheduled annual trigger                         | None                                 | Get many rows                                    |                                                                                                                     |
| Get many rows                 | NocoDB (getAll)           | Retrieve all companies                           | When clicking ‘Execute workflow’, Schedule Trigger2 | Loop Over Items                                 |                                                                                                                     |
| Loop Over Items               | SplitInBatches            | Batch processing of company records             | Get many rows                        | Filter1, Wait                                    |                                                                                                                     |
| Wait                         | Wait                      | Pause 1 minute between batches                   | Loop Over Items                     | Filter1                                          |                                                                                                                     |
| Filter1                      | Filter                    | Filter companies without domain status          | Loop Over Items, Wait                | serper search domains                            |                                                                                                                     |
| serper search domains         | HTTP Request              | Search Google for company domains via Serper.dev | Filter1                            | extract url & domain                             |                                                                                                                     |
| extract url & domain          | OpenAI (Langchain)        | Analyze search results to find official domain  | serper search domains               | Update Companies Domains                         |                                                                                                                     |
| Update Companies Domains      | NocoDB (update)           | Update company records with domain info         | extract url & domain                | Get Sales Decision Maker Email, Get CEO Email, Get Marketing Email |                                                                                                                     |
| Get Sales Decision Maker Email | HTTP Request              | Find sales decision maker email via Anymailfinder | Update Companies Domains           | Extract Data5                                    |                                                                                                                     |
| Get CEO Email                | HTTP Request              | Find CEO email via Anymailfinder                 | Update Companies Domains           | Extract Data3                                    |                                                                                                                     |
| Get Marketing Email          | HTTP Request              | Find marketing decision maker email via Anymailfinder | Update Companies Domains           | Extract Data4                                    |                                                                                                                     |
| Extract Data5                | Set                       | Extract fields from sales email API response    | Get Sales Decision Maker Email      | Merge1                                           |                                                                                                                     |
| Extract Data3                | Set                       | Extract fields from CEO email API response      | Get CEO Email                      | Merge1                                           |                                                                                                                     |
| Extract Data4                | Set                       | Extract fields from marketing email API response | Get Marketing Email                | Merge1                                           |                                                                                                                     |
| Merge1                      | Merge                     | Combine all decision maker emails                | Extract Data5, Extract Data3, Extract Data4 | Filter                                           |                                                                                                                     |
| Filter                      | Filter                    | Filter out entries without email                 | Merge1                            | Remove Duplicates                                |                                                                                                                     |
| Remove Duplicates            | RemoveDuplicates          | Remove duplicate emails                           | Filter                            | Create Contacts                                  |                                                                                                                     |
| Create Contacts             | NocoDB (create)           | Insert contact records into database             | Remove Duplicates                 | Determine Email Status by Company                |                                                                                                                     |
| Determine Email Status by Company | Code                     | Aggregate email validity status per company      | Create Contacts                  | Update Company Status                            |                                                                                                                     |
| Update Company Status        | NocoDB (update)           | Update company record with email status          | Determine Email Status by Company | None                                             |                                                                                                                     |
| Get All Company Status       | NocoDB (getAll)           | Retrieve all company statuses                     | None (Execute Once)               | If Email Found1                                  |                                                                                                                     |
| If Email Found1             | If                        | Check if company has any found email              | Get All Company Status            | If Only Risky Emails1 (true), Merge (false)     |                                                                                                                     |
| If Only Risky Emails1         | Filter                    | Check if all emails are risky                      | If Email Found1                  | Merge (true)                                     |                                                                                                                     |
| Merge                       | Merge                     | Combine branches for broad email search           | If Email Found1, If Only Risky Emails1 | Get All Company Emails                          |                                                                                                                     |
| Get All Company Emails       | HTTP Request              | Broad search for all company emails via Anymailfinder | Merge                          | Update Company Emails                            |                                                                                                                     |
| Update Company Emails        | NocoDB (update)           | Update company emails and status                   | Get All Company Emails           | Loop Over Items                                  |                                                                                                                     |
| Sticky Note                  | Sticky Note               | Explains domain finding process                   | None                             | None                                             | ## Find Domain from company - User Serper.dev - Filter results using ai - Log the Records in Nocodb using Batches... |
| Sticky Note1                 | Sticky Note               | Explains decision maker email finding             | None                             | None                                             | ## Find decision makers - Use [anymailfinder](https://anymailfinder.com/?via=alexandra) - search for marketing, sales and CEO Categories... |
| Sticky Note2                 | Sticky Note               | Explains email filtering and duplicate removal    | None                             | None                                             | ## Filter emails found and remove duplicates - Create Contacts for the ones where we have an email                     |
| Sticky Note3                 | Sticky Note               | Explains status update if company has valid email | None                             | None                                             | ## Find if company has almost one valid email - then log the status in the company table                               |
| Sticky Note4                 | Sticky Note               | Explains broad company email search if no valid emails | None                             | None                                             | ## Search All Companies email - if a company has not any email found yet, or if all the email found are risky - then search all company email (1 credit = up to 20 emails) |
| Sticky Note5                 | Sticky Note               | Explains update of company emails                  | None                             | None                                             | ## Update company emails                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’".
   - Add a **Schedule Trigger** node named "Schedule Trigger2" with an interval of 12 months.

2. **Retrieve Companies:**
   - Add a **NocoDB** node named "Get many rows" configured to perform `getAll` on table "m4v2qbu9q4yewh4" from project "p3iac4hmm93iief".
   - Authenticate with a NocoDB API token credential.
   - Connect outputs of both triggers to this node.

3. **Batch Processing:**
   - Add a **SplitInBatches** node named "Loop Over Items" with batch size 500.
   - Connect "Get many rows" output to this node.

4. **Wait Node:**
   - Add a **Wait** node named "Wait" configured to pause 1 minute.
   - Connect the secondary output of "Loop Over Items" (batch completion) to "Wait".
   - Connect "Wait" output back to a filter node for next batch.

5. **Filter Companies Without Domain:**
   - Add a **Filter** node named "Filter1" with condition: `status` field is empty.
   - Connect "Loop Over Items" main output and "Wait" output to "Filter1".

6. **Serper.dev Domain Search:**
   - Add an **HTTP Request** node named "serper search domains".
   - Configure to POST to `https://google.serper.dev/search`.
   - Body parameters:  
     - `q`: expression `{{$json.company_name || "a"}}`  
     - `location`: `{{$json.location}}`  
   - Authenticate with an HTTP header API key credential for Serper.dev.
   - Connect "Filter1" to this node.

7. **OpenAI Domain Extraction:**
   - Add an **OpenAI** node named "extract url & domain".
   - Use GPT-4.1-mini model, temperature 0.2.
   - Paste the provided prompt that instructs AI to parse search results and extract company domain info.
   - Enable JSON output.
   - Connect "serper search domains" output to this node.

8. **Update Company Domains:**
   - Add a **NocoDB** node named "Update Companies Domains" to update table "m4v2qbu9q4yewh4".
   - Fields to update:  
     - `url` from AI output content.url or explanation  
     - `domain` from AI output content.domain  
     - `status` set to "domain found" if domain exists, else "domain not found"  
     - `Id` from current company record  
   - Authenticate with NocoDB API token.
   - Connect "extract url & domain" output to this node.

9. **Anymailfinder Decision Maker Emails:**
   - Create three **HTTP Request** nodes named:  
     - "Get Sales Decision Maker Email" (category: sales)  
     - "Get CEO Email" (category: ceo)  
     - "Get Marketing Email" (category: marketing)  
   - URL: `https://api.anymailfinder.com/v5.1/find-email/decision-maker` POST requests.
   - Body parameters:  
     - Dynamic key: `domain` if domain exists, else `company_name` (first part split by comma).  
     - `decision_maker_category`: set accordingly.  
   - Authenticate with HTTP header API key for Anymailfinder.
   - Connect "Update Companies Domains" output to all three nodes.

10. **Extract Data from Decision Maker Responses:**
    - Add three **Set** nodes named "Extract Data5" (sales), "Extract Data3" (CEO), "Extract Data4" (marketing).
    - Extract fields: company_name, name, linkedin, person_job_title, email, email_status, company_id.
    - Use expressions referencing "Get many rows" and "Update Companies Domains" for context.
    - Connect respective Anymailfinder nodes to their extract nodes.

11. **Merge Decision Maker Data:**
    - Add a **Merge** node named "Merge1" with 3 inputs.
    - Connect "Extract Data5", "Extract Data3", "Extract Data4" outputs to "Merge1" inputs.

12. **Filter and Deduplicate Emails:**
    - Add a **Filter** node named "Filter" to allow items with non-empty `email`.
    - Connect "Merge1" output to "Filter".
    - Add a **Remove Duplicates** node named "Remove Duplicates" to remove duplicate emails based on `email`.
    - Connect "Filter" to "Remove Duplicates".

13. **Create Contacts in NocoDB:**
    - Add a **NocoDB** node named "Create Contacts" to create records in table "m1yxldkvwib7f98".
    - Map fields: companies_id, name, position, email, email_status, linkedin_url.
    - Connect "Remove Duplicates" to "Create Contacts".

14. **Aggregate Email Status:**
    - Add a **Code** node named "Determine Email Status by Company".
    - Paste the JavaScript code that groups emails by company and sets status to "risky" if all emails are risky, else "valid".
    - Connect "Create Contacts" to this node.

15. **Update Company Email Status:**
    - Add a **NocoDB** node named "Update Company Status".
    - Update "m4v2qbu9q4yewh4" with `status` field as "Email Found: [email_status]" for each company ID.
    - Connect "Determine Email Status by Company" to this node.

16. **Retrieve All Company Statuses:**
    - Add a **NocoDB** node named "Get All Company Status" to get all company statuses.
    - Set to execute once per workflow run.
    - Connect to "If Email Found1".

17. **Conditional Checks for Broad Email Search:**
    - Add an **If** node named "If Email Found1" checking if `status` contains "Email Found".
      - True: connect to "If Only Risky Emails1".
      - False: connect to "Merge".
    - Add a **Filter** node named "If Only Risky Emails1" checking if `status` contains "risky".
      - True: connect to "Merge".
    - Add a **Merge** node named "Merge" to join these branches.

18. **Broad Company Email Search:**
    - Add an **HTTP Request** node named "Get All Company Emails".
    - POST to `https://api.anymailfinder.com/v5.1/find-email/company`.
    - Body param: domain or company_name (first split).
    - Batch size parameter set to 20 emails.
    - Authenticate with Anymailfinder HTTP header key.
    - Connect "Merge" output to this node.

19. **Update Company Emails:**
    - Add a **NocoDB** node named "Update Company Emails" to update company emails and email status fields.
    - Connect "Get All Company Emails" output to this node.
    - Connect this node back to "Loop Over Items" to continue processing batches.

20. **Add Sticky Notes:**
    - Add sticky notes with contents as in the original workflow describing each major section.
    - Place near relevant nodes for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                 |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| Uses Serper.dev Google Search API for domain discovery.                                                              | https://serper.dev/                                                             |
| Uses OpenAI GPT-4.1-mini model for AI analysis of search results to identify official company domain.                 | OpenAI API                                                                       |
| Uses Anymailfinder API to find decision maker emails in categories CEO, Sales, Marketing with email validation status. | https://anymailfinder.com/?via=alexandra                                         |
| Employs NocoDB as the backend database for company and contact management.                                            | https://nocodb.com/                                                              |
| Workflow handles batching and waiting to respect API quotas and avoid timeouts.                                       |                                                                                 |
| Email status aggregation logic flags companies with all risky emails for further broad email search.                  |                                                                                 |
| Cost-sensitive API usage: 2 credits per valid email from decision makers; broad search costs 1 credit per 20 emails.  |                                                                                 |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.