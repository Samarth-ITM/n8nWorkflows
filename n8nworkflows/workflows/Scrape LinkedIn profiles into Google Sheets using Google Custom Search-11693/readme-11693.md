Scrape LinkedIn profiles into Google Sheets using Google Custom Search

https://n8nworkflows.xyz/workflows/scrape-linkedin-profiles-into-google-sheets-using-google-custom-search-11693


# Scrape LinkedIn profiles into Google Sheets using Google Custom Search

---
### 1. Workflow Overview

This workflow automates the process of scraping LinkedIn profiles based on user-input criteria and saving the results into a Google Sheet. It is designed for users who want to gather LinkedIn profile data filtered by position, industry, and region without manual searching.

**Use cases:**
- Recruiters or HR professionals sourcing candidates
- Market researchers collecting professional profiles
- Sales or business development teams targeting prospects

**Logical blocks:**

- **1.1 Input Reception:** Captures search parameters via a web form.
- **1.2 Initialization:** Sets initial pagination parameters.
- **1.3 Google Custom Search Query:** Executes a Google Custom Search to find LinkedIn profiles matching criteria.
- **1.4 Data Processing:** Parses and cleans the search results, preparing profile data.
- **1.5 Data Storage:** Appends or updates the profile data into a Google Sheet.
- **1.6 Pagination Control:** Determines if more search result pages should be fetched.
- **1.7 Loop Control:** Implements waiting between requests and loops back for pagination until all results are fetched or limit reached.

---
### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Captures the search criteria (position, industry, region) from a user via a web form trigger to initiate the workflow.

**Nodes Involved:**  
- On form submission  
- Edit Fields  

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point, receives user input from a custom web form.  
  - Configuration: Form titled "form" with required fields: position, industry, region.  
  - Inputs: External HTTP request via webhook when form submitted.  
  - Outputs: Emits JSON with user inputs.  
  - Edge cases: Missing required fields block submission; webhook availability required.

- **Edit Fields**  
  - Type: Set Node  
  - Role: Initializes pagination variables.  
  - Configuration: Sets `currentStartIndex` = 1, `maxPages` = 10 for controlling search pagination.  
  - Inputs: Data from "On form submission" node.  
  - Outputs: Adds pagination control fields to JSON.  
  - Edge cases: Hardcoded maxPages; no dynamic adjustment based on API usage or user input.

---

#### 1.2 Google Custom Search Query

**Overview:**  
Executes a Google Custom Search API request to search LinkedIn profiles using the user criteria and pagination index.

**Nodes Involved:**  
- HTTP Request  

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request Node  
  - Role: Queries Google Custom Search API with search parameters.  
  - Configuration:  
    - URL: https://www.googleapis.com/customsearch/v1  
    - Authentication: HTTP Basic Auth (API key stored in credentials)  
    - Query parameters:  
      - `cx`: Custom Search Engine ID  
      - `q`: Search query combining position, industry, region, plus site:linkedin.com/in  
      - `start`: Pagination start index (dynamic, depends on run index and previous pagination node)  
  - Inputs: From "Edit Fields" for initial run or "Pagination Check" for subsequent pages  
  - Outputs: JSON response from Google API including search items and pagination data  
  - Edge cases:  
    - API key invalid or quota exceeded causes request failure.  
    - Network timeouts or rate limiting.  
    - Search query malformed if input variables missing or incorrect.  
  - Version: 4.2

---

#### 1.3 Data Processing

**Overview:**  
Processes the raw Google Custom Search response to extract, clean, and format LinkedIn profile information, and prepares pagination data for next steps.

**Nodes Involved:**  
- Code in JavaScript (processing search results)  
- Wait (delay between requests)  

**Node Details:**

- **Code in JavaScript**  
  - Type: Code Node (JS)  
  - Role: Parses search results, extracts profile name, title, link, snippet, image; cleans title text; extracts pagination info (`nextStartIndex` and `hasMoreResults`).  
  - Key expressions:  
    - Extracts `name` from title before " - " delimiter  
    - Cleans `title` by removing trailing "| LinkedIn" and redundant name prefix  
    - Checks `response.queries.nextPage` for next page index  
    - Limits paging to max 100 results (Google API limit)  
  - Inputs: Google Custom Search API response JSON  
  - Outputs: Array of structured profile objects with pagination info attached  
  - Edge cases:  
    - No search results returns a default empty item with pagination info  
    - Unexpected response structure may cause undefined errors (e.g., missing pagemap)  
  - Version: 2

- **Wait**  
  - Type: Wait Node  
  - Role: Inserts a 3-second delay between requests to avoid API rate limiting or throttling.  
  - Inputs: From previous Code node  
  - Outputs: Delayed output to next node  
  - Edge cases: Delay may not suffice if API rate limits are strict; no dynamic adjustment.

---

#### 1.4 Data Storage

**Overview:**  
Stores or updates the extracted LinkedIn profile data into a specified Google Sheet for record keeping and further use.

**Nodes Involved:**  
- Append or update row in sheet  
- Edit Fields1  

**Node Details:**

- **Append or update row in sheet**  
  - Type: Google Sheets Node  
  - Role: Appends new rows or updates existing rows in target Google Sheet based on profile name matching.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Document ID and sheet name fixed to user's target Google Sheet  
    - Mapping: profile fields mapped to sheet columns (name, position, profile links, description, image link, and searched inputs)  
    - Matching on "name" column to prevent duplicates  
  - Inputs: Processed profile data from previous nodes  
  - Outputs: Confirmation of sheet update  
  - Credential: Google Sheets OAuth2 credentials required  
  - Edge cases:  
    - Sheet schema mismatch or missing columns causes failure  
    - API quota or authorization errors  
    - Name collisions may cause unintended overwrites  

- **Edit Fields1**  
  - Type: Set Node  
  - Role: Passes pagination control parameters (`startIndex`, `hasMoreResults`) forward for loop control.  
  - Inputs: Output from Google Sheets node (preserves all fields)  
  - Outputs: Enriched JSON with pagination info for next iteration  
  - Edge cases: None significant

---

#### 1.5 Pagination Control & Loop

**Overview:**  
Controls the iteration of search requests by checking if more pages exist and looping back to fetch them until the limit or no results remain.

**Nodes Involved:**  
- Code in JavaScript1 (pagination info extraction)  
- Pagination Check (IF node)  

**Node Details:**

- **Code in JavaScript1**  
  - Type: Code Node (JS)  
  - Role: Collects all input items (from multiple search result pages), extracts the next start index and whether more results exist for pagination continuation.  
  - Key expressions: Reads `startIndex` and `hasMoreResults` from first input item.  
  - Outputs: JSON with `continueLoop` boolean and `startIndex` for next API call  
  - Edge cases: Empty inputs or missing fields handled with default values.

- **Pagination Check**  
  - Type: IF Node  
  - Role: Decides whether to continue looping based on `continueLoop` flag.  
  - Condition: Continue if `continueLoop == true`  
  - Outputs:  
    - True: loops back to "HTTP Request" node to fetch next page  
    - False: workflow ends  
  - Edge cases: Logical errors in flag setting may cause infinite loops or premature termination.

---
### 3. Summary Table

| Node Name                | Node Type               | Functional Role                      | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                   |
|--------------------------|-------------------------|------------------------------------|---------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger            | Receives search criteria input     | -                         | Edit Fields                | ## get the search data                                                                                        |
| Edit Fields              | Set                     | Initialize pagination parameters   | On form submission        | HTTP Request               |                                                                                                              |
| HTTP Request             | HTTP Request            | Query Google Custom Search API     | Edit Fields, Pagination Check | Code in JavaScript        | ## geather linkedin profile and update sheets\nsetup:\nadd custom search api key in http node               |
| Code in JavaScript       | Code (JavaScript)       | Parse & clean search results       | HTTP Request              | Wait                       |                                                                                                              |
| Wait                     | Wait                    | Pause between API requests         | Code in JavaScript        | Append or update row in sheet |                                                                                                              |
| Append or update row in sheet | Google Sheets       | Store/update profile data in sheet | Wait                      | Edit Fields1               |                                                                                                              |
| Edit Fields1             | Set                     | Pass pagination data forward       | Append or update row in sheet | Code in JavaScript1       |                                                                                                              |
| Code in JavaScript1      | Code (JavaScript)       | Extract pagination control info    | Edit Fields1              | Pagination Check           |                                                                                                              |
| Pagination Check         | IF                      | Control pagination loop decision   | Code in JavaScript1       | HTTP Request (if true)     |                                                                                                              |
| Sticky Note              | Sticky Note             | Instructional note                 | -                         | -                          | ## Scrape LinkedIn Profiles to Google Sheets\n\n### How it works\n1. You provide a position, industry, and region via a web form.\n2. The workflow performs a Google Custom Search for matching LinkedIn profiles.\n3. It extracts key profile information such as name, title, link, and description from the search results.\n4. All found profiles are appended or updated in your specified Google Sheet.\n5. The workflow automatically fetches multiple pages of results, stopping when no more are available or the Google Search API limit is reached.\n\n### Setup\n- [ ] Connect your Google Sheets account.\n- [ ] Add your Google Custom Search API Key and Search Engine ID (CX).\n- [ ] Select your Google Sheet document and the target sheet within the \"Append or update row in sheet\" node.\n- [ ] Ensure your Google Sheet has columns for: Name, Position, Profile Links, Description, Image Link, Searched Position, Searched Industry, and Searched Region.\n- [ ] Fill out the \"On form submission\" trigger form with your desired search criteria to initiate the workflow." |
| Sticky Note1             | Sticky Note             | Instructional note                 | -                         | -                          | ## get the search data                                                                                        |
| Sticky Note2             | Sticky Note             | Instructional note                 | -                         | -                          | ## geather linkedin profile and update sheets\nsetup:\nadd custom search api key in http node               |

---
### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: "On form submission"  
   - Set webhookId or leave default for auto-generated  
   - Configure form: Title "form"  
   - Add required fields: "postion", "industry", "region" (all required)  
   - Save and activate webhook

2. **Add a Set Node for Initialization**  
   - Name: "Edit Fields"  
   - Add fields:  
     - `currentStartIndex` (number) = 1  
     - `maxPages` (number) = 10  
   - Connect "On form submission" → "Edit Fields"

3. **Add HTTP Request Node to call Google Custom Search API**  
   - Name: "HTTP Request"  
   - Method: GET  
   - URL: https://www.googleapis.com/customsearch/v1  
   - Authentication: HTTP Basic Auth (create credential with your Google API Key)  
   - Query Parameters:  
     - `cx` = your Custom Search Engine ID (e.g., "c0fa13214e7ee42a4")  
     - `q` = expression: `={{ $('On form submission').item.json.postion }} {{ $('On form submission').item.json.industry }} {{ $('On form submission').item.json.region }} site:linkedin.com/in`  
     - `start` = expression: `={{ $runIndex == 0 ? $json.currentStartIndex : $node["Pagination Check"].json.startIndex }}`  
   - Connect "Edit Fields" → "HTTP Request"

4. **Add a JavaScript Code Node to Parse API Response**  
   - Name: "Code in JavaScript"  
   - Paste the provided JS code that extracts profile info, cleans titles, and sets pagination variables (`startIndex`, `hasMoreResults`)  
   - Connect "HTTP Request" → "Code in JavaScript"

5. **Add a Wait Node for Rate Limiting**  
   - Name: "Wait"  
   - Set wait time to 3 seconds  
   - Connect "Code in JavaScript" → "Wait"

6. **Add Google Sheets Node to Append or Update Data**  
   - Name: "Append or update row in sheet"  
   - Operation: appendOrUpdate  
   - Set Google Sheets OAuth2 credentials (authenticate your Google account)  
   - Document ID: your target Google Sheet ID  
   - Sheet Name: target sheet name or GID (e.g., "gid=0")  
   - Define schema columns:  
     - name, position, profile links, description, img link, searched position, searched industry, searched region  
   - Map these fields to node inputs using expressions (e.g., `={{ $json.name }}`, etc.)  
   - Match rows by "name" to avoid duplicates  
   - Connect "Wait" → "Append or update row in sheet"

7. **Add a Set Node to Pass Pagination Info**  
   - Name: "Edit Fields1"  
   - Assign fields from previous node:  
     - `startIndex` = `={{ $('Code in JavaScript').item.json.startIndex }}`  
     - `hasMoreResults` = `={{ $('Code in JavaScript').item.json.hasMoreResults }}`  
   - Include other fields intact  
   - Connect "Append or update row in sheet" → "Edit Fields1"

8. **Add JavaScript Code Node to Extract Loop Control Info**  
   - Name: "Code in JavaScript1"  
   - Copy the JS code that reads pagination flags (`startIndex`, `hasMoreResults`) and sets `continueLoop` flag  
   - Connect "Edit Fields1" → "Code in JavaScript1"

9. **Add an IF Node to Control Pagination Loop**  
   - Name: "Pagination Check"  
   - Condition: Check if `continueLoop == true` (boolean equals true)  
   - Connect "Code in JavaScript1" → "Pagination Check"  
   - True output connects back to "HTTP Request" node (to fetch next page)  
   - False output ends workflow

10. **Add Sticky Notes (Optional)**  
    - Add notes describing workflow purpose, setup instructions, and usage tips  
    - Position notes for clarity near relevant nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| The workflow relies on a Google Custom Search API key and a Custom Search Engine configured to search LinkedIn profiles (`site:linkedin.com/in`). Ensure your API key has sufficient quota and billing enabled.                                                                                                                                                                                                                                                                                   | Google Custom Search API documentation: https://developers.google.com/custom-search/v1/                                                 |
| The Google Sheet must have columns named exactly as mapped in the workflow: Name, Position, Profile Links, Description, Image Link, Searched Position, Searched Industry, and Searched Region for proper data mapping and deduplication.                                                                                                                                                                                                                                                         | Google Sheets API and OAuth2 setup in n8n: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/             |
| Pagination stops automatically when no more results are available or when the Google API limit of 100 results is reached. The workflow includes a 3-second wait between requests to mitigate rate limiting.                                                                                                                                                                                                                                                                                      | Consider adjusting wait time or adding error handling for API quota errors for robustness.                                               |
| The form trigger requires external HTTP access and a valid webhook URL; ensure your n8n instance is publicly reachable or tunneled (e.g., via ngrok) for form submissions to work.                                                                                                                                                                                                                                                                                                                  | n8n Webhook security best practices: https://docs.n8n.io/nodes/nodes-library/n8n-nodes-base.webhook/                                    |
| This workflow can be adapted for other profile scraping tasks by modifying the search query and Google Custom Search Engine configuration accordingly.                                                                                                                                                                                                                                                                                                                                           |                                                                                                                                        |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.