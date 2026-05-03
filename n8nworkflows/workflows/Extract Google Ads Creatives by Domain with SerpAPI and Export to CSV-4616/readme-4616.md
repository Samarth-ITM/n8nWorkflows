Extract Google Ads Creatives by Domain with SerpAPI and Export to CSV

https://n8nworkflows.xyz/workflows/extract-google-ads-creatives-by-domain-with-serpapi-and-export-to-csv-4616


# Extract Google Ads Creatives by Domain with SerpAPI and Export to CSV

### 1. Workflow Overview

This workflow automates the extraction of Google Ads creatives associated with a specified domain and region by querying the SerpAPI Google Ads Transparency Center. It processes the returned ads to isolate those targeted specifically at the domain of interest, sorts them by format (text, image, and video), and exports each category into separate CSV files for further analysis or reporting.

**Target Use Cases:**  
- Digital marketers seeking transparency on competitor ads by domain  
- Analysts compiling creative assets from Google Ads for a given region  
- Automated reporting pipelines for ad creative audits

**Logical Blocks:**

- **1.1 Input Reception:** Manual trigger and setting domain and region parameters  
- **1.2 Data Retrieval:** HTTP request to SerpAPI to fetch ads transparency data  
- **1.3 Data Filtering:** Extract creatives relevant to the specified domain  
- **1.4 Data Classification:** Split ads by their format type (text, image, video)  
- **1.5 Data Export:** Convert each ad format subset into CSV files saved locally  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Receives manual initiation of the workflow and sets the key input parameters: domain and region code.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set Domain & Region

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or one-off runs  
    - Configuration: No parameters required  
    - Connections: Outputs to Set Domain & Region  
    - Edge Cases: None typical; user must trigger manually  

  - **Set Domain & Region**  
    - Type: Set  
    - Role: Define variables for domain and geographic region code  
    - Configuration:  
      - `domain`: e.g., "example.com" (string)  
      - `region`: e.g., "example_region_code" (string, numeric code per SerpAPI region codes)  
    - Connections: Outputs to Get Ads Page 1  
    - Edge Cases: Incorrect or unsupported region codes may yield empty or error responses  
    - Notes: Region codes must conform to SerpAPI specification [SerpAPI Regions](https://serpapi.com/google-ads-transparency-center-regions)

#### 1.2 Data Retrieval

- **Overview:**  
  Queries SerpAPI’s Google Ads Transparency Center API for ads related to the specified domain and region.

- **Nodes Involved:**  
  - Get Ads Page 1

- **Node Details:**

  - **Get Ads Page 1**  
    - Type: HTTP Request  
    - Role: Fetch Google Ads data from SerpAPI based on input params  
    - Configuration:  
      - URL: `https://serpapi.com/search.json`  
      - Authentication: Generic HTTP Query with SerpAPI key (`api_key`)  
      - Query Parameters:  
        - `engine`: fixed to `google_ads_transparency_center`  
        - `text`: domain input (`{{$json["domain"]}}`)  
        - `region`: region input (`{{$json["region"]}}`)  
        - `api_key`: user’s SerpAPI key placeholder  
    - Connections: Outputs to Extract Ad Creatives  
    - Edge Cases:  
      - Invalid API key → HTTP 401/403  
      - Unsupported region or domain → no results or empty arrays  
      - API rate limits or network errors  
    - Notes: Pagination not implemented; only first page fetched  
    - Documentation: [SerpAPI Google Ads Transparency API](https://serpapi.com/google-ads-transparency-center-api)

#### 1.3 Data Filtering

- **Overview:**  
  Filters the returned ads to include only those with a target domain matching the input domain precisely.

- **Nodes Involved:**  
  - Extract Ad Creatives

- **Node Details:**

  - **Extract Ad Creatives**  
    - Type: Function  
    - Role: Extract and filter ad creatives based on domain match  
    - Configuration:  
      - JavaScript function filtering `ad_creatives` by comparing each ad’s `target_domain` (lowercased) to the input domain (`search_parameters.text`)  
      - Returns filtered ads as individual items  
    - Connections: Outputs to Split by Format1  
    - Edge Cases:  
      - Missing `ad_creatives` property → runtime error or empty output  
      - Ads without `target_domain` property are ignored  
      - Case sensitivity handled by lowercasing both sides  
    - Notes: Ensures only pertinent ads are processed downstream

#### 1.4 Data Classification

- **Overview:**  
  Classifies filtered ads by their `format` attribute into three categories: text, image, or video.

- **Nodes Involved:**  
  - Split by Format1

- **Node Details:**

  - **Split by Format1**  
    - Type: Switch  
    - Role: Routes ads to different outputs based on `format` field  
    - Configuration:  
      - Switch on `{{$json["format"]}}`  
      - Rules:  
        - Equal to `text` → Output 1  
        - Equal to `image` → Output 2  
        - Equal to `video` → Output 3  
    - Connections:  
      - Output 1 → Convert Text Ads to CSV  
      - Output 2 → Convert Image Ads to CSV  
      - Output 3 → Convert Video Ads to CSV1  
    - Edge Cases:  
      - Ads with missing or unexpected `format` value are dropped (no default path)  
      - Case sensitivity assumed exact match (all lowercase)  
    - Notes: Facilitates clean separation for CSV export

#### 1.5 Data Export

- **Overview:**  
  Converts each category of ads into CSV files saved under `/files/` directory with domain-specific filenames.

- **Nodes Involved:**  
  - Convert Text Ads to CSV  
  - Convert Image Ads to CSV  
  - Convert Video Ads to CSV1

- **Node Details:**

  - **Convert Text Ads to CSV**  
    - Type: Spreadsheet File  
    - Role: Export text format ads to CSV file  
    - Configuration:  
      - Operation: toFile  
      - File Name: `/files/text_{{ $json.target_domain }}_ads.csv` (dynamic by domain)  
      - Format: CSV (default)  
    - Connections: None (end-node)  
    - Edge Cases: Empty input results in empty CSV  
    - Notes: Contains text-only ad creatives  

  - **Convert Image Ads to CSV**  
    - Type: Spreadsheet File  
    - Role: Export image format ads to CSV file  
    - Configuration:  
      - Operation: toFile  
      - File Name: `/files/image_{{ $json.target_domain }}_ads.csv`  
      - File Format: CSV  
    - Connections: None (end-node)  
    - Edge Cases: Ads without image URLs or dimensions may have missing fields  
    - Notes: Includes image URLs and metadata  

  - **Convert Video Ads to CSV1**  
    - Type: Spreadsheet File  
    - Role: Export video format ads to CSV file  
    - Configuration:  
      - Operation: toFile  
      - File Name: `/files/video_{{ $json.target_domain }}_ads.csv`  
      - File Format: CSV  
    - Connections: None (end-node)  
    - Edge Cases: Video ads lacking preview links may have empty fields  
    - Notes: Contains video preview or embed links  

---

### 3. Summary Table

| Node Name                | Node Type         | Functional Role                     | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                    |
|--------------------------|-------------------|-----------------------------------|----------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Manual start of workflow           |                            | Set Domain & Region              | ✅ When clicking 'Test workflow' - Trigger the workflow manually during testing or development. No config needed. |
| Set Domain & Region       | Set               | Define domain and region parameters| When clicking ‘Test workflow’| Get Ads Page 1                  | ✏️ Set Domain & Region - Define which domain and region you want to extract ads for. See region codes link.     |
| Get Ads Page 1            | HTTP Request      | Fetch ads data from SerpAPI        | Set Domain & Region         | Extract Ad Creatives            | 🌐 Get Ads Page 1 - Calls SerpApi using google_ads_transparency_center engine. Requires SerpApi API Key.         |
| Extract Ad Creatives      | Function          | Filter ads matching the domain     | Get Ads Page 1              | Split by Format1                | 🧠 Extract Ad Creatives - Filters ads where target_domain matches input domain, ensuring relevance.              |
| Split by Format1          | Switch            | Route ads by format (text/image/video)| Extract Ad Creatives      | Convert Text Ads to CSV, Convert Image Ads to CSV, Convert Video Ads to CSV1 | 🔀 Split by Format - Sorts ads by format for clean CSV outputs.                                                  |
| Convert Text Ads to CSV   | Spreadsheet File  | Export text ads to CSV             | Split by Format1 (text output)|                             | 📄 Convert Text Ads to CSV - Saves text ads as text_{domain}_ads.csv in /files/ directory.                       |
| Convert Image Ads to CSV  | Spreadsheet File  | Export image ads to CSV            | Split by Format1 (image output)|                            | 🖼️ Convert Image Ads to CSV - Saves image ads as image_{domain}_ads.csv with dimensions and URLs.                |
| Convert Video Ads to CSV1 | Spreadsheet File  | Export video ads to CSV            | Split by Format1 (video output)|                            | 🎞️ Convert Video Ads to CSV - Exports video ads with preview or embedded links as video_{domain}_ads.csv.       |
| Sticky Note2              | Sticky Note       | Annotation for manual trigger      |                            |                                 | ✅ When clicking 'Test workflow' - Trigger manually, no config needed.                                           |
| Sticky Note               | Sticky Note       | Annotation for Set Domain & Region |                            |                                 | ✏️ Set Domain & Region - Define domain and region, see SerpAPI region codes link.                               |
| Sticky Note1              | Sticky Note       | Annotation for Get Ads Page 1      |                            |                                 | 🌐 Get Ads Page 1 - Uses SerpAPI Google Ads Transparency Center API, needs API key.                              |
| Sticky Note3              | Sticky Note       | Annotation for Extract Ad Creatives|                            |                                 | 🧠 Extract Ad Creatives - Filters only ads targeting the input domain.                                          |
| Sticky Note4              | Sticky Note       | Annotation for Split by Format1    |                            |                                 | 🔀 Split by Format - Sorts ads into text, image, and video based on format field.                                |
| Sticky Note5              | Sticky Note       | Annotation for Convert Text Ads to CSV |                         |                                 | 📄 Convert Text Ads to CSV - Saves text-only creatives to CSV.                                                  |
| Sticky Note6              | Sticky Note       | Annotation for Convert Image Ads to CSV |                         |                                 | 🖼️ Convert Image Ads to CSV - Saves image creatives including URLs and dimensions.                              |
| Sticky Note7              | Sticky Note       | Annotation for Convert Video Ads to CSV1 |                        |                                 | 🎞️ Convert Video Ads to CSV - Saves video creatives with preview/embed links.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow during testing or routine runs  
   - No configuration needed

2. **Create Set Node (Set Domain & Region)**  
   - Type: Set  
   - Parameters:  
     - Add two string values:  
       - `domain`: e.g., "example.com"  
       - `region`: e.g., "example_region_code" (consult SerpAPI region codes)  
   - Connect Manual Trigger node output to this node’s input

3. **Create HTTP Request Node (Get Ads Page 1)**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://serpapi.com/search.json`  
     - Authentication: Generic HTTP Query with HTTP Query Auth  
     - Query Parameters:  
       - `engine` = `google_ads_transparency_center` (static)  
       - `text` = Expression: `{{$json["domain"]}}` (from Set node)  
       - `region` = Expression: `{{$json["region"]}}`  
       - `api_key` = Your SerpAPI API key (store securely in credentials)  
   - Connect Set Domain & Region node output to this node’s input

4. **Create Function Node (Extract Ad Creatives)**  
   - Type: Function  
   - Code:  
```javascript
const targetDomain = items[0].json.search_parameters.text.toLowerCase();

return items[0].json.ad_creatives
  .filter(ad => (ad.target_domain || '').toLowerCase() === targetDomain)
  .map(ad => ({ json: ad }));
```
   - Connect HTTP Request output to this node’s input

5. **Create Switch Node (Split by Format1)**  
   - Type: Switch  
   - Property to check: `format` (string)  
   - Rules:  
     - If `format` equals `text` → output 1  
     - If `format` equals `image` → output 2  
     - If `format` equals `video` → output 3  
   - Connect Function node output to Switch node input

6. **Create Spreadsheet File Node (Convert Text Ads to CSV)**  
   - Type: Spreadsheet File  
   - Operation: toFile  
   - File Name: `/files/text_{{ $json.target_domain }}_ads.csv` (use expression)  
   - Connect Switch node output 1 (text) to this node input

7. **Create Spreadsheet File Node (Convert Image Ads to CSV)**  
   - Type: Spreadsheet File  
   - Operation: toFile  
   - File Format: CSV  
   - File Name: `/files/image_{{ $json.target_domain }}_ads.csv`  
   - Connect Switch node output 2 (image) to this node input

8. **Create Spreadsheet File Node (Convert Video Ads to CSV1)**  
   - Type: Spreadsheet File  
   - Operation: toFile  
   - File Format: CSV  
   - File Name: `/files/video_{{ $json.target_domain }}_ads.csv`  
   - Connect Switch node output 3 (video) to this node input

9. **Verify Credential Setup**  
   - Create and configure a Credential for SerpAPI with your API key  
   - Assign this credential to the HTTP Request node

10. **Set Up File Storage**  
    - Ensure n8n has write access to `/files/` directory or configure accordingly  
    - Confirm CSV files are generated and accessible after workflow runs

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Region codes must be numeric and conform to SerpAPI's Google Ads Transparency Center specification. | https://serpapi.com/google-ads-transparency-center-regions                                        |
| The workflow currently fetches only the first page of ads; pagination can be implemented with `next_page_token` from SerpAPI. | https://serpapi.com/google-ads-transparency-center-api                                            |
| The ad creatives extraction function expects the JSON response structure to include `ad_creatives` and `search_parameters`. | n8n workflow internal expectation                                                                |
| CSV export nodes save files under `/files/` directory—ensure this directory exists and has proper permissions. | n8n file system configuration                                                                      |