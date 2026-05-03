Get Qualified Leads in One Click from Apollo to Airtable

https://n8nworkflows.xyz/workflows/get-qualified-leads-in-one-click-from-apollo-to-airtable-3435


# Get Qualified Leads in One Click from Apollo to Airtable

### 1. Workflow Overview

This workflow automates the process of extracting qualified leads from Apollo, enriching and organizing their data, and saving the results directly into Airtable for easy outreach and CRM integration. It targets users who want to bypass expensive lead generation tools and manual data handling by leveraging Apollo’s filtered contact lists and n8n’s automation capabilities.

**Use Cases:**  
- Founders and solo entrepreneurs managing DIY outbound campaigns  
- Growth marketers scaling cold email outreach  
- Agencies providing lead generation services  
- Anyone seeking cost-effective, clean, and enriched lead lists without manual exports or CSV handling

**Logical Blocks:**  
- **1.1 Input Reception:** Manual trigger to start the workflow  
- **1.2 Lead Scraping:** HTTP request to Apollo API to scrape leads based on a provided filter URL  
- **1.3 Lead Filtering:** Conditional filtering to exclude leads without emails  
- **1.4 Data Transformation:** Mapping and cleaning lead fields for consistency  
- **1.5 Data Storage:** Saving the enriched leads into an Airtable base for further use  
- **1.6 Informational Notes:** Sticky notes providing context, instructions, and branding  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually, allowing the user to trigger the lead generation process on demand.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow execution manually  
    - Configuration: No parameters; simply triggers downstream nodes  
    - Input: None (manual start)  
    - Output: Triggers "Scrape Leads" node  
    - Edge Cases: None typical; user must manually trigger  
    - Version: n8n v1+ compatible  

#### 2.2 Lead Scraping

- **Overview:**  
  This block sends a request to Apollo’s API to scrape leads matching the user’s filter URL, requesting enriched data including personal and work emails.

- **Nodes Involved:**  
  - Scrape Leads

- **Node Details:**  
  - **Scrape Leads**  
    - Type: HTTP Request  
    - Role: Fetches lead data from Apollo API based on user input URL  
    - Configuration:  
      - Method: POST (implied by JSON body)  
      - URL: Dynamically set (empty string placeholder in JSON, to be replaced by user input)  
      - Body (JSON):  
        - getPersonalEmails: true  
        - getWorkEmails: true  
        - totalRecords: 500 (max leads to fetch)  
        - url: (Apollo filter URL, to be provided dynamically)  
      - Send Body as JSON  
    - Input: Trigger from manual node  
    - Output: JSON array of lead objects with detailed fields  
    - Edge Cases:  
      - API authentication errors (not shown but assumed)  
      - Empty or invalid Apollo URL causing no results or errors  
      - API rate limits or timeouts  
    - Version: Requires n8n v0.150+ for HTTP Request node v4.2 features  

#### 2.3 Lead Filtering

- **Overview:**  
  Filters out any leads that do not have an email address, ensuring only contactable leads proceed.

- **Nodes Involved:**  
  - Filter leads without email

- **Node Details:**  
  - **Filter leads without email**  
    - Type: If (Conditional)  
    - Role: Checks if the lead’s email field exists and is non-empty  
    - Configuration:  
      - Condition: Check if `$json.email` exists (string exists operator)  
      - Case sensitive: true  
      - Type validation: strict  
    - Input: Leads JSON from "Scrape Leads"  
    - Output:  
      - True branch: leads with emails → "Edit Fields" node  
      - False branch: leads without emails → discarded (no connection)  
    - Edge Cases:  
      - Leads with malformed or empty email fields  
      - Potential false negatives if email field is missing or null  
    - Version: n8n v0.154+ for If node v2.2  

#### 2.4 Data Transformation

- **Overview:**  
  Maps and renames fields from Apollo’s raw lead data to a clean, consistent schema suitable for Airtable import.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**  
  - **Edit Fields**  
    - Type: Set  
    - Role: Assigns and renames lead properties to standardized field names  
    - Configuration:  
      - Assignments include:  
        - first_name ← $json.first_name  
        - last_name ← $json.last_name  
        - email ← $json.email  
        - email_status ← $json.email_status  
        - linkedin_url ← $json.linkedin_url  
        - headline ← $json.headline  
        - organization ← $json.organization_name  
        - organization_website ← $json.organization_website_url  
        - organization_linkedin_url ← $json.organization_linkedin_url  
        - current_job_title ← $json.title  
        - country ← $json.country  
        - city ← $json.city  
    - Input: Filtered leads with emails  
    - Output: Cleaned lead data to "Save Leads in database"  
    - Edge Cases:  
      - Missing fields in source data result in empty strings  
      - Expression evaluation errors if source fields are undefined  
    - Version: n8n v0.154+ for Set node v3.4  

#### 2.5 Data Storage

- **Overview:**  
  Saves the cleaned and enriched lead data into a predefined Airtable base and table for easy access and further processing.

- **Nodes Involved:**  
  - Save Leads in database

- **Node Details:**  
  - **Save Leads in database**  
    - Type: Airtable  
    - Role: Inserts new lead records into Airtable  
    - Configuration:  
      - Base: "Lead Gen - Mastersheet" (ID: appy1hlfTk0UuYwRb)  
      - Table: "Leads" (ID: tbl0rwfpUYkqMiysR)  
      - Columns mapped one-to-one from cleaned fields: city, country, headline, last_name, first_name, email_status, linkedin_url, email_address, current_job_title, organization_name, organization_website, organization_linkedin_url  
      - Operation: Create new records  
      - Matching columns: none (always create)  
      - Credential: Airtable Personal Access Token (configured)  
    - Input: Cleaned lead data from "Edit Fields"  
    - Output: Airtable API response (record creation confirmation)  
    - Edge Cases:  
      - Airtable API rate limits or authentication errors  
      - Schema mismatch if Airtable table structure changes  
      - Network timeouts  
    - Version: n8n v0.154+ for Airtable node v2.1  

#### 2.6 Informational Notes

- **Overview:**  
  Provides user-facing notes and branding information within the workflow editor for clarity and support.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  
  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Displays a summary message: "Lead Generation: Get thousands of enriched leads in seconds."  
    - Position: Top-left for visibility  
  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Displays branding and setup instructions:  
      - "🚨 readMeFirst 🚨"  
      - Template built by Not Another Marketer with link: https://notanothermarketer.com  
      - Step-by-step guide: https://notanothermarketer.gitbook.io/  
      - Support contact: https://x.com/notanothermrktr  
    - Position: Top-left, separate from main logic nodes  

---

### 3. Summary Table

| Node Name               | Node Type       | Functional Role                  | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                              |
|-------------------------|-----------------|--------------------------------|-----------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger  | Workflow start trigger          | None                        | Scrape Leads              |                                                                                                        |
| Scrape Leads            | HTTP Request    | Fetch leads from Apollo API     | When clicking ‘Test workflow’ | Filter leads without email |                                                                                                        |
| Filter leads without email | If              | Filter out leads without email  | Scrape Leads                | Edit Fields               |                                                                                                        |
| Edit Fields             | Set             | Map and clean lead data         | Filter leads without email  | Save Leads in database    |                                                                                                        |
| Save Leads in database  | Airtable        | Save leads into Airtable base   | Edit Fields                 | None                      |                                                                                                        |
| Sticky Note             | Sticky Note     | Lead generation summary message | None                        | None                      | "## Lead Generation\nGet thousands of enriched leads in seconds."                                     |
| Sticky Note1            | Sticky Note     | Branding and setup instructions | None                        | None                      | "## 🚨 readMeFirst 🚨\nThis template is built by [Not Another Marketer](https://notanothermarketer.com)\n\nStep-by-step setup guide: https://notanothermarketer.gitbook.io/\n\nAny questions? [Reach out on X](https://x.com/notanothermrktr)" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Type: Manual Trigger  
   - No parameters needed  
   - This node will start the workflow manually  

2. **Create HTTP Request Node**  
   - Name: "Scrape Leads"  
   - Type: HTTP Request  
   - Set HTTP Method to POST  
   - Set URL to Apollo API endpoint (to be configured by user)  
   - Under Body Parameters, select "JSON" and enter:  
     ```json
     {
       "getPersonalEmails": true,
       "getWorkEmails": true,
       "totalRecords": 500,
       "url": ""  // User must input Apollo filter URL here dynamically
     }
     ```  
   - Enable "Send Body" as JSON  
   - Connect "When clicking ‘Test workflow’" → "Scrape Leads"  

3. **Create If Node**  
   - Name: "Filter leads without email"  
   - Type: If  
   - Condition: Check if `$json.email` exists (string exists operator)  
   - Case sensitive: true  
   - Connect "Scrape Leads" → "Filter leads without email"  

4. **Create Set Node**  
   - Name: "Edit Fields"  
   - Type: Set  
   - Add fields with exact names and assign values from incoming JSON:  
     - first_name = `{{$json.first_name}}`  
     - last_name = `{{$json.last_name}}`  
     - email = `{{$json.email}}`  
     - email_status = `{{$json.email_status}}`  
     - linkedin_url = `{{$json.linkedin_url}}`  
     - headline = `{{$json.headline}}`  
     - organization = `{{$json.organization_name}}`  
     - organization_website = `{{$json.organization_website_url}}`  
     - organization_linkedin_url = `{{$json.organization_linkedin_url}}`  
     - current_job_title = `{{$json.title}}`  
     - country = `{{$json.country}}`  
     - city = `{{$json.city}}`  
   - Connect "Filter leads without email" (true output) → "Edit Fields"  

5. **Create Airtable Node**  
   - Name: "Save Leads in database"  
   - Type: Airtable  
   - Operation: Create  
   - Configure Credentials: Use Airtable Personal Access Token with write permissions  
   - Select Base: "Lead Gen - Mastersheet" (or your own base)  
   - Select Table: "Leads" (or your own table)  
   - Map fields exactly as:  
     - city ← `{{$json.city}}`  
     - country ← `{{$json.country}}`  
     - headline ← `{{$json.headline}}`  
     - last_name ← `{{$json.last_name}}`  
     - first_name ← `{{$json.first_name}}`  
     - email_status ← `{{$json.email_status}}`  
     - linkedin_url ← `{{$json.linkedin_url}}`  
     - email_address ← `{{$json.email}}`  
     - current_job_title ← `{{$json.current_job_title}}`  
     - organization_name ← `{{$json.organization}}`  
     - organization_website ← `{{$json.organization_website}}`  
     - organization_linkedin_url ← `{{$json.organization_linkedin_url}}`  
   - Connect "Edit Fields" → "Save Leads in database"  

6. **Add Sticky Notes for Documentation (Optional)**  
   - Create a Sticky Note with content:  
     "## Lead Generation\nGet thousands of enriched leads in seconds."  
   - Create another Sticky Note with content:  
     "## 🚨 readMeFirst 🚨\nThis template is built by [Not Another Marketer](https://notanothermarketer.com)\n\nStep-by-step setup guide: https://notanothermarketer.gitbook.io/\n\nAny questions? [Reach out on X](https://x.com/notanothermrktr)"  

7. **Test the Workflow**  
   - Trigger manually using "When clicking ‘Test workflow’"  
   - Provide a valid Apollo filter URL in the "Scrape Leads" node’s JSON body before execution  
   - Verify leads are scraped, filtered, transformed, and saved in Airtable  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This template is built by Not Another Marketer                                                           | https://notanothermarketer.com                                                                      |
| Step-by-step setup guide available                                                                        | https://notanothermarketer.gitbook.io/home/templates/lead-generation                                |
| Support and questions can be directed to Not Another Marketer on X (formerly Twitter)                      | https://x.com/notanothermrktr                                                                       |
| The workflow enables cost-effective lead generation by leveraging Apollo data without expensive subscriptions | Workflow description and value proposition                                                         |
| Airtable base and table IDs are placeholders; users should replace with their own Airtable configuration | Airtable integration requires user-specific base/table setup and API token                           |

---

This documentation fully describes the "Get Qualified Leads in One Click from Apollo to Airtable" workflow, enabling advanced users and automation agents to understand, reproduce, and maintain the workflow effectively.