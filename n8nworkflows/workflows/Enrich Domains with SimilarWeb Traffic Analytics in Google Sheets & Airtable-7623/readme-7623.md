Enrich Domains with SimilarWeb Traffic Analytics in Google Sheets & Airtable

https://n8nworkflows.xyz/workflows/enrich-domains-with-similarweb-traffic-analytics-in-google-sheets---airtable-7623


# Enrich Domains with SimilarWeb Traffic Analytics in Google Sheets & Airtable

---

## 1. Workflow Overview

This workflow automates the enrichment of domain data with web traffic analytics sourced from SimilarWeb, integrating and updating results in Google Sheets and optionally Airtable. It is designed for users managing lists of company or website domains who want to append valuable web traffic and engagement insights for competitive intelligence, market research, or lead enrichment.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & Preprocessing:**  
  Triggered by new rows added in a Google Sheet containing domain URLs, this block cleans and standardizes the domain input for uniform API querying.

- **1.2 SimilarWeb Data Retrieval & Processing:**  
  This block queries the SimilarWeb API for detailed traffic analytics on each domain, then extracts and formats key metrics into a concise, structured output.

- **1.3 Output & Data Export:**  
  The final block updates the original Google Sheet with enriched traffic data and optionally exports the results to an Airtable base for further use or integration.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Preprocessing

**Overview:**  
This block listens for new domain entries added to a specified Google Sheet, then cleans the domain URLs by stripping protocols (`http://`, `https://`), `www.` prefixes, and trailing slashes to ensure consistency before querying SimilarWeb.

**Nodes Involved:**  
- 🟢 Sheet Trigger: New Domain  
- 🧼 Clean Domain URL  
- Sticky Note (documentation)

**Node Details:**

- **🟢 Sheet Trigger: New Domain**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Watches for new rows added to the configured Google Sheet.  
  - *Configuration:*  
    - Triggers every minute (`pollTimes` set to everyMinute) to detect new rows.  
    - Monitors the sheet tab with `gid=0` (Sheet1) in the Google Sheet identified by `YOUR_GOOGLE_SHEET_ID_HERE`.  
    - Uses OAuth2 credentials linked to Google Sheets.  
  - *Input:* None (trigger node).  
  - *Output:* Emits each new row as JSON, including raw domain URLs and row metadata.  
  - *Potential Failures:*  
    - Authentication errors if OAuth2 token expires or is invalid.  
    - Rate limits on Google Sheets API if polling too frequently.  
  - *Sticky Note:* Explains the purpose of this block as fetching and preparing domain URLs.

- **🧼 Clean Domain URL**  
  - *Type:* Set Node  
  - *Role:* Cleans and standardizes the domain string from the raw input.  
  - *Configuration:*  
    - Uses expressions to remove `http://`, `https://`, `www.`, and trailing slashes using regex replacements on `List` field from the input JSON.  
    - Assigns cleaned domain string to a new field `domain`.  
    - Passes along the sheet row number as `rowNumber` for reference.  
  - *Input:* Receives raw row data from the Google Sheets trigger.  
  - *Output:* Outputs cleaned domain with row number for downstream nodes.  
  - *Edge Cases:*  
    - Domains missing entirely or malformed URLs could lead to empty or invalid strings.  
    - Regex failures if input format changes.  
  - *Version:* Uses n8n Set node version 3.3 features.  
  - *Connections:* Output to SimilarWeb API HTTP Request node.

- **Sticky Note:**  
  - Content explains the purpose of this block: fetching URLs from the sheet and preparing them for enrichment.

---

### 2.2 SimilarWeb Data Retrieval & Processing

**Overview:**  
This block sends cleaned domains to the SimilarWeb API via a RapidAPI endpoint to retrieve detailed traffic analytics. It then extracts and formats key metrics such as global rank, country rank, monthly visits, bounce rate, top traffic sources, top countries by traffic share, and device usage split.

**Nodes Involved:**  
- 🌐 Fetch Analysis (SimilarWeb API)  
- 📊 Extract Key Traffic Metrics  
- Sticky Note (documentation)

**Node Details:**

- **🌐 Fetch Analysis (SimilarWeb API)**  
  - *Type:* HTTP Request  
  - *Role:* Queries the SimilarWeb API to get domain traffic data.  
  - *Configuration:*  
    - HTTP GET request to `https://similarweb8.p.rapidapi.com/get-analysis`.  
    - Query parameter `domain` set dynamically from the cleaned domain input (`{{$json.domain}}`).  
    - Headers include `X-RapidAPI-Key` and `X-RapidAPI-Host` with user-provided credentials for RapidAPI access.  
    - Timeout set to 30 seconds.  
    - Continue on fail enabled to avoid workflow interruption on API errors.  
  - *Input:* Takes cleaned domain from previous node.  
  - *Output:* Returns raw JSON data from SimilarWeb API under `$json.data`.  
  - *Potential Failures:*  
    - API key invalid or quota exceeded causing auth errors.  
    - Network timeouts or API downtime.  
    - Malformed domains causing API errors or empty responses.  
  - *Version:* HTTP Request node v4.1.

- **📊 Extract Key Traffic Metrics**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses raw SimilarWeb API response to extract and format key metrics into a simplified JSON structure.  
  - *Configuration:*  
    - JavaScript code accesses `$json.data` and constructs a new object with:  
      - Domain name  
      - Global rank  
      - Country rank (country and rank combined)  
      - Category rank (category and rank combined)  
      - Total visits  
      - Bounce rate formatted as percent string  
      - Pages per visit  
      - Average visit duration  
      - Top three traffic sources: direct, search, social (percentages formatted)  
      - Top three countries by traffic share (country name and percent)  
      - Device split between mobile and desktop (percentages formatted)  
  - *Input:* Receives raw SimilarWeb API JSON data.  
  - *Output:* Outputs a JSON object with only the essential traffic metrics for easier consumption downstream.  
  - *Edge Cases:*  
    - Missing or undefined fields in the API response may cause runtime errors or incomplete data.  
  - *Version:* Code node v2.

- **Sticky Note:**  
  - Describes the purpose: retrieving and formatting key SimilarWeb metrics for each domain.

---

### 2.3 Output & Data Export

**Overview:**  
This block updates the original Google Sheet with the enriched traffic data corresponding to each domain row and optionally exports the data to an Airtable base for further use or integration.

**Nodes Involved:**  
- 📤 Update Sheet with Traffic Data  
- 📁 Export to Airtable (Optional)  
- Sticky Note (documentation)

**Node Details:**

- **📤 Update Sheet with Traffic Data**  
  - *Type:* Google Sheets  
  - *Role:* Updates the existing Google Sheet entries by inserting the enriched traffic data into the corresponding rows.  
  - *Configuration:*  
    - Operation set to `update`.  
    - Target Sheet: `Sheet1` in a specified Google Sheet (`YOUR_OUTPUT_GOOGLE_SHEET_ID_HERE`).  
    - Columns mapping configured dynamically below the node (not detailed in JSON, but expected to map extracted metrics to sheet columns).  
    - Uses OAuth2 credentials for Google Sheets access.  
  - *Input:* Receives formatted traffic metrics from the code node.  
  - *Output:* None (terminal node for this branch).  
  - *Potential Failures:*  
    - Authentication or permission issues with Google Sheets.  
    - Mapping errors if columns do not exist or mismatch.  
    - Race conditions if multiple updates occur simultaneously.  
  - *Version:* Google Sheets node v4.

- **📁 Export to Airtable (Optional)**  
  - *Type:* Airtable  
  - *Role:* Creates new records in Airtable base/table with the enriched data. Optional step for users who want to synchronize data with Airtable.  
  - *Configuration:*  
    - Operation: `create` new records.  
    - Base ID and Table Name provided by user (`YOUR_AIRTABLE_BASE_ID`, `YOUR_AIRTABLE_TABLE_NAME`).  
    - Column mappings prepared dynamically.  
    - Uses Airtable OAuth2 Personal Access Token credential.  
  - *Input:* Receives formatted traffic data identical to the Google Sheets update node.  
  - *Output:* None (terminal node for this path).  
  - *Potential Failures:*  
    - Authentication failures or expired tokens.  
    - API rate limits on Airtable side.  
    - Schema mismatches between workflow data and Airtable table columns.  
  - *Version:* Airtable node v2.

- **Sticky Note:**  
  - Summarizes this block’s function: appending enriched data back to Google Sheets and optionally exporting to Airtable.

---

## 3. Summary Table

| Node Name                        | Node Type               | Functional Role                        | Input Node(s)              | Output Node(s)                             | Sticky Note                                                                                                 |
|---------------------------------|-------------------------|-------------------------------------|---------------------------|--------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| 🟢 Sheet Trigger: New Domain     | Google Sheets Trigger   | Detect new domain rows in Google Sheet | None                      | 🧼 Clean Domain URL                        | Fetches site URLs from the connected Google Sheet and prepares each row for enrichment and traffic analysis |
| 🧼 Clean Domain URL              | Set                    | Cleans and standardizes domain URLs | 🟢 Sheet Trigger: New Domain | 🌐 Fetch Analysis (SimilarWeb API)          | See above                                                                                                   |
| 🌐 Fetch Analysis (SimilarWeb API) | HTTP Request           | Queries SimilarWeb API for traffic data | 🧼 Clean Domain URL        | 📊 Extract Key Traffic Metrics              | Fetches SimilarWeb traffic and engagement insights, formats key metrics                                     |
| 📊 Extract Key Traffic Metrics    | Code                   | Extracts and formats key traffic metrics | 🌐 Fetch Analysis         | 📤 Update Sheet with Traffic Data, 📁 Export to Airtable (Optional) | See above                                                                                                   |
| 📤 Update Sheet with Traffic Data | Google Sheets           | Updates Google Sheet with enriched data | 📊 Extract Key Traffic Metrics | None                                       | Appends enriched lead data back to Google Sheet for further use                                            |
| 📁 Export to Airtable (Optional)  | Airtable                | Creates new records in Airtable with traffic data | 📊 Extract Key Traffic Metrics | None                                       | Optional export of enriched data to Airtable                                                               |
| Sticky Note                     | Sticky Note             | Documentation                       | None                      | None                                       | Explains purpose of Input Reception & Preprocessing block                                                    |
| Sticky Note1                    | Sticky Note             | Documentation                       | None                      | None                                       | Explains purpose of SimilarWeb API Request & Data Processing block                                          |
| Sticky Note2                    | Sticky Note             | Documentation                       | None                      | None                                       | Explains purpose of Output & Data Export block                                                              |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheets Trigger Node**  
   - Node Type: Google Sheets Trigger  
   - Configure:  
     - Event: `Row Added`  
     - Polling interval: every minute  
     - Sheet Name: select the sheet tab with `gid=0` (e.g., "Sheet1")  
     - Document ID: your Google Sheet ID containing the domain list  
     - Credentials: Google Sheets OAuth2 with access to the document  

2. **Create the Set Node to Clean Domain URL**  
   - Node Type: Set  
   - Configure:  
     - Add a new string field named `domain` with expression:  
       `{{$json.List.replace(/^https?:\/\/|^www\.|\/$/g, '')}}`  
       (This removes protocol, "www.", and trailing slash.)  
     - Add a new number field named `rowNumber` assigned from `{{$json.row_number}}`  
   - Connect output of Google Sheets Trigger node to this node  

3. **Create the HTTP Request Node to Query SimilarWeb API**  
   - Node Type: HTTP Request  
   - Configure:  
     - HTTP Method: GET  
     - URL: `https://similarweb8.p.rapidapi.com/get-analysis`  
     - Query Parameters: add key `domain` with value `{{$json.domain}}`  
     - Headers:  
       - `X-RapidAPI-Key` = Your RapidAPI key for SimilarWeb  
       - `X-RapidAPI-Host` = `similarweb8.p.rapidapi.com`  
     - Timeout: 30000 ms (30 seconds)  
     - Continue On Fail: enabled  
   - Connect output of Set node ("Clean Domain URL") to this node  

4. **Create the Code Node to Extract Key Traffic Metrics**  
   - Node Type: Code (JavaScript)  
   - Configure: Paste the following code:
     ```javascript
     const data = $json.data;

     const output = {
       domain: data.domain,
       globalRank: data.global_rank,
       countryRank: `${data.country_rank.country} - ${data.country_rank.rank}`,
       categoryRank: `${data.category_rank.category} - ${data.category_rank.rank}`,
       totalVisits: data.traffic_overview.total_visits,
       bounceRate: `${(data.traffic_overview.bounce_rate * 100).toFixed(2)}%`,
       pagesPerVisit: data.traffic_overview.pages_per_visit,
       avgVisitDuration: data.traffic_overview.avg_visit_duration,

       topTrafficSources: {
         direct: `${(data.traffic_sources.direct * 100).toFixed(1)}%`,
         search: `${(data.traffic_sources.search * 100).toFixed(1)}%`,
         social: `${(data.traffic_sources.social * 100).toFixed(1)}%`
       },

       topCountries: data.geography.top_countries.slice(0, 3).map(c => `${c.country}: ${(c.share * 100).toFixed(1)}%`),

       deviceSplit: {
         mobile: `${(data.mobile_vs_desktop.mobile * 100).toFixed(1)}%`,
         desktop: `${(data.mobile_vs_desktop.desktop * 100).toFixed(1)}%`
       }
     };

     return [{ json: output }];
     ```
   - Connect output of HTTP Request node to this Code node  

5. **Create the Google Sheets Node to Update Sheet with Traffic Data**  
   - Node Type: Google Sheets  
   - Configure:  
     - Operation: Update  
     - Sheet Name: "Sheet1" (or your target sheet)  
     - Document ID: your output Google Sheet ID (can be same as input or different)  
     - Define column mappings to match keys from the Code node output (domain, globalRank, countryRank, etc.)  
     - Credentials: Google Sheets OAuth2 with write access  
   - Connect output of Code node to this node  

6. **Optionally, Create the Airtable Node to Export Data**  
   - Node Type: Airtable  
   - Configure:  
     - Operation: Create  
     - Base ID and Table Name: your Airtable base and table IDs  
     - Map fields to the same keys from the Code node output for insertion  
     - Credentials: Airtable OAuth2 Personal Access Token with required permissions  
   - Connect output of Code node to this node  

7. **Add Sticky Notes for Documentation (Optional)**  
   - Add Sticky Notes near each logical block to explain purpose, as per the original workflow.  

8. **Final Connections & Activation**  
   - Ensure all nodes are connected in the following order:  
     Google Sheets Trigger → Set (Clean Domain URL) → HTTP Request (SimilarWeb) → Code (Extract Metrics) → Google Sheets Update (+ optional Airtable export)  
   - Activate the workflow  

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The SimilarWeb API is accessed via RapidAPI; you must create an account and subscribe to the SimilarWeb API to obtain the required API key (`X-RapidAPI-Key`). | [RapidAPI SimilarWeb API](https://rapidapi.com/similarweb/api/similarweb8/)                        |
| Google Sheets OAuth2 credentials require enabling the Google Sheets API in your Google Cloud Console and authorizing the workflow to access the spreadsheets. | [Google Sheets API Documentation](https://developers.google.com/sheets/api)                      |
| Airtable OAuth2 Personal Access Token must have permissions to create records in the specified base and table.                                               | [Airtable API Documentation](https://airtable.com/api)                                           |
| The workflow includes error tolerance on the HTTP Request node to continue processing other domains if one API call fails.                                  | Important for batch processing large domain lists without workflow interruption.                   |
| The domain cleaning uses regex replacement to standardize URLs; ensure input domains follow expected formats to avoid empty or invalid domain strings.      |                                                                                                   |

---

**Disclaimer:** The provided description and analysis are based solely on the automated n8n workflow JSON source. All data handled is legal and public, and the process complies with current content policies.

---