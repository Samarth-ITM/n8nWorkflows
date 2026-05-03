Generating New Keywords and their Search Volumes using the Google Ads API

https://n8nworkflows.xyz/workflows/generating-new-keywords-and-their-search-volumes-using-the-google-ads-api-2695


# Generating New Keywords and their Search Volumes using the Google Ads API

### 1. Workflow Overview

This workflow is designed to generate new SEO keywords along with their monthly search volumes by leveraging the Google Ads API. It is targeted at marketers, SEO specialists, and advertisers who want to expand their keyword lists for SEO or Google Ads campaigns with accurate search volume data.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Accepts an array of seed keywords to process.
- **1.2 Keyword Generation via Google Ads API:** Sends the seed keywords to Google Ads API to generate keyword ideas with metrics.
- **1.3 Data Transformation:** Splits and formats the API response to extract relevant keyword metrics.
- **1.4 Data Storage:** Upserts the processed keyword data into a Google Sheets spreadsheet for record-keeping and further analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives the initial set of keywords to be expanded and analyzed. It prepares the data for the API call by mapping the input keywords into the expected format.

- **Nodes Involved:**  
  - Trigger  
  - Set Keywords

- **Node Details:**

  - **Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for the workflow; can be replaced by any trigger (e.g., webhook, manual trigger).  
    - Configuration: Default empty trigger node.  
    - Inputs: None (start node)  
    - Outputs: Connects to "Set Keywords" node.  
    - Edge Cases: None specific; ensure trigger is properly configured in production.

  - **Set Keywords**  
    - Type: Set  
    - Role: Maps incoming data to a field named `Keyword` expected by the Google Ads API node.  
    - Configuration: Assigns `Keyword` from the input JSON path `$json.query.Keyword` (an array of keywords).  
    - Inputs: From Trigger  
    - Outputs: To "Generate new keywords" HTTP Request node.  
    - Edge Cases: Input data must contain `query.Keyword` as an array; otherwise, the API call will fail or return no results.

---

#### 2.2 Keyword Generation via Google Ads API

- **Overview:**  
  This block calls the Google Ads API to generate keyword ideas based on the input keywords, including metrics such as competition and average monthly searches.

- **Nodes Involved:**  
  - Generate new keywords

- **Node Details:**

  - **Generate new keywords**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Google Ads API endpoint `/v18/customers/{customer-id}:generateKeywordIdeas` to fetch keyword ideas and metrics.  
    - Configuration:  
      - URL: `https://googleads.googleapis.com/v18/customers/{customer-id}:generateKeywordIdeas` (replace `{customer-id}` with actual customer ID).  
      - Method: POST  
      - Headers:  
        - `content-type: application/json`  
        - `developer-token: {developer-token}` (replace with actual token)  
        - `login-customer-id: {login-customer-id}` (replace with actual login customer ID)  
      - Body (JSON): Includes geo-targeting, language, page size, and the keyword seed array mapped dynamically from the previous node (`{{ $json.Keyword }}`).  
      - Authentication: Uses Google Ads OAuth2 credentials configured in n8n.  
      - Retry on Fail: Enabled to handle transient errors.  
    - Inputs: From "Set Keywords" node  
    - Outputs: To "Split Out" node  
    - Edge Cases / Failures:  
      - Authentication errors if credentials or tokens are invalid.  
      - API quota limits or rate limiting.  
      - Incorrect customer ID or developer token causing authorization failures.  
      - Empty or malformed keyword array causing no results.  
    - Notes: Requires proper setup of Google Ads API credentials and correct URL parameters as per setup instructions.

---

#### 2.3 Data Transformation

- **Overview:**  
  This block processes the raw API response by splitting the array of keyword results and extracting relevant fields into a structured format.

- **Nodes Involved:**  
  - Split Out  
  - Edit Fields

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the `results` array from the API response into individual items for further processing.  
    - Configuration: Field to split out is `results`.  
    - Inputs: From "Generate new keywords" node  
    - Outputs: To "Edit Fields" node  
    - Edge Cases: If `results` is empty or missing, downstream nodes will receive no data.

  - **Edit Fields**  
    - Type: Set  
    - Role: Maps and formats the keyword idea metrics into individual fields for easier consumption and storage.  
    - Configuration:  
      - Extracts and renames fields such as:  
        - `keyword` from `$json.text`  
        - `competition` from `$json.keywordIdeaMetrics.competition`  
        - `avgMonthlySearches` from `$json.keywordIdeaMetrics.avgMonthlySearches`  
        - `competitionIndex` from `$json.keywordIdeaMetrics.competitionIndex`  
        - `lowTopOfPageBidMicros` and `highTopOfPageBidMicros` converted from micros to standard currency units (divided by 1,000,000 and fixed to 2 decimals).  
    - Inputs: From "Split Out" node  
    - Outputs: To "Upsert" node  
    - Edge Cases: Missing or null metric fields may cause empty or incorrect values; ensure API response is valid.

---

#### 2.4 Data Storage

- **Overview:**  
  This block appends the processed keyword data into a Google Sheets spreadsheet for record-keeping and further analysis.

- **Nodes Involved:**  
  - Upsert

- **Node Details:**

  - **Upsert**  
    - Type: Google Sheets  
    - Role: Appends new keyword data rows into a specified Google Sheets document and sheet.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Points to a specific Google Sheets document (SEO DOCTOR: Keyword Tool).  
      - Sheet Name: "Keyword" (sheet ID 1475484115)  
      - Columns: Automatically mapped from input data fields such as `keyword`, `competition`, `avgMonthlySearches`, `competitionIndex`, `lowTopOfPageBidMicros`, `highTopOfPageBidMicros`, and others.  
      - Credentials: Uses Google Sheets OAuth2 credentials configured in n8n.  
    - Inputs: From "Edit Fields" node  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - Google Sheets API quota limits or permission errors.  
      - Mismatched column names or schema changes in the spreadsheet.  
      - Network or authentication failures.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                          | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                          |
|--------------------|---------------------|----------------------------------------|---------------------|------------------------|----------------------------------------------------------------------------------------------------|
| Trigger            | Execute Workflow Trigger | Entry point for workflow                | None                | Set Keywords            | Set up a new trigger and map the data with a column name as keyword                                |
| Set Keywords       | Set                 | Maps input keywords to `Keyword` field | Trigger             | Generate new keywords   | Set up a new trigger and map the data with a column name as keyword                                |
| Generate new keywords | HTTP Request       | Calls Google Ads API to generate keywords and metrics | Set Keywords        | Split Out              | Call the endpoint: https://googleads.googleapis.com/v18/customers/{customer_id}:generateKeywordIdeas. Replace placeholders with your values. |
| Split Out          | Split Out            | Splits API response array into individual keyword results | Generate new keywords | Edit Fields            | Generate new keywords and split the data out to set only the keyword and average monthly search    |
| Edit Fields        | Set                  | Extracts and formats keyword metrics   | Split Out           | Upsert                 | Generate new keywords and split the data out to set only the keyword and average monthly search    |
| Upsert             | Google Sheets        | Appends processed keyword data to Google Sheets | Edit Fields         | None                   | Upsert the new keywords to sheets                                                                 |
| Sticky Note        | Sticky Note          | Documentation and usage notes           | None                | None                   | Generate new keywords for SEO with the monthly Search volumes. Workflow improvements and usage instructions. |
| Sticky Note1       | Sticky Note          | Setup instructions                      | None                | None                   | Setup instructions for Google Ads API URL, headers, and JSON body format                           |
| Sticky Note2       | Sticky Note          | Troubleshooting tips                    | None                | None                   | Troubleshooting tips for credentials, customer ID, and developer token                             |
| Sticky Note3       | Sticky Note          | Block description                      | None                | None                   | Generate new keywords and split the data out to set only the keyword and average monthly search    |
| Sticky Note4       | Sticky Note          | Block description                      | None                | None                   | Set up a new trigger and map the data with a column name as keyword                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Trigger Node:**  
   - Add an "Execute Workflow Trigger" node (or replace with your preferred trigger such as Webhook or Manual Trigger).  
   - No special configuration needed.

2. **Add a Set Node named "Set Keywords":**  
   - Purpose: Map incoming data to a field named `Keyword`.  
   - Configuration:  
     - Add an assignment:  
       - Name: `Keyword`  
       - Type: String  
       - Value: `={{ $json.query.Keyword }}` (expects an array of keywords under `query.Keyword` in the input JSON).  
   - Connect Trigger's output to this node's input.

3. **Add an HTTP Request Node named "Generate new keywords":**  
   - Purpose: Call Google Ads API to generate keyword ideas.  
   - Configuration:  
     - HTTP Method: POST  
     - URL: `https://googleads.googleapis.com/v18/customers/{customer-id}:generateKeywordIdeas` (replace `{customer-id}` with your actual Google Ads customer ID).  
     - Authentication: Use Google Ads OAuth2 credentials (set up in n8n beforehand).  
     - Headers:  
       - `content-type: application/json`  
       - `developer-token: {your-developer-token}`  
       - `login-customer-id: {your-login-customer-id}`  
     - Body Content Type: JSON  
     - JSON Body:  
       ```json
       {
         "geoTargetConstants": ["geoTargetConstants/2840"],
         "includeAdultKeywords": false,
         "pageToken": "",
         "pageSize": 2,
         "keywordPlanNetwork": "GOOGLE_SEARCH",
         "language": "languageConstants/1000",
         "keywordSeed": {
           "keywords": {{ $json.Keyword }}
         }
       }
       ```  
     - Enable "Send Body" and "Send Headers".  
     - Enable "Retry on Fail" to handle transient errors.  
   - Connect "Set Keywords" node output to this node input.

4. **Add a Split Out Node named "Split Out":**  
   - Purpose: Split the `results` array from the API response into individual items.  
   - Configuration:  
     - Field to split out: `results`  
   - Connect "Generate new keywords" node output to this node input.

5. **Add a Set Node named "Edit Fields":**  
   - Purpose: Extract and format keyword metrics for storage.  
   - Configuration: Add the following assignments:  
     - `keyword`: `={{ $json.text }}`  
     - `competition`: `={{ $json.keywordIdeaMetrics.competition }}`  
     - `avgMonthlySearches`: `={{ $json.keywordIdeaMetrics.avgMonthlySearches }}`  
     - `competitionIndex`: `={{ $json.keywordIdeaMetrics.competitionIndex }}`  
     - `lowTopOfPageBidMicros`: `={{ ($json["keywordIdeaMetrics"].lowTopOfPageBidMicros / 1000000).toFixed(2) }}`  
     - `highTopOfPageBidMicros`: `={{ ($json.keywordIdeaMetrics.highTopOfPageBidMicros / 1000000).toFixed(2) }}`  
   - Connect "Split Out" node output to this node input.

6. **Add a Google Sheets Node named "Upsert":**  
   - Purpose: Append the processed keyword data to a Google Sheets document.  
   - Configuration:  
     - Operation: Append  
     - Document ID: Use your Google Sheets document ID (e.g., `10mXXLB987b7UySHtS9F4EilxeqbQjTkLOfMabnR2i5s`)  
     - Sheet Name: "Keyword" (or your target sheet)  
     - Columns: Map columns to the fields from "Edit Fields" node (e.g., `keyword`, `competition`, `avgMonthlySearches`, `competitionIndex`, `lowTopOfPageBidMicros`, `highTopOfPageBidMicros`, etc.)  
     - Credentials: Use Google Sheets OAuth2 credentials configured in n8n.  
   - Connect "Edit Fields" node output to this node input.

7. **Save and Activate the Workflow:**  
   - Test with sample input data containing an array of keywords under `query.Keyword`.  
   - Monitor for errors and adjust credentials or parameters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow is an improvement on previous workflows for generating SEO keywords with search volume data.     | [Generate SEO Keyword Search Volume Data using Google API](https://n8n.io/workflows/2494-generate-seo-keyword-search-volume-data-using-google-api/) and [Generating Keywords using Google Autosuggest](https://n8n.io/workflows/2155-generating-keywords-using-google-autosuggest/) |
| For detailed setup and troubleshooting of Google Ads API credentials, refer to the blog post by the author.   | [Automating Keyword Generation with n8n & Google Ads API](https://funautomations.io/workflows/automating-keyword-generation-with-n8n-google-ads-api/) |
| Make a copy of the provided Google Sheets template to store and manage your keyword data.                      | [Google Sheets Template](https://docs.google.com/spreadsheets/d/10mXXLB987b7UySHtS9F4EilxeqbQjTkLOfMabnR2i5s/edit?usp=sharing) |
| Replace placeholders `{customer-id}`, `{developer-token}`, and `{login-customer-id}` in the HTTP Request node with your actual Google Ads account values. | Setup instructions in Sticky Note1 within the workflow.                                                     |
| Workflow created by [@Imperol](https://www.linkedin.com/in/zacharia-kimotho/)                                  | Author credit                                                                                               |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting for advanced users and automation agents alike.