Find internal linking opportunities with SerpAPI and Google Sheets

https://n8nworkflows.xyz/workflows/find-internal-linking-opportunities-with-serpapi-and-google-sheets-9071


# Find internal linking opportunities with SerpAPI and Google Sheets

### 1. Workflow Overview

This workflow is designed to identify internal linking opportunities for a website by leveraging SerpAPI search results and Google Sheets for data storage and management. It targets SEO professionals or site managers who want to improve their site’s internal linking structure to enhance search engine understanding and ranking.

**Logical blocks:**

- **1.1 Input Reception:** Retrieve URLs and their associated keywords from a Google Sheet, filtering to include only those that have not yet been processed for internal link suggestions.
- **1.2 Batch Processing Loop:** Process URLs in manageable batches to avoid API rate limits.
- **1.3 SerpAPI Search Query:** For each URL and keyword, perform a site-restricted search excluding the URL itself to find related pages on the same domain.
- **1.4 Results Extraction:** Extract organic result URLs from the SerpAPI response.
- **1.5 Google Sheets Update:** Write the suggested internal link URLs back into the Google Sheet, marking missing results as "N/A."

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block retrieves the URLs and their target keywords from a Google Sheet. It filters the rows to exclude those already processed (i.e., those with non-empty "Internal link 1" field), ensuring the workflow only works on new or unprocessed entries.

- **Nodes Involved:**  
  - Manual trigger  
  - Get URLs and keywords  
  - Sticky Note (explanatory)

- **Node Details:**

  - **Manual trigger**  
    - Type: Trigger  
    - Role: Initiates the workflow manually.  
    - Configuration: Default manual trigger with no parameters.  
    - Connections: Outputs to "Get URLs and keywords."  
    - Edge cases: None significant; user must trigger manually.

  - **Get URLs and keywords**  
    - Type: Google Sheets node (read)  
    - Role: Reads data from Google Sheet named "Internal links" in the spreadsheet "Find internal links using SerpAPI - N8N demo" (document ID `1U0whmy1d7wPoWTNTa1u68QrLTE1NqYgyyFXxARWo_1M`).  
    - Configuration: Reads from sheet with gid=0, filters rows where "Internal link 1" is empty, to avoid overwriting existing data. Retrieves columns including Domain, URL, Keyword, and internal links columns (though only URL and Keyword are used downstream).  
    - Key expressions: Filters applied on "Internal link 1."  
    - Connections: Outputs to "Loop Over Items" node.  
    - Edge cases:  
      - Authentication errors if Google Sheets credentials are invalid.  
      - Filtering might miss rows if column names change or data is malformed.  
      - Empty or missing columns could cause no data to process.

  - **Sticky Note (Get URLs and keywords)**  
    - Provides explanation about the purpose of the "Get URLs and keywords" node and the filtering logic.  
    - No technical function.

---

#### 2.2 Batch Processing Loop

- **Overview:**  
Processes the URLs in batches of 5 to avoid breaching API rate limits for SerpAPI and Google Sheets. This ensures controlled and efficient processing.

- **Nodes Involved:**  
  - Loop Over Items  
  - Sticky Note (Loop over URLs)

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Splits the input data into batches of 5 items each for sequential processing.  
    - Configuration: Batch size set to 5. No additional options set.  
    - Input: Receives JSON array of URLs and keywords.  
    - Output: Passes batches one by one downstream.  
    - Connections: On batch completion, loops back for next batch; main output connects to "Get search results using SerpAPI."  
    - Edge cases:  
      - Large batch sizes risk API rate limits or timeouts.  
      - Very small batches can slow processing.  
      - If input is empty, node outputs nothing, effectively ending processing.  

  - **Sticky Note (Loop over URLs)**  
    - Explains the batch processing rationale and warns about rate limits.  
    - Suggests adjusting batch size or adding wait nodes for larger datasets.

---

#### 2.3 SerpAPI Search Query

- **Overview:**  
For each URL and keyword batch, this node queries SerpAPI to find related pages on the same domain, excluding the current URL, to identify potential internal link candidates.

- **Nodes Involved:**  
  - Get search results using SerpAPI  
  - Sticky Note (Get search results using SerpAPI)

- **Node Details:**

  - **Get search results using SerpAPI**  
    - Type: HTTP Request  
    - Role: Calls SerpAPI's search endpoint with constructed query parameters.  
    - Configuration:  
      - URL: `https://serpapi.com/search`  
      - Authentication: Uses predefined SerpAPI credentials stored in n8n (`serpApi` credential).  
      - Query parameters:  
        - `q` parameter dynamically constructed as:  
          `{{ $json.Keyword }} site:{{ $json.Domain }} -inurl:{{ $json.URL }}`  
        - This performs a keyword search restricted to the domain, excluding the current URL to avoid self-references.  
    - Input: Receives one item per batch from "Loop Over Items" with fields: Keyword, Domain, URL.  
    - Output: JSON response from SerpAPI including organic search results and other metadata.  
    - Connections: Outputs to "Extract links from JSON."  
    - Edge cases:  
      - API key invalid or rate limited by SerpAPI.  
      - Malformed queries if input fields are missing or invalid.  
      - Network or request timeout errors.  
      - If no results, organic_results array may be empty or missing.  
    - Version-specific notes: Uses n8n HTTP Request node version 4.2 with OAuth-like credential support.

  - **Sticky Note (Get search results using SerpAPI)**  
    - Explains the logic of the query construction and the purpose of excluding the current URL.  
    - Notes a recommendation to use Google Programmable Search Engine for large data sets.  
    - Provides a link: https://drlee.io/build-your-own-google-create-a-custom-search-engine-with-trusted-sources-c1c113e845cc

---

#### 2.4 Results Extraction

- **Overview:**  
Simplifies the SerpAPI response by extracting only the organic result URLs into an array, along with the original URL.

- **Nodes Involved:**  
  - Extract links from JSON  
  - Sticky Note (Extract organic links from JSON)

- **Node Details:**

  - **Extract links from JSON**  
    - Type: Set node  
    - Role: Transforms the SerpAPI JSON response to a simplified format.  
    - Configuration:  
      - Assigns two fields:  
        - `URL`: Carries forward the original URL from input (`$('Loop Over Items').item.json.URL`).  
        - `organic_results`: Array of strings, extracted by mapping over the SerpAPI `organic_results` array and selecting their `link` property: `{{ $json.organic_results.map(item => item.link) }}`  
      - Removes all other fields from output.  
    - Input: JSON from SerpAPI node, which contains potentially many fields.  
    - Output: JSON with only URL and organic_results array.  
    - Connections: Outputs to "Add internal URL to Google Sheet."  
    - Edge cases:  
      - If `organic_results` is missing or empty, `organic_results` will be an empty array.  
      - If SerpAPI JSON structure changes, `map` expression could fail.  
      - If input URL is missing, `URL` field will be empty.  

  - **Sticky Note (Extract organic links from JSON)**  
    - Describes that only organic results are extracted from the SerpAPI data, ignoring irrelevant data.  

---

#### 2.5 Google Sheets Update

- **Overview:**  
Updates the original Google Sheet with the internal linking suggestions by writing the extracted URLs into columns "Internal link 1" through "Internal link 10" for each processed URL. If fewer than 10 suggestions exist, fills remaining columns with "N/A."

- **Nodes Involved:**  
  - Add internal URL to Google Sheet  
  - Sticky Note (Update Google Sheet)

- **Node Details:**

  - **Add internal URL to Google Sheet**  
    - Type: Google Sheets node (update)  
    - Role: Updates existing rows in the Google Sheet with internal linking suggestions.  
    - Configuration:  
      - Document ID and Sheet Name same as "Get URLs and keywords."  
      - Operation: Update existing rows matched by "URL" column.  
      - Mapping:  
        - `URL` column updated with current URL.  
        - `Internal link 1` through `Internal link 10` columns set to the corresponding elements of `organic_results` array or "N/A" if the element is undefined.  
        - Uses inline expressions with ternary operators, e.g.:  
          `={{ $json.organic_results[0] ? $json.organic_results[0] : "N/A" }}`  
      - Matching column to find row: "URL"  
      - Converts fields as strings; no type conversion.  
    - Input: JSON with `URL` and `organic_results` array from the previous node.  
    - Output: Connects back to "Loop Over Items" to continue batch processing.  
    - Edge cases:  
      - Google Sheets API errors: authentication, rate limits, or quota exceeded.  
      - Row matching fails if URLs do not exactly match or if URL column is missing.  
      - If sheet structure changes (column names or order), mapping may fail.  
      - If no results, all internal links set to "N/A."  
    - Credentials: Uses Google Sheets OAuth2 credentials with personal Google account.  

  - **Sticky Note (Update Google Sheet)**  
    - Notes the use of inline if-statements to safely handle missing URLs.  
    - Explains that not all URLs will have 10 suggestions.

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                          | Input Node(s)        | Output Node(s)             | Sticky Note                                                                                                      |
|-----------------------------|------------------------|----------------------------------------|----------------------|----------------------------|------------------------------------------------------------------------------------------------------------------|
| Manual trigger              | Manual Trigger          | Initiates workflow manually             |                      | Get URLs and keywords       |                                                                                                                  |
| Get URLs and keywords       | Google Sheets           | Reads URLs and keywords from Google Sheet, filtering processed rows | Manual trigger        | Loop Over Items             | Get URLs and keywords: retrieves URLs and keywords, filters rows with filled 'Internal link 1'                   |
| Loop Over Items             | SplitInBatches          | Processes URLs in batches of 5           | Get URLs and keywords | Get search results using SerpAPI | Loop over URLs: batch size 5 to avoid API rate limits; adjust batch size or add wait nodes for large sets      |
| Get search results using SerpAPI | HTTP Request         | Queries SerpAPI for related pages on site excluding current URL | Loop Over Items       | Extract links from JSON     | Get search results using SerpAPI: search with `site:` and `-inurl:`; link to Programmable Search Engine guide    |
| Extract links from JSON     | Set                     | Extracts organic result URLs from SerpAPI JSON | Get search results using SerpAPI | Add internal URL to Google Sheet | Extract organic links from JSON: filters out unnecessary data, keeps only URLs                                   |
| Add internal URL to Google Sheet | Google Sheets        | Updates Google Sheet with internal link suggestions | Extract links from JSON | Loop Over Items             | Update Google Sheet: writes up to 10 internal links or 'N/A' if missing                                          |
| Sticky Note                | Sticky Note             | Documentation/explanation                |                      |                            | Overview: explains workflow purpose, SEO use case, setup steps                                                  |
| Sticky Note1               | Sticky Note             | Documentation/explanation                |                      |                            | Loop over URLs: batch size and rate limit warnings                                                              |
| Sticky Note2               | Sticky Note             | Documentation/explanation                |                      |                            | SerpAPI search explanation, advanced setup suggestion                                                          |
| Sticky Note3               | Sticky Note             | Documentation/explanation                |                      |                            | Organic link extraction explanation                                                                              |
| Sticky Note4               | Sticky Note             | Documentation/explanation                |                      |                            | Google Sheet update explanation                                                                                  |
| Sticky Note5               | Sticky Note             | Documentation/explanation                |                      |                            | Full workflow overview and setup instructions                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger node**  
   - Type: Manual Trigger  
   - No parameters needed  
   - This node will start the workflow manually.

2. **Add "Get URLs and keywords" Google Sheets node**  
   - Type: Google Sheets (Read)  
   - Set credentials to your Google Sheets OAuth2 account.  
   - Document ID: Use your target spreadsheet ID.  
   - Sheet Name: Use the sheet with target URLs and keywords (e.g., `gid=0`).  
   - Operation: Read rows.  
   - Add a filter to exclude rows where "Internal link 1" column is not empty, so only unprocessed rows are retrieved.

3. **Add "Loop Over Items" node (SplitInBatches)**  
   - Type: SplitInBatches  
   - Batch Size: 5 (adjustable based on API limits).  
   - Connect "Get URLs and keywords" node's main output to this node's input.

4. **Add "Get search results using SerpAPI" HTTP Request node**  
   - Type: HTTP Request  
   - Credentials: Configure with your SerpAPI API key credential.  
   - URL: `https://serpapi.com/search`  
   - HTTP Method: GET  
   - Query Parameters:  
     - `q` = `={{ $json.Keyword }} site:{{ $json.Domain }} -inurl:{{ $json.URL }}`  
   - Connect the output of "Loop Over Items" node to this node.

5. **Add "Extract links from JSON" Set node**  
   - Type: Set node  
   - Add two fields:  
     - `URL` (string): `={{ $('Loop Over Items').item.json.URL }}`  
     - `organic_results` (array): `={{ $json.organic_results.map(item => item.link) }}`  
   - Remove other fields.  
   - Connect from "Get search results using SerpAPI" node.

6. **Add "Add internal URL to Google Sheet" node**  
   - Type: Google Sheets (Update)  
   - Credentials: Use the same Google Sheets OAuth2 credentials.  
   - Document ID and Sheet Name: same as "Get URLs and keywords."  
   - Operation: Update row.  
   - Matching Column: "URL"  
   - Map columns:  
     - `URL` to `={{ $json.URL }}`  
     - `Internal link 1` to `={{ $json.organic_results[0] ? $json.organic_results[0] : "N/A" }}`  
     - ... similarly for `Internal link 2` through `Internal link 10` with index 1 to 9, defaulting to "N/A" if missing.  
   - Connect the output of "Extract links from JSON" node here.

7. **Connect "Add internal URL to Google Sheet" node output back to "Loop Over Items" node input**  
   - This forms a loop to process all batches sequentially.

8. **Set workflow execution order to "v1" for proper batch looping.**

9. **Optional: Add Sticky Notes for documentation at each major point to explain purpose and usage.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                                            |
|-------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| This workflow is intended to improve internal linking for SEO by suggesting related pages on the same domain.                                   | Workflow purpose                                                                                                           |
| For large URL sets, consider lowering batch size or adding wait nodes to reduce rate limit errors.                                              | Batch processing best practice                                                                                             |
| If SerpAPI limits are restrictive, consider using a Google Programmable Search Engine as an alternative.                                         | https://drlee.io/build-your-own-google-create-a-custom-search-engine-with-trusted-sources-c1c113e845cc                      |
| Setup requires enabling Google Sheets API, creating a sheet with your domain, URLs, and keywords, plus configuring SerpAPI and Google creds.    | Setup instructions                                                                                                         |

---

This completes a comprehensive reference document for the "Find internal linking opportunities with SerpAPI and Google Sheets" n8n workflow. It is designed to be actionable for both advanced users and AI agents to understand, reproduce, or modify.