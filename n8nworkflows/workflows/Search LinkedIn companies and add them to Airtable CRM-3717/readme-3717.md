Search LinkedIn companies and add them to Airtable CRM

https://n8nworkflows.xyz/workflows/search-linkedin-companies-and-add-them-to-airtable-crm-3717


# Search LinkedIn companies and add them to Airtable CRM

### 1. Workflow Overview

This workflow automates the process of searching for companies on LinkedIn using the Ghost Genius API and adding qualified companies to an Airtable CRM database. It is designed for sales teams, business development professionals, and marketers who want to build a prospect database efficiently without manual LinkedIn research.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Company Search:** Receives manual trigger input, sets search parameters, and queries the Ghost Genius API to retrieve a list of companies matching specified criteria.

- **1.2 Company Data Processing and Filtering:** Processes each company individually, retrieves detailed company information, and filters companies based on quality indicators such as follower count and website availability.

- **1.3 CRM Integration:** Checks if each company already exists in Airtable to avoid duplicates and adds new companies with detailed information to the Airtable CRM.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Company Search

**Overview:**  
This block initiates the workflow manually, sets the search parameters for LinkedIn company search, and performs the search via the Ghost Genius API. It handles pagination and extracts the list of companies from the API response.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Set Variables  
- Search Companies (HTTP Request)  
- Extract Company Data (Split Out)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually for testing or execution.  
  - Configuration: No parameters; simply triggers the workflow.  
  - Inputs: None  
  - Outputs: Triggers "Set Variables" node.  
  - Edge Cases: None specific; user must trigger manually.

- **Set Variables**  
  - Type: Set  
  - Role: Defines search criteria variables such as target keywords, company size, and location ID.  
  - Configuration:  
    - "Your target": "Growth Marketing Agency"  
    - Company size code: "C" (11-50 employees)  
    - Location: "103644278" (LinkedIn location ID)  
  - Expressions: Variables are used as query parameters in the next node.  
  - Inputs: From Manual Trigger  
  - Outputs: To "Search Companies" node  
  - Edge Cases: Incorrect or missing location ID or size code may return no results.

- **Search Companies**  
  - Type: HTTP Request  
  - Role: Queries Ghost Genius API to search companies on LinkedIn based on set variables.  
  - Configuration:  
    - URL: `https://api.ghostgenius.fr/v2/search/companies`  
    - Pagination enabled with 2-second interval between pages, limits pages fetched based on API response.  
    - Query parameters: keywords, company_size, location (from Set Variables)  
    - Authentication: HTTP Header Auth with Bearer token (Ghost Genius API key)  
  - Inputs: From "Set Variables"  
  - Outputs: To "Extract Company Data"  
  - Edge Cases: API rate limits, invalid API key, network timeouts, or empty results.

- **Extract Company Data**  
  - Type: Split Out  
  - Role: Extracts the "data" array from the API response to process companies individually.  
  - Configuration: Field to split out: "data"  
  - Inputs: From "Search Companies"  
  - Outputs: To "Process Each Company"  
  - Edge Cases: Empty or malformed data array.

---

#### 2.2 Company Data Processing and Filtering

**Overview:**  
This block processes each company individually by retrieving detailed company information from the Ghost Genius API, then filters companies based on follower count and website availability to ensure quality.

**Nodes Involved:**  
- Process Each Company (Split In Batches)  
- Get Company Info (HTTP Request)  
- Filter Valid Companies (If)

**Node Details:**

- **Process Each Company**  
  - Type: Split In Batches  
  - Role: Processes companies one at a time to respect API rate limits and avoid timeouts.  
  - Configuration: Default batch size (1), no special options.  
  - Inputs: From "Extract Company Data" and from "Add Company to CRM" (loop back)  
  - Outputs:  
    - Main output: To "Get Company Info"  
    - Second output (empty): To continue workflow if no further processing needed.  
  - Edge Cases: Batch processing errors, empty batches.

- **Get Company Info**  
  - Type: HTTP Request  
  - Role: Retrieves detailed company information using the company LinkedIn URL.  
  - Configuration:  
    - URL: `https://api.ghostgenius.fr/v2/company`  
    - Query parameter: `url` set dynamically from current company's LinkedIn URL (`{{$json.url}}`)  
    - Authentication: HTTP Header Auth with Bearer token (Ghost Genius API key)  
    - Batching: 1 request per 2 seconds to respect API limits  
    - Retry on failure enabled  
  - Inputs: From "Process Each Company"  
  - Outputs: To "Filter Valid Companies"  
  - Edge Cases: API failures, invalid URLs, rate limiting, timeouts.

- **Filter Valid Companies**  
  - Type: If  
  - Role: Filters companies to only those with a non-empty website and followers count greater than 200.  
  - Configuration:  
    - Conditions:  
      - Website field is not empty  
      - Followers count > 200  
  - Inputs: From "Get Company Info"  
  - Outputs:  
    - True: To "Check If Company Exists" (valid companies)  
    - False: Loops back to "Process Each Company" (skip invalid companies)  
  - Edge Cases: Missing fields, unexpected data types.

---

#### 2.3 CRM Integration

**Overview:**  
This block integrates with Airtable CRM. It checks if a company already exists in the CRM by LinkedIn ID to avoid duplicates. If the company is new, it adds the company with detailed fields to the Airtable base.

**Nodes Involved:**  
- Check If Company Exists (Airtable)  
- Is New Company? (If)  
- Add Company to CRM (Airtable)

**Node Details:**

- **Check If Company Exists**  
  - Type: Airtable  
  - Role: Searches the Airtable "CRM" base to find if the company with the given LinkedIn ID already exists.  
  - Configuration:  
    - Base: "CRM" (appjYSpxvs8mlJaIW)  
    - Table: "CRM" (tbliG5xhydGGgk3nD)  
    - Operation: Search  
    - Filter formula: `{id} = '{{ $json.id.toNumber() }}'` (matches company ID)  
    - Credentials: Airtable Personal Access Token  
  - Inputs: From "Filter Valid Companies"  
  - Outputs: To "Is New Company?"  
  - Edge Cases: API errors, invalid credentials, rate limits.

- **Is New Company?**  
  - Type: If  
  - Role: Checks if the search result from Airtable is empty (i.e., company does not exist).  
  - Configuration:  
    - Condition: The first item in the Airtable search result is empty (no match found)  
  - Inputs: From "Check If Company Exists"  
  - Outputs:  
    - True: To "Add Company to CRM" (new company)  
    - False: Loops back to "Process Each Company" (skip existing company)  
  - Edge Cases: Unexpected Airtable response structure.

- **Add Company to CRM**  
  - Type: Airtable  
  - Role: Adds a new company record to the Airtable CRM with mapped fields.  
  - Configuration:  
    - Base: "CRM"  
    - Table: "CRM"  
    - Operation: Create  
    - Columns mapped:  
      - id: Company ID (number)  
      - Name: Company name  
      - Country: Fixed value "🇺🇸 United States" (customizable)  
      - Summary: Company description  
      - Tagline: Company tagline  
      - Website: Company website URL  
      - Category: Fixed value "Growth Marketing Agency 11-50 🌍" (customizable)  
      - LinkedIn: Company LinkedIn URL  
    - Credentials: Airtable Personal Access Token  
  - Inputs: From "Is New Company?"  
  - Outputs: Loops back to "Process Each Company" for next company processing  
  - Edge Cases: Airtable API errors, field mapping mismatches.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                         | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                         |
|-------------------------|---------------------|---------------------------------------|------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts workflow manually               | None                         | Set Variables               |                                                                                                                     |
| Set Variables           | Set                 | Defines search criteria variables      | When clicking ‘Test workflow’ | Search Companies            |                                                                                                                     |
| Search Companies        | HTTP Request        | Searches companies via Ghost Genius API| Set Variables                | Extract Company Data        | LinkedIn Company Search: explains search criteria, pagination, and tips                                              |
| Extract Company Data    | Split Out           | Extracts company list from API response| Search Companies             | Process Each Company        | LinkedIn Company Search                                                                                              |
| Process Each Company    | Split In Batches    | Processes companies one by one         | Extract Company Data, Add Company to CRM | Get Company Info, (empty output) | Company Data Processing: explains batch processing and API rate limits                                              |
| Get Company Info        | HTTP Request        | Retrieves detailed company info        | Process Each Company         | Filter Valid Companies      | Company Data Processing                                                                                              |
| Filter Valid Companies  | If                  | Filters companies by website and followers count | Get Company Info             | Check If Company Exists, Process Each Company | Company Data Processing                                                                                              |
| Check If Company Exists | Airtable            | Checks if company exists in CRM        | Filter Valid Companies       | Is New Company?             | CRM Integration: explains duplicate check and CRM addition                                                          |
| Is New Company?         | If                  | Determines if company is new            | Check If Company Exists      | Add Company to CRM, Process Each Company | CRM Integration                                                                                                      |
| Add Company to CRM      | Airtable            | Adds new company to Airtable CRM        | Is New Company?              | Process Each Company        | CRM Integration                                                                                                      |
| Sticky Note             | Sticky Note         | Documentation note                     | None                         | None                       | LinkedIn Company Search: overview and usage tips                                                                    |
| Sticky Note1            | Sticky Note         | Documentation note                     | None                         | None                       | Company Data Processing: overview and batch processing details                                                      |
| Sticky Note2            | Sticky Note         | Documentation note                     | None                         | None                       | CRM Integration: overview and duplicate prevention                                                                  |
| Sticky Note3            | Sticky Note         | Documentation note                     | None                         | None                       | Introduction and author contact                                                                                      |
| Sticky Note4            | Sticky Note         | Documentation note                     | None                         | None                       | Setup instructions including API keys and Airtable base setup                                                       |
| Sticky Note5            | Sticky Note         | Documentation note                     | None                         | None                       | Tools and external links for API and CRM                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Type: Manual Trigger  
   - No parameters.

2. **Create Set Node**  
   - Name: "Set Variables"  
   - Type: Set  
   - Add string fields:  
     - "Your target" = "Growth Marketing Agency"  
     - "B: 1-10 employees, C: 11-50 employees, D: 51-200 employees, E: 201-500 employees, F: 501-1000 employees, G: 1001-5000 employees, H: 5001-10,000 employees, I: 10,001+ employees" = "C"  
     - "Location (find it on : https://www.ghostgenius.fr/tools/search-sales-navigator-locations-id)" = "103644278"  
   - Connect "When clicking ‘Test workflow’" → "Set Variables".

3. **Create HTTP Request Node for Company Search**  
   - Name: "Search Companies"  
   - Type: HTTP Request  
   - URL: `https://api.ghostgenius.fr/v2/search/companies`  
   - Authentication: HTTP Header Auth with Bearer token (Ghost Genius API key)  
   - Query Parameters:  
     - keywords = `={{ $json['Your target'] }}`  
     - company_size = `={{ $json['B: 1-10 employees, C: 11-50 employees, D: 51-200 employees, E: 201-500 employees, F: 501-1000 employees, G: 1001-5000 employees, H: 5001-10,000 employees, I: 10,001+ employees'] }}`  
     - location = `={{ $json['Location (find it on : https://www.ghostgenius.fr/tools/search-sales-navigator-locations-id)'] }}`  
   - Enable Pagination:  
     - Parameter: page = `{{$pageCount + 1}}`  
     - Request interval: 2000 ms  
     - Limit pages fetched: true  
     - Complete expression: `={{ $response.body.data.isEmpty() }}`  
   - Connect "Set Variables" → "Search Companies".

4. **Create Split Out Node**  
   - Name: "Extract Company Data"  
   - Type: Split Out  
   - Field to split out: "data"  
   - Connect "Search Companies" → "Extract Company Data".

5. **Create Split In Batches Node**  
   - Name: "Process Each Company"  
   - Type: Split In Batches  
   - Batch size: 1 (default)  
   - Connect "Extract Company Data" → "Process Each Company".

6. **Create HTTP Request Node for Company Info**  
   - Name: "Get Company Info"  
   - Type: HTTP Request  
   - URL: `https://api.ghostgenius.fr/v2/company`  
   - Query Parameter: url = `={{ $json.url }}`  
   - Authentication: HTTP Header Auth with Bearer token (Ghost Genius API key)  
   - Enable batching: 1 request per 2000 ms  
   - Retry on failure: enabled  
   - Connect "Process Each Company" → "Get Company Info".

7. **Create If Node for Filtering Companies**  
   - Name: "Filter Valid Companies"  
   - Type: If  
   - Conditions (AND):  
     - Website field is not empty (`{{$json.website}}` not empty)  
     - Followers count > 200 (`{{$json.followers_count}} > 200`)  
   - Connect "Get Company Info" → "Filter Valid Companies".

8. **Create Airtable Node to Check for Existing Company**  
   - Name: "Check If Company Exists"  
   - Type: Airtable  
   - Operation: Search  
   - Base: Select your Airtable base "CRM"  
   - Table: Select your table "CRM"  
   - Filter formula: `={id} = '{{ $json.id.toNumber() }}'`  
   - Credentials: Airtable Personal Access Token  
   - Connect "Filter Valid Companies" (true output) → "Check If Company Exists".

9. **Create If Node to Determine New Company**  
   - Name: "Is New Company?"  
   - Type: If  
   - Condition: Check if first item in Airtable search result is empty (`{{ $('Check If Company Exists').all().first().json }}` is empty)  
   - Connect "Check If Company Exists" → "Is New Company?".

10. **Create Airtable Node to Add New Company**  
    - Name: "Add Company to CRM"  
    - Type: Airtable  
    - Operation: Create  
    - Base: "CRM"  
    - Table: "CRM"  
    - Map fields:  
      - id: `={{ $('Filter Valid Companies').item.json.id.toNumber() }}`  
      - Name: `={{ $('Filter Valid Companies').item.json.name }}`  
      - Country: "🇺🇸 United States" (customize as needed)  
      - Summary: `={{ $('Filter Valid Companies').item.json.description }}`  
      - Tagline: `={{ $('Filter Valid Companies').item.json.tagline }}`  
      - Website: `={{ $('Filter Valid Companies').item.json.website }}`  
      - Category: "Growth Marketing Agency 11-50 🌍" (customize as needed)  
      - LinkedIn: `={{ $('Filter Valid Companies').item.json.url }}`  
    - Credentials: Airtable Personal Access Token  
    - Connect "Is New Company?" (true output) → "Add Company to CRM".

11. **Loop Back Connections**  
    - Connect "Add Company to CRM" → "Process Each Company" (to continue processing next company)  
    - Connect "Is New Company?" (false output) → "Process Each Company" (skip existing company)  
    - Connect "Filter Valid Companies" (false output) → "Process Each Company" (skip invalid company)

12. **Add Sticky Notes** (Optional but recommended for documentation)  
    - Add notes for each major block explaining purpose, usage tips, and setup instructions as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This automation is very easy to implement and designed for anyone wanting to build and enrich a solid CRM through LinkedIn research. The initial data source can be changed as long as you have the LinkedIn URL of the company. | Introduction sticky note with author contact: [LinkedIn](https://www.linkedin.com/in/matthieu-belin83/) |
| Create an account on Ghost Genius API and get your API key. Configure HTTP Header Auth credentials with "Authorization" and "Bearer api_key". Create Airtable base "CRM" with required columns and set up Airtable credentials. | Setup sticky note                                                                                         |
| Use Ghost Genius API for LinkedIn company search and details. Use Airtable as CRM. Use Ghost Genius Locations ID Finder to find LinkedIn location IDs for precise targeting.                                                    | Tools sticky note                                                                                         |
| The workflow limits API calls by batching requests with 2-second intervals to avoid rate limiting and timeouts. Adjust follower count threshold and search parameters to fit your target market.                                | Company Data Processing sticky note                                                                      |
| The workflow prevents duplicates by checking Airtable for existing company IDs before adding new records. Customize Airtable fields and categories to match your CRM schema.                                                  | CRM Integration sticky note                                                                               |
| Maximum 1000 companies per search (100 LinkedIn pages). Refine search criteria to avoid losing prospects.                                                                                                                     | LinkedIn Company Search sticky note                                                                       |

---

This documentation provides a complete, structured understanding of the "Search LinkedIn companies and add them to Airtable CRM" workflow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the automation effectively.