SERP Competitor Research with Scrape.do API & Google Sheets

https://n8nworkflows.xyz/workflows/serp-competitor-research-with-scrape-do-api---google-sheets-8917


# SERP Competitor Research with Scrape.do API & Google Sheets

---

### 1. Workflow Overview

This n8n workflow performs automated SERP (Search Engine Results Page) competitor research by retrieving keywords from a Google Sheet, querying Google Search results via the Scrape.do API, extracting competitor data from the HTML response, and appending the results back into another Google Sheet tab. It is designed for digital marketers, SEO analysts, and competitive researchers who want to monitor search rankings and competitor presence dynamically.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger and Input Fetching:** Starts the workflow manually and reads keywords plus target country codes from a Google Sheet.
- **1.2 Keyword Encoding:** URL encodes the keywords to safely embed them into search URLs.
- **1.3 SERP Data Fetching:** Calls the Scrape.do API to fetch rendered Google search results HTML for each encoded keyword and target country.
- **1.4 Data Extraction:** Parses the HTML response to extract competitor data such as titles, URLs, descriptions, and positions.
- **1.5 Output to Google Sheets:** Appends the extracted competitor data into a designated Google Sheet tab for further analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger and Input Fetching
- **Overview:**  
This block initiates the workflow manually and retrieves the list of keywords along with their target country codes from a Google Sheet.
- **Nodes Involved:**  
  - When clicking 'Execute workflow' (Manual Trigger)  
  - Get Keywords from Sheet (Google Sheets Read)  
- **Node Details:**  
  - **When clicking 'Execute workflow'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command  
    - Config: No parameters needed  
    - Inputs: None  
    - Outputs: Triggers next node  
    - Edge cases: None, manual trigger depends on user action  
  - **Get Keywords from Sheet**  
    - Type: Google Sheets (Read)  
    - Role: Reads keywords and country codes from a specific Google Sheet tab  
    - Config:  
      - Spreadsheet ID: Set to the target Google Sheet ID  
      - Sheet Name: ‘Keywords’ tab (gid=0)  
      - Credentials: Google Sheets OAuth2  
    - Key variables: Expects columns 'Keyword' and 'Target Country'  
    - Input: Trigger from manual node  
    - Output: JSON array of rows with keywords and country codes  
    - Edge cases:  
      - Missing or malformed credentials cause auth errors  
      - Empty or missing columns cause downstream failures  
      - API rate limits from Google Sheets API

#### 2.2 Keyword Encoding
- **Overview:**  
URL-encodes each keyword to ensure it is safe for inclusion in HTTP query parameters.
- **Nodes Involved:**  
  - URL Encode Keywords (Code node)  
- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Applies `encodeURIComponent` to each ‘Keyword’ field  
  - Configuration:  
    - Iterates all input items, replaces `Keyword` with encoded value  
  - Input: Rows from Google Sheets  
  - Output: Items with encoded keywords  
  - Edge cases:  
    - Keywords with special characters must be encoded properly  
    - If 'Keyword' field is missing, encoding might produce errors or blank output

#### 2.3 SERP Data Fetching
- **Overview:**  
Fetches Google search results HTML using Scrape.do API for each encoded keyword and target country.
- **Nodes Involved:**  
  - Fetch Google Search Results (HTTP Request)  
- **Node Details:**  
  - Type: HTTP Request  
  - Role: Calls Scrape.do API to scrape Google SERP  
  - Configuration:  
    - URL constructed dynamically using expressions:  
      `https://api.scrape.do/?token={{$vars.SCRAPEDO_TOKEN}}&url={{encodeURIComponent('https://www.google.com/search?q=' + $json.Keyword)}}&geoCode={{$json['Target Country'] || 'us'}}&render=true`  
    - Headers: Accept header set for HTML content  
    - Timeout: 60 seconds  
  - Input: Encoded keywords with target country  
  - Output: Raw HTML response from Scrape.do  
  - Version-specific: Requires n8n version supporting advanced HTTP node expressions (v0.153+)  
  - Edge cases:  
    - API token missing or invalid leads to 401 Unauthorized  
    - Network timeouts or rate limits on Scrape.do API  
    - Unexpected HTML structure or empty responses

#### 2.4 Data Extraction
- **Overview:**  
Parses the HTML of Google SERP to extract competitor results including title, URL, description, and position.
- **Nodes Involved:**  
  - Extract Competitor Data from HTML (Code node)  
- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Uses regex to extract organic search results from HTML  
  - Configuration:  
    - Attempts to match organic results divs with class patterns  
    - Extracts title (h3), URL (href), description (div with specific class)  
    - Limits to top 10 results per keyword  
    - Fallback to a simpler regex approach if no results found  
    - Returns structured JSON items with fields: position, websiteTitle, websiteUrl, websiteDescription, keyword  
  - Input: Raw HTML from HTTP request  
  - Output: Parsed competitor result items  
  - Edge cases:  
    - HTML structure changes may break regex parsing  
    - Google may block scraping or serve CAPTCHAs causing invalid HTML  
    - Missing title or URL results are filtered out  
    - Empty results are handled with a default “No results found” entry

#### 2.5 Output to Google Sheets
- **Overview:**  
Appends extracted competitor data into a "Competitor Results" tab in the same Google Sheet.
- **Nodes Involved:**  
  - Append Results to Sheet (Google Sheets Append)  
- **Node Details:**  
  - Type: Google Sheets (Append)  
  - Role: Inserts rows into the results sheet with competitor data  
  - Configuration:  
    - Document ID same as input sheet  
    - Sheet Name: ‘Competitor Results’ (gid=1113333162)  
    - Operation: Append rows  
    - Columns mapped automatically based on input JSON keys (keyword, position, websiteTitle, websiteUrl, websiteDescription)  
    - Credentials: Same Google Sheets OAuth2 credentials as input  
  - Input: Parsed competitor data items from code node  
  - Output: Confirmation of append operation  
  - Edge cases:  
    - Sheet tab missing causes operation to fail (must create ‘Competitor Results’ tab manually if not present)  
    - Google API rate limits or quota exceeded errors  
    - Data type mismatches if input data schema changes

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                         | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                                        |
|------------------------------|---------------------|---------------------------------------|--------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When clicking 'Execute workflow' | Manual Trigger      | Starts workflow manually               | None                           | Get Keywords from Sheet        | This workflow is triggered manually by clicking 'Execute workflow'. It fetches keywords and target countries from a Google Sheet. |
| Get Keywords from Sheet       | Google Sheets       | Reads keywords and target countries    | When clicking 'Execute workflow'| URL Encode Keywords            | Reads data from the specified Google Sheet. Expects columns: 'Keyword', 'Target Country'. Remember to replace sheet ID and cred.  |
| URL Encode Keywords           | Code                | URL-encodes keywords for safe usage    | Get Keywords from Sheet         | Fetch Google Search Results    | Encodes 'Keyword' value using encodeURIComponent to ensure proper URL formatting.                                                 |
| Fetch Google Search Results   | HTTP Request        | Calls Scrape.do API to get SERP HTML   | URL Encode Keywords             | Extract Competitor Data from HTML | Scrape.do API config: token, url, geoCode, render=true. Replace YOUR_SCRAPEDO_TOKEN. See https://dashboard.scrape.do/              |
| Extract Competitor Data from HTML | Code             | Parses HTML to extract competitor info | Fetch Google Search Results     | Append Results to Sheet        | Parses HTML response to extract titles, URLs, descriptions, and positions. Filters out ads and Google links.                      |
| Append Results to Sheet       | Google Sheets       | Appends competitor data to results tab | Extract Competitor Data from HTML | None                          | Appends parsed SERP rows to 'Results' sheet. Columns expected: keyword, position, websiteTitle, websiteUrl, websiteDescription.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named "When clicking 'Execute workflow'". No parameters needed. This node starts the workflow.

2. **Add Google Sheets Node to Read Keywords**  
   - Add a Google Sheets node named "Get Keywords from Sheet".  
   - Set operation to "Read Rows".  
   - Configure Document ID with your Google Sheet ID containing keywords.  
   - Set Sheet Name to the tab containing keywords (e.g., 'Keywords' or gid=0).  
   - Ensure columns include at least 'Keyword' and 'Target Country' (2-letter code).  
   - Attach Google Sheets OAuth2 credentials for access.  
   - Connect the Manual Trigger output to this node’s input.

3. **Add Code Node to URL Encode Keywords**  
   - Add a Code node named "URL Encode Keywords".  
   - Use JavaScript code to iterate over input items and replace the 'Keyword' field with URL-encoded version using `encodeURIComponent`.  
   ```javascript
   for (const item of $input.all()) {
     item.json.Keyword = encodeURIComponent(item.json.Keyword);
   }
   return $input.all();
   ```  
   - Connect the output of "Get Keywords from Sheet" to this node.

4. **Add HTTP Request Node to Fetch SERP Data**  
   - Add an HTTP Request node named "Fetch Google Search Results".  
   - Configure the URL as an expression to call Scrape.do API:  
   ```
   https://api.scrape.do/?token={{$vars.SCRAPEDO_TOKEN}}&url={{encodeURIComponent('https://www.google.com/search?q=' + $json.Keyword)}}&geoCode={{$json['Target Country'] || 'us'}}&render=true
   ```  
   - Set request method to GET.  
   - Add header parameter: `Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`  
   - Set timeout to 60000 ms (60 seconds).  
   - Connect output of "URL Encode Keywords" to this node.

5. **Add Code Node to Extract Competitor Data**  
   - Add a Code node named "Extract Competitor Data from HTML".  
   - Paste JavaScript code that parses the HTML response from Scrape.do to extract up to 10 organic search results, grabbing title, URL, description, and position.  
   - The code must handle fallback mechanisms if no results are found.  
   - Connect output of "Fetch Google Search Results" to this node.

6. **Add Google Sheets Node to Append Results**  
   - Add a Google Sheets node named "Append Results to Sheet".  
   - Set operation to "Append".  
   - Configure Document ID to the same spreadsheet as input.  
   - Set Sheet Name to the tab you want to store results in, e.g., 'Competitor Results' (create this tab if it does not exist).  
   - Map columns automatically to input fields (keyword, position, websiteTitle, websiteUrl, websiteDescription).  
   - Use the same Google Sheets OAuth2 credentials.  
   - Connect output of "Extract Competitor Data from HTML" to this node.

7. **Variable Setup**  
   - Define a workflow variable `SCRAPEDO_TOKEN` holding your Scrape.do API token.  
   - Ensure this variable is accessible to the HTTP Request node for token injection.

8. **Testing and Validation**  
   - Test the workflow by manually triggering it.  
   - Verify keywords are read, encoded, scraped, parsed, and results appended correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Scrape.do API requires a token; get yours at https://dashboard.scrape.do/                                      | Scrape.do API documentation and dashboard           |
| Ensure Google Sheets credentials have proper OAuth2 scopes to read and write sheets                            | Google Sheets API and OAuth2 setup                   |
| The 'Competitor Results' sheet/tab must exist before appending; create manually if missing                     | Google Sheets UI                                      |
| HTML parsing relies on Google SERP structure; changes in Google’s frontend may require regex updates            | Monitor Google SERP changes periodically             |
| Workflow triggered manually; for automation consider adding scheduled triggers                                  | n8n Trigger node options                              |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---