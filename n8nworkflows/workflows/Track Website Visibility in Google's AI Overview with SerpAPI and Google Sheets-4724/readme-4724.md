Track Website Visibility in Google's AI Overview with SerpAPI and Google Sheets

https://n8nworkflows.xyz/workflows/track-website-visibility-in-google-s-ai-overview-with-serpapi-and-google-sheets-4724


# Track Website Visibility in Google's AI Overview with SerpAPI and Google Sheets

### 1. Workflow Overview

This workflow automates the tracking of website visibility within Google's AI Overview feature for SEO purposes. It is designed to read a list of SEO keywords from a Google Sheet, query SerpApi’s Google Search API for AI Overview data related to each keyword, analyze the AI Overview to extract referenced sources, check whether the user’s own domain appears among those sources, and then record these results back into another Google Sheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Keyword Acquisition:** Reading SEO keywords from a Google Sheet.
- **1.3 AI Overview Data Retrieval:** Querying SerpApi for AI Overview data for each keyword.
- **1.4 Data Processing & Domain Check:** Extracting cited sources from API responses and verifying domain presence.
- **1.5 Output Storage:** Writing analyzed results back to a Google Sheet.
- **1.6 Documentation:** A sticky note summarizing workflow steps for clarity.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow manually.
- **Nodes Involved:** 
  - Start: Manual trigger

##### Node Details

- **Start: Manual trigger**
  - Type: Manual Trigger node
  - Role: Entry point to start the workflow on demand.
  - Configuration: No parameters; triggers workflow execution manually.
  - Inputs: None
  - Outputs: Connects to "Read Keywords from Google Sheet"
  - Edge cases: None; manual start prevents unintended runs.
  - Version: 1

---

#### 1.2 Keyword Acquisition

- **Overview:** Reads a list of SEO keywords from a specified Google Sheet to be processed downstream.
- **Nodes Involved:**
  - Read Keywords from Google Sheet

##### Node Details

- **Read Keywords from Google Sheet**
  - Type: Google Sheets node (v4.6)
  - Role: Fetch keyword data from a Google Sheet document.
  - Configuration:
    - DocumentId points to a specific Google Sheet containing keywords.
    - Sheet is the first tab (gid=0).
    - Operation defaults to reading rows (implicit).
  - Credentials: Uses Google Sheets OAuth2 credentials.
  - Inputs: From manual trigger
  - Outputs: Sends each keyword as a separate item to the next node.
  - Expressions: None explicitly; data directly fetched from the sheet.
  - Edge cases:
    - Auth errors if Google Sheets OAuth2 token expires.
    - Empty or malformed sheet could result in no keywords or invalid data.
    - API quota limits.
  - Version: 4.6

---

#### 1.3 AI Overview Data Retrieval

- **Overview:** For each keyword, calls SerpApi’s Google Search API to retrieve AI Overview data.
- **Nodes Involved:**
  - Call SerpApi for AI Overview

##### Node Details

- **Call SerpApi for AI Overview**
  - Type: HTTP Request node (v4.2)
  - Role: Fetches Google Search results with AI Overview details via SerpApi.
  - Configuration:
    - HTTP Method: GET (default)
    - URL: `https://serpapi.com/search.json`
    - Query Parameters:
      - `q`: dynamically set to keyword from previous node (`={{ $json["keyword"] }}`).
      - `api_key`: placeholder for user's SerpApi API key (must be replaced with actual key).
  - Inputs: Keywords from Google Sheets node.
  - Outputs: JSON response containing AI Overview data per keyword.
  - Edge cases:
    - API key missing or invalid causing authentication errors.
    - Rate limiting by SerpApi.
    - Network timeouts.
    - Missing AI Overview data in response.
  - Version: 4.2

---

#### 1.4 Data Processing & Domain Check

- **Overview:** Parses the SerpApi responses to extract AI Overview references and checks if the user’s domain is cited.
- **Nodes Involved:**
  - Extract Sources & Check My Domain

##### Node Details

- **Extract Sources & Check My Domain**
  - Type: Code node (JavaScript, v2)
  - Role: Processes API response JSON to identify AI Overview presence, extracts source links, and verifies if the user’s domain is in the list.
  - Configuration:
    - Custom JavaScript code snippet.
    - User must replace `myDomain` variable with their actual domain (e.g., `"example.com"`).
  - Key expressions:
    - Uses `item.json.ai_overview.references` to get links.
    - Checks for domain presence via `.some(link => link.includes(myDomain))`.
  - Inputs: API responses from previous HTTP Request node.
  - Outputs: JSON objects with fields:
    - keyword
    - ai_overview_exists (boolean)
    - links (array of reference URLs)
    - is_my_domain_listed (boolean)
  - Edge cases:
    - AI Overview might be missing or lack references.
    - Domain check is a substring match, which may cause false positives if domain string is too generic.
    - Code errors if the JSON structure varies.
  - Version: 2

---

#### 1.5 Output Storage

- **Overview:** Appends the processed data (keyword, AI Overview existence, source links, domain presence) into a results Google Sheet.
- **Nodes Involved:**
  - Write Results to Google Sheet

##### Node Details

- **Write Results to Google Sheet**
  - Type: Google Sheets node (v4.6)
  - Role: Append processed results to a dedicated Google Sheet for record keeping.
  - Configuration:
    - DocumentId points to the results Google Sheet.
    - Sheet: first tab (gid=0).
    - Operation: append rows.
    - Columns mapped explicitly:
      - keyword
      - has_ai_overview (from ai_overview_exists)
      - links (array converted to string)
      - is_my_domain_listed
  - Credentials: Google Sheets OAuth2 credentials (same as input sheet).
  - Inputs: Processed data from the code node.
  - Outputs: None (terminal node).
  - Edge cases:
    - Authentication errors.
    - Schema mismatch if columns do not exist or data types are inconsistent.
    - Quota limits on Google Sheets API.
  - Version: 4.6

---

#### 1.6 Documentation

- **Overview:** Provides a sticky note in the editor summarizing the workflow steps.
- **Nodes Involved:**
  - Sticky Note

##### Node Details

- **Sticky Note**
  - Type: Sticky Note node (v1)
  - Role: Visual documentation inside the workflow editor.
  - Configuration: Contains a markdown summary of workflow steps.
  - Position: Separate from main node flow.
  - Inputs/Outputs: None
  - Edge cases: None
  - Version: 1

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                       | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                             |
|-------------------------------|---------------------|------------------------------------|---------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------|
| Start: Manual trigger          | Manual Trigger      | Initiates workflow execution        | None                      | Read Keywords from Google Sheet |                                                                                                       |
| Read Keywords from Google Sheet| Google Sheets       | Reads SEO keywords from Google Sheet| Start: Manual trigger     | Call SerpApi for AI Overview    |                                                                                                       |
| Call SerpApi for AI Overview   | HTTP Request        | Retrieves AI Overview data from SerpApi API| Read Keywords from Google Sheet | Extract Sources & Check My Domain |                                                                                                       |
| Extract Sources & Check My Domain | Code (JavaScript) | Parses API response, extracts references, checks domain presence| Call SerpApi for AI Overview | Write Results to Google Sheet   |                                                                                                       |
| Write Results to Google Sheet  | Google Sheets       | Appends processed results to Google Sheet| Extract Sources & Check My Domain | None                           |                                                                                                       |
| Sticky Note                   | Sticky Note         | Provides documentation summary      | None                      | None                           | ## Track Google AI Overview Presence for SEO  1. **Read** keyword list from Google Sheets  2. **Call** SerpApi for each keyword  3. **Extract** AI Overview sources from the results  4. **Check** if your domain is listed in the sources  5. **Save** the results back into Google Sheets |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**
   - Name: `Start: Manual trigger`
   - No parameters needed.
   - Position at the beginning.

2. **Add a Google Sheets node to read keywords**
   - Name: `Read Keywords from Google Sheet`
   - Operation: Read rows from Google Sheets.
   - Set Document ID to the keyword source sheet’s ID.
   - Set Sheet to the first tab (`gid=0`).
   - Configure Google Sheets OAuth2 credentials.
   - Connect output of manual trigger to this node.

3. **Add an HTTP Request node to call SerpApi**
   - Name: `Call SerpApi for AI Overview`
   - Method: GET
   - URL: `https://serpapi.com/search.json`
   - Query parameters:
     - `q` = Expression: `={{ $json["keyword"] }}`
     - `api_key` = Your SerpApi API key (replace placeholder).
   - Connect output of Google Sheets node to this node.

4. **Add a Code node to extract sources and check domain**
   - Name: `Extract Sources & Check My Domain`
   - Language: JavaScript
   - Paste the following JS code:

     ```javascript
     const myDomain = "example.com"; // Replace with your actual domain

     return items.map(item => {
       const overview = item.json.ai_overview;
       const keyword = item.json.search_parameters?.q || '';

       if (!overview || !overview.references) {
         return {
           json: {
             keyword,
             ai_overview_exists: false,
             links: [],
             is_my_domain_listed: false
           }
         };
       }

       const links = overview.references
         .filter(ref => ref.link)
         .map(ref => ref.link);

       const isMyDomainListed = links.some(link => link.includes(myDomain));

       return {
         json: {
           keyword,
           ai_overview_exists: true,
           links,
           is_my_domain_listed: isMyDomainListed
         }
       };
     });
     ```

   - Connect output of HTTP Request node to this node.

5. **Add a Google Sheets node to write results**
   - Name: `Write Results to Google Sheet`
   - Operation: Append rows.
   - Document ID: Results Google Sheet ID.
   - Sheet: first tab (`gid=0`).
   - Map columns explicitly:
     - `keyword` → `{{$json["keyword"]}}`
     - `has_ai_overview` → `{{$json["ai_overview_exists"]}}`
     - `links` → `{{$json["links"]}}` (will be stringified)
     - `is_my_domain_listed` → `{{$json["is_my_domain_listed"]}}`
   - Use same Google Sheets OAuth2 credentials.
   - Connect output of Code node to this node.

6. **Optional: Add a Sticky Note node**
   - Position away from main flow.
   - Paste workflow summary text:

     ```
     ## Track Google AI Overview Presence for SEO
     1. **Read** keyword list from Google Sheets  
     2. **Call** SerpApi for each keyword  
     3. **Extract** AI Overview sources from the results  
     4. **Check** if your domain is listed in the sources  
     5. **Save** the results back into Google Sheets
     ```

7. **Set workflow execution order as default (v1).**

8. **Test workflow manually by triggering it and monitoring logs for errors.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Replace `"example.com"` in the Code node with your actual domain to accurately detect your domain’s presence. | Important for accurate domain presence check in AI Overview references.                        |
| Ensure your SerpApi API key is valid and has sufficient quota for the number of keywords processed.       | SerpApi documentation: https://serpapi.com/                                                    |
| Google Sheets OAuth2 credentials must have read/write permissions on both source and destination sheets. | Setup instructions: https://developers.google.com/sheets/api/quickstart/js                     |
| Workflow tags include SEO, SerpApi, AI Overview, Google, Marketing to aid search and categorization.      | Useful for filtering and organizing workflows in n8n.                                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.