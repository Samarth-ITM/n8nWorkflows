Automated On-Page SEO Analysis & Logging with RapidAPI and Google Sheets

https://n8nworkflows.xyz/workflows/automated-on-page-seo-analysis---logging-with-rapidapi-and-google-sheets-7367


# Automated On-Page SEO Analysis & Logging with RapidAPI and Google Sheets

### 1. Workflow Overview

This workflow automates an On-Page SEO analysis process for a given website URL submitted via a web form. It leverages multiple RapidAPI endpoints to gather comprehensive SEO-related metrics such as website traffic, domain authority (DA) and page authority (PA), backlink profiles, and competitor analysis data. The collected data is then reformatted and logged into dedicated tabs of a Google Sheets document for structured reporting and ongoing tracking.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Captures the website input from a user-submitted form and stores the relevant data globally for subsequent use.
- **1.2 SEO Data Retrieval via RapidAPI:** Executes multiple HTTP POST requests to various RapidAPI endpoints to fetch different SEO metrics.
- **1.3 Data Reformatting:** Processes and extracts meaningful parts of the complex API responses to prepare data for spreadsheet insertion.
- **1.4 Data Logging to Google Sheets:** Appends the processed data into specific Google Sheets tabs corresponding to each SEO data category.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
This block triggers the workflow upon form submission, extracts the website URL (and optionally country), and stores these values for reference by downstream nodes.

**Nodes Involved:**  
- On form submission  
- Global Storage

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point that displays a web form titled "OnPage SEO ( WebSite )" with a required field named `website`. Triggers the workflow upon submission.  
  - Configuration: Form has one required field `website` (string). No additional options configured.  
  - Input: User's form submission data.  
  - Output: JSON containing `website` (and optionally `country` if provided).  
  - Edge Cases: Form validation ensures `website` is provided, but no URL validation is done here. Malformed URLs may cause downstream API errors.

- **Global Storage**  
  - Type: Set node  
  - Role: Saves `website` (and `country` if present) from trigger for use by subsequent nodes.  
  - Configuration: Copies `website` and `country` from input JSON into the node’s output JSON.  
  - Input: Output from "On form submission" node.  
  - Output: JSON with explicitly assigned keys `website` and `country`.  
  - Edge Cases: If `country` is undefined, it remains unset.

---

#### 1.2 SEO Data Retrieval via RapidAPI

**Overview:**  
This block calls four different RapidAPI endpoints using HTTP POST to gather SEO-related data for the given website.

**Nodes Involved:**  
- Website Traffic Cheker  
- Website Metrics DA PA  
- Top Baclinks  
- Competitors Analysis

**Node Details:**

- **Website Traffic Cheker**  
  - Type: HTTP Request  
  - Role: Queries `webtraffic.php` endpoint on RapidAPI to retrieve website traffic summary.  
  - Configuration: POST request with multipart/form-data containing `website` parameter. Headers include `x-rapidapi-host` and `x-rapidapi-key` (placeholder "your key" must be replaced with a valid key).  
  - Input: JSON with `website` from Global Storage.  
  - Output: Raw API response JSON containing traffic summary data.  
  - Edge Cases: API key or host errors, request timeouts, invalid website URL response.

- **Website Metrics DA PA**  
  - Type: HTTP Request  
  - Role: Queries `dapa.php` endpoint to get domain authority (DA), page authority (PA), spam score, domain rating (DR), and organic traffic metrics.  
  - Configuration: Similar POST multipart/form-data with `website`. Headers with RapidAPI host and key.  
  - Input: JSON with `website`.  
  - Output: Raw API response JSON with DA/PA metrics.  
  - Edge Cases: Same as above.

- **Top Baclinks**  
  - Type: HTTP Request  
  - Role: Calls `backlink.php` endpoint to retrieve backlink data for the website.  
  - Configuration: POST multipart/form-data with `website`, RapidAPI headers.  
  - Input: JSON with `website`.  
  - Output: Raw API JSON containing backlinks overview and detailed backlinks data.  
  - Edge Cases: API limits, key errors, malformed response.

- **Competitors Analysis**  
  - Type: HTTP Request  
  - Role: Calls `competitor.php` endpoint to fetch competitor websites and related SEO datasets.  
  - Configuration: POST multipart/form-data with `website`, RapidAPI headers.  
  - Input: JSON with `website`.  
  - Output: Raw API JSON with competitor data arrays.  
  - Edge Cases: Large data volume, API throttling, invalid data.

---

#### 1.3 Data Reformatting

**Overview:**  
This block processes raw API responses to extract relevant data subsets, flatten complex nested structures, and prepare clean JSON objects that can be appended to Google Sheets.

**Nodes Involved:**  
- Re-Format  
- Re-Format 2  
- Re -Format 3  
- Re -Format 4  
- Re -Format 5

**Node Details:**

- **Re-Format**  
  - Type: Code (JavaScript)  
  - Role: Extracts the first traffic summary object from the nested API response: `data.semrushAPI.trafficSummary[0]`.  
  - Input: Output from "Website Traffic Cheker".  
  - Output: JSON object containing traffic summary metrics.  
  - Edge Cases: If `trafficSummary` array is empty or missing, code may throw or return undefined.

- **Re-Format 2**  
  - Type: Code  
  - Role: Extracts the `data` object from DA/PA API response for direct mapping.  
  - Input: Output from "Website Metrics DA PA".  
  - Output: Clean JSON with DA, PA, spam score, etc.  
  - Edge Cases: Missing or malformed `data` object.

- **Re -Format 3**  
  - Type: Code  
  - Role: Extracts the `data.semrushAPI.backlinksOverview` object containing aggregate backlink metrics.  
  - Input: Output from "Top Baclinks".  
  - Output: JSON object with backlink overview fields.  
  - Edge Cases: Missing backlinks overview data.

- **Re -Format 4**  
  - Type: Code  
  - Role: Extracts detailed backlinks list from `data.semrushAPI.backlinks` array.  
  - Input: Output from "Top Baclinks".  
  - Output: Array of backlink objects.  
  - Edge Cases: Empty backlinks array.

- **Re -Format 5**  
  - Type: Code  
  - Role: Flattens multiple arrays inside `data.semrushAPI` from competitor API response into individual rows, each tagged with a `dataset` name. Loops through all arrays under `data.semrushAPI` and creates separate records.  
  - Input: Output from "Competitors Analysis".  
  - Output: Array of flattened JSON objects with dataset labels.  
  - Edge Cases: Non-array properties ignored; empty arrays result in no rows.

---

#### 1.4 Data Logging to Google Sheets

**Overview:**  
This block appends the reformatted SEO data into corresponding Google Sheets tabs, using a service account for authentication.

**Nodes Involved:**  
- Website Traffic  
- DA PA  
- Backlinks Overview  
- Backlinks  
- Competitor Analysis

**Node Details:**

- **Website Traffic**  
  - Type: Google Sheets  
  - Role: Appends traffic metrics (users, visits, bounce rate, time on site, pages per visit, organic search) into the "WebSite Traffic" sheet.  
  - Configuration: Appends rows auto-mapped from input JSON fields; uses service account credentials.  
  - Input: Output from "Re-Format".  
  - Output: Confirmation of append operation.  
  - Edge Cases: Sheet ID or name mismatch, API quota limits.

- **DA PA**  
  - Type: Google Sheets  
  - Role: Appends DA, PA, spam score, domain rating, and organic traffic into the "DA PA" sheet.  
  - Configuration: Auto-mapped to columns, service account auth.  
  - Input: Output from "Re-Format 2".  
  - Output: Append confirmation.  
  - Edge Cases: Same as above.

- **Backlinks Overview**  
  - Type: Google Sheets  
  - Role: Appends aggregated backlink overview metrics into the "Backlinks Overview" sheet.  
  - Configuration: Auto-mapped to columns, service account auth.  
  - Input: Output from "Re -Format 3".  
  - Output: Append confirmation.  
  - Edge Cases: Service account permissions, sheet structure.

- **Backlinks**  
  - Type: Google Sheets  
  - Role: Appends detailed backlinks rows into the "Backlinks" sheet.  
  - Configuration: Auto-mapped, service account auth.  
  - Input: Output from "Re -Format 4".  
  - Output: Append confirmation.  
  - Edge Cases: Large data volume may cause rate limiting or timeouts.

- **Competitor Analysis**  
  - Type: Google Sheets  
  - Role: Appends competitor and keyword-related datasets into the "Competitor Analysis" sheet.  
  - Configuration: Auto-mapped columns, service account auth.  
  - Input: Output from "Re -Format 5".  
  - Output: Append confirmation.  
  - Edge Cases: Large data volume, complex data flattening.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                              | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                         |
|-----------------------|--------------------|----------------------------------------------|------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| On form submission    | Form Trigger       | Receive website input via form submission     | (trigger)                    | Global Storage                | **On form submission** — Shows a web form (field: `website`) and triggers the workflow on submit.                                  |
| Global Storage        | Set                | Store website and country for reuse           | On form submission           | Website Traffic Cheker, Competitors Analysis , Website Metrics DA PA, Top Baclinks | **Global Storage** — Copies `website` (and optional `country`) into the execution JSON for reuse.                                  |
| Website Traffic Cheker | HTTP Request       | Fetch website traffic data from RapidAPI      | Global Storage               | Re-Format                    | **Website Traffic Cheker** — POSTs `website` to `webtraffic.php` (RapidAPI) to fetch traffic summary.                              |
| Re-Format             | Code               | Extract first traffic summary object          | Website Traffic Cheker       | Website Traffic              | **Re-Format** — Extracts `data.semrushAPI.trafficSummary[0]` from the traffic API response.                                        |
| Website Traffic       | Google Sheets      | Append traffic data to "WebSite Traffic" sheet| Re-Format                   |                              | **Website Traffic** — Appends traffic metrics (visits, users, bounce, etc.) to the **"WebSite Traffic"** sheet.                   |
| Website Metrics DA PA  | HTTP Request       | Fetch domain/page authority and metrics       | Global Storage               | Re-Format 2                  | **Website Metrics DA PA** — POSTs `website` to `dapa.php` (RapidAPI) to get DA, PA, spam score, DR, org traffic.                   |
| Re-Format 2           | Code               | Extract DA/PA data object from response       | Website Metrics DA PA        | DA PA                        | **Re-Format 2** — Pulls the `data` object from the DA/PA API response for clean mapping                                            |
| DA PA                 | Google Sheets      | Append DA/PA data to "DA PA" sheet             | Re-Format 2                 |                              | **DA PA** — Appends DA/PA and related fields into the **"DA PA"** sheet.                                                          |
| Top Baclinks          | HTTP Request       | Fetch backlink data for website                | Global Storage               | Re -Format 3, Re -Format 4   | **Top Baclinks** — POSTs `website` to `backlink.php` (RapidAPI) to retrieve backlink data.                                         |
| Re -Format 3          | Code               | Extract backlinks overview data                 | Top Baclinks                | Backlinks Overview           | **Re-Format 3** — Extracts `data.semrushAPI.backlinksOverview` (aggregate backlink metrics).                                       |
| Backlinks Overview    | Google Sheets      | Append backlinks overview to sheet             | Re -Format 3                |                              | **Backlinks Overview** — Appends overview metrics into the **"Backlinks Overview"** sheet.                                        |
| Re -Format 4          | Code               | Extract detailed backlinks array                | Top Baclinks                | Backlinks                   | **Re-Format 4** — Extracts detailed `data.semrushAPI.backlinks` (individual backlinks list).                                       |
| Backlinks             | Google Sheets      | Append detailed backlinks to "Backlinks" sheet| Re -Format 4                |                              | **Backlinks** — Appends each backlink row into the **"Backlinks"** sheet.                                                         |
| Competitors Analysis  | HTTP Request       | Fetch competitor SEO data and datasets          | Global Storage               | Re -Format 5                 | **Competitors Analysis** — POSTs `website` to `competitor.php` (RapidAPI) to fetch competitors/data sets.                         |
| Re -Format 5          | Code               | Flatten competitor API data arrays into rows   | Competitors Analysis         | Competitor Analysis          | **Re-Format 5** — Flattens all array datasets under `data.semrushAPI` into rows with a `dataset` label.                           |
| Competitor Analysis   | Google Sheets      | Append competitor data rows to sheet            | Re -Format 5                |                              | **Competitor Analysis** — Appends the flattened competitor and keyword rows into the **"Competitor Analysis"** sheet.             |
| Sticky Notes          | Sticky Note        | Documentation and comments                      | -                            | -                            | Various notes describing node roles and workflow overview.                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Title: "OnPage SEO ( WebSite )"  
   - Add one required field: `website` (string)  
   - No additional form options  
   - This node triggers the workflow on form submission.

2. **Create Set Node "Global Storage"**  
   - Type: Set  
   - Add two fields:  
     - `website` : Set to expression `{{$json["website"]}}`  
     - `country` : Set to expression `{{$json["country"]}}` (optional)  
   - Connect "On form submission" → "Global Storage"

3. **Create HTTP Request Node "Website Traffic Cheker"**  
   - URL: `https://seo-on-page.p.rapidapi.com/webtraffic.php`  
   - Method: POST  
   - Content-Type: multipart/form-data  
   - Body Parameters: `website` = `{{$json["website"]}}`  
   - Headers:  
     - `x-rapidapi-host`: `seo-on-page.p.rapidapi.com`  
     - `x-rapidapi-key`: *Your RapidAPI key here*  
   - Connect "Global Storage" → "Website Traffic Cheker"

4. **Create Code Node "Re-Format"**  
   - Code (JavaScript):  
     ```javascript
     return $input.first().json.data.semrushAPI.trafficSummary[0];
     ```  
   - Connect "Website Traffic Cheker" → "Re-Format"

5. **Create Google Sheets Node "Website Traffic"**  
   - Operation: Append  
   - Document ID: Use your Google Sheet ID (matches your project)  
   - Sheet Name: "WebSite Traffic" (tab name)  
   - Columns: Auto-map from input data fields (users, visits, bounceRate, etc.)  
   - Authentication: Use Google Service Account credentials  
   - Connect "Re-Format" → "Website Traffic"

6. **Create HTTP Request Node "Website Metrics DA PA"**  
   - URL: `https://seo-on-page.p.rapidapi.com/dapa.php`  
   - Method: POST  
   - Content-Type: multipart/form-data  
   - Body Parameters: `website` = `{{$json["website"]}}`  
   - Headers same as step 3  
   - Connect "Global Storage" → "Website Metrics DA PA"

7. **Create Code Node "Re-Format 2"**  
   - Code:  
     ```javascript
     return $input.first().json.data;
     ```  
   - Connect "Website Metrics DA PA" → "Re-Format 2"

8. **Create Google Sheets Node "DA PA"**  
   - Operation: Append  
   - Document ID: same as step 5  
   - Sheet Name: "DA PA"  
   - Columns: Auto-map DA, PA, spam_score, dr, org_traffic  
   - Authentication: Service Account  
   - Connect "Re-Format 2" → "DA PA"

9. **Create HTTP Request Node "Top Baclinks"**  
   - URL: `https://seo-on-page.p.rapidapi.com/backlink.php`  
   - Method: POST  
   - Content-Type: multipart/form-data  
   - Body Parameters: `website` = `{{$json["website"]}}`  
   - Headers same as step 3  
   - Connect "Global Storage" → "Top Baclinks"

10. **Create Code Node "Re -Format 3"**  
    - Code:  
      ```javascript
      return $input.first().json.data.semrushAPI.backlinksOverview;
      ```  
    - Connect "Top Baclinks" → "Re -Format 3"

11. **Create Google Sheets Node "Backlinks Overview"**  
    - Operation: Append  
    - Document ID: same as step 5  
    - Sheet Name: "Backlinks Overview"  
    - Columns: Auto-map backlink overview fields (ascore, domainsNum, followsNum, etc.)  
    - Authentication: Service Account  
    - Connect "Re -Format 3" → "Backlinks Overview"

12. **Create Code Node "Re -Format 4"**  
    - Code:  
      ```javascript
      return $input.first().json.data.semrushAPI.backlinks;
      ```  
    - Connect "Top Baclinks" → "Re -Format 4"

13. **Create Google Sheets Node "Backlinks"**  
    - Operation: Append  
    - Document ID: same as step 5  
    - Sheet Name: "Backlinks"  
    - Columns: Auto-map all backlink data fields  
    - Authentication: Service Account  
    - Connect "Re -Format 4" → "Backlinks"

14. **Create HTTP Request Node "Competitors Analysis"**  
    - URL: `https://seo-on-page.p.rapidapi.com/competitor.php`  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Body Parameters: `website` = `{{$json["website"]}}`  
    - Headers same as step 3  
    - Connect "Global Storage" → "Competitors Analysis"

15. **Create Code Node "Re -Format 5"**  
    - Code:  
      ```javascript
      const apiData = $input.first().json.data.semrushAPI;
      let allRows = [];
      for (const key in apiData) {
        if (Array.isArray(apiData[key])) {
          apiData[key].forEach(item => {
            allRows.push({ json: { dataset: key, ...item } });
          });
        }
      }
      return allRows;
      ```  
    - Connect "Competitors Analysis" → "Re -Format 5"

16. **Create Google Sheets Node "Competitor Analysis"**  
    - Operation: Append  
    - Document ID: same as step 5  
    - Sheet Name: "Competitor Analysis"  
    - Columns: Auto-map fields including `dataset` label  
    - Authentication: Service Account  
    - Connect "Re -Format 5" → "Competitor Analysis"

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow requires valid RapidAPI keys with access to `seo-on-page.p.rapidapi.com` endpoints for traffic, DA/PA, backlinks, etc.| Replace `"your key"` placeholders in HTTP Request nodes with your actual RapidAPI subscription key.|
| Google Sheets service account credentials must have edit access to the target spreadsheet.                                           | Setup Google API credentials in n8n for authentication.                                           |
| The workflow uses multipart/form-data POST requests for all API calls to RapidAPI.                                                  | Ensure the RapidAPI plan supports these calls and monitor quota limits to avoid failures.         |
| Data flattening in "Re -Format 5" assumes all competitor API datasets are arrays under `data.semrushAPI`.                           | Changes in API response structure may require code adjustments.                                   |
| Form input validation is minimal; users should enter valid website URLs to avoid API errors.                                         | Consider adding additional validation steps if needed.                                           |
| Sticky notes in the workflow provide detailed node descriptions and workflow logic summaries.                                       | Useful for onboarding new users or reviewing workflow changes.                                   |

---

**Disclaimer:** The text provided is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.