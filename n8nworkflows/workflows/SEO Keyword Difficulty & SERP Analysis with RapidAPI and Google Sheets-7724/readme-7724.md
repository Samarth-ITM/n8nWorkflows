SEO Keyword Difficulty & SERP Analysis with RapidAPI and Google Sheets

https://n8nworkflows.xyz/workflows/seo-keyword-difficulty---serp-analysis-with-rapidapi-and-google-sheets-7724


# SEO Keyword Difficulty & SERP Analysis with RapidAPI and Google Sheets

### 1. Workflow Overview

This workflow automates SEO keyword analysis by collecting user inputs, querying an external keyword difficulty API, and saving relevant data into Google Sheets for further analysis. It is designed for SEO analysts or marketers who want to quickly assess keyword difficulty and review SERP (Search Engine Results Page) details programmatically.

The workflow logic is divided into the following blocks:

- **1.1 Input Reception:** Collects user input (`keyword` and `country`) via a web form.
- **1.2 API Request:** Sends the input data to an external RapidAPI service to retrieve keyword difficulty and SERP results.
- **1.3 Response Reformatting:** Extracts and separates the keyword difficulty metric and SERP results from the API response.
- **1.4 Data Storage:** Appends the reformatted keyword difficulty and SERP data into two separate Google Sheets tabs for record and analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon submission of a web form, capturing the keyword and target country from the user.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point that listens for form submissions titled "Keyword Difficulty Checker" with fields `keyword` (required) and `country` (required).  
    - Configuration:  
      - `formTitle`: "Keyword Difficulty Checker"  
      - `formFields`: Two fields (`keyword`, `country`), both required  
      - Webhook ID assigned for external calls  
    - Inputs: None (trigger node)  
    - Outputs: Passes submitted form data downstream  
    - Edge Cases:  
      - Missing required fields will prevent submission  
      - Webhook connectivity issues may cause trigger failures  
    - Version: 2.2  

---

#### 1.2 API Request

- **Overview:**  
  This block sends the collected keyword and country to the RapidAPI keyword difficulty checker service via HTTP POST to retrieve keyword difficulty index and SERP results.

- **Nodes Involved:**  
  - Keyword Difficulty Checker

- **Node Details:**

  - **Keyword Difficulty Checker**  
    - Type: HTTP Request  
    - Role: Sends POST request to external API to fetch SEO metrics  
    - Configuration:  
      - URL: `https://keyword-difficulty-checker1.p.rapidapi.com/keywordDifficulty.php`  
      - Method: POST  
      - Headers: `x-rapidapi-host` and `x-rapidapi-key` (API key required)  
      - Body: Form-data with parameters `keyword` and `country` from previous node’s JSON  
      - Content-Type: multipart-form-data  
    - Inputs: Form data from "On form submission"  
    - Outputs: JSON response containing `keywordDifficulty` and `serpResults` data  
    - Edge Cases:  
      - API authentication failure due to invalid/missing key  
      - Timeout or network issues  
      - Unexpected or malformed API response  
    - Version: 4.2  

---

#### 1.3 Response Reformatting

- **Overview:**  
  This block separates the keyword difficulty value and SERP results from the API response JSON into two streams for individual processing.

- **Nodes Involved:**  
  - Reformat 1  
  - Reformat 2

- **Node Details:**

  - **Reformat 1**  
    - Type: Code (JavaScript)  
    - Role: Extracts the `keywordDifficulty` value from the API response JSON  
    - Configuration:  
      - JS Code: `return $input.first().json.data.semrushAPI.keywordDifficulty;`  
    - Inputs: API response from "Keyword Difficulty Checker"  
    - Outputs: Single JSON object with keyword difficulty metric  
    - Edge Cases:  
      - If the path `data.semrushAPI.keywordDifficulty` is missing, may fail or return undefined  
    - Version: 2  

  - **Reformat 2**  
    - Type: Code (JavaScript)  
    - Role: Extracts the `serpResults` array from the API response JSON  
    - Configuration:  
      - JS Code: `return $input.first().json.data.semrushAPI.serpResults;`  
    - Inputs: API response from "Keyword Difficulty Checker"  
    - Outputs: JSON array of SERP results  
    - Edge Cases:  
      - Missing or empty `serpResults` will lead to empty output  
    - Version: 2  

---

#### 1.4 Data Storage

- **Overview:**  
  This block appends the extracted keyword difficulty and SERP data into two separate Google Sheets tabs, enabling ongoing data collection and analysis.

- **Nodes Involved:**  
  - Keyword Difficulty Checker1  
  - SERP Results

- **Node Details:**

  - **Keyword Difficulty Checker1**  
    - Type: Google Sheets  
    - Role: Appends keyword and difficulty index to the "backlink overflow" sheet  
    - Configuration:  
      - Operation: Append  
      - Document ID: (empty in JSON, must be set)  
      - Sheet Name: 4590546 (likely sheet GID)  
      - Columns mapped automatically for `keyword` and `keywordDifficultyIndex`  
      - Authentication: Service Account credential for Google API  
    - Inputs: Output from "Reformat 1" (keyword difficulty data)  
    - Outputs: None  
    - Edge Cases:  
      - Missing or incorrect Google Sheets credentials cause auth errors  
      - Document ID must be valid and accessible  
    - Version: 4.6  

  - **SERP Results**  
    - Type: Google Sheets  
    - Role: Appends SERP results data array to the "backlinks" sheet  
    - Configuration:  
      - Operation: Append  
      - Document ID: `1BL8gZNhmaiZVThuEIFFra0XNQnnNrkxTsgEKeRoRHFM`  
      - Sheet Name: "gid=0" (first sheet)  
      - Columns mapped automatically from input data  
      - Authentication: Service Account credential for Google API  
    - Inputs: Output from "Reformat 2" (SERP results array)  
    - Outputs: None  
    - Edge Cases:  
      - Authentication or permission errors with Google Sheets  
      - Data schema mismatches leading to failed appends  
    - Version: 4.6  

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                                   | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                                        |
|----------------------------|---------------------|-------------------------------------------------|---------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger        | Receives user input (keyword, country)           | None                      | Keyword Difficulty Checker      | 📝 On form submission Collects user-submitted data (`keyword` and `country`) via a form trigger.                   |
| Keyword Difficulty Checker | HTTP Request       | Sends POST to RapidAPI to get keyword difficulty | On form submission        | Reformat 1, Reformat 2          | 🌐 Keyword Difficulty Checker Sends the submitted data to an external API (`keywordDifficulty.php`) using POST.    |
| Reformat 1                 | Code                | Extracts keywordDifficulty from API response     | Keyword Difficulty Checker | Keyword Difficulty Checker1     | 📦 Reformat 1 Extracts the `keywordDifficulty` value from the API response JSON.                                   |
| Reformat 2                 | Code                | Extracts serpResults array from API response      | Keyword Difficulty Checker | SERP Results                   | 📦 Reformat 2 Extracts the `serpResults` data array from the API response JSON.                                    |
| Keyword Difficulty Checker1| Google Sheets       | Appends keyword difficulty data to Google Sheet  | Reformat 1                | None                           | 📊 Keyword Difficulty Checker1 Appends the extracted `keyword` and `keywordDifficultyIndex` to Google Sheets.      |
| SERP Results               | Google Sheets       | Appends SERP results to Google Sheet              | Reformat 2                | None                           | 📄 SERP Results Appends the SERP results to the "backlinks" sheet in Google Sheets.                               |
| Sticky Note                | Sticky Note         | Documentation block                               | None                      | None                           | # 📊 Keyword Difficulty Checker Workflow for SEO Analysis ... (full note content as per Sticky Note node)          |
| Sticky Note1               | Sticky Note         | Documentation for form submission node            | None                      | None                           | 📝 On form submission Collects user-submitted data (`keyword` and `country`) via a form trigger.                   |
| Sticky Note2               | Sticky Note         | Documentation for API request node                 | None                      | None                           | 🌐 Keyword Difficulty Checker Sends the submitted data to an external API (`keywordDifficulty.php`) using POST.    |
| Sticky Note3               | Sticky Note         | Documentation for Reformat 1                        | None                      | None                           | 📦 Reformat 1 Extracts the `keywordDifficulty` value from the API response JSON.                                   |
| Sticky Note4               | Sticky Note         | Documentation for Google Sheets append difficulty | None                      | None                           | 📊 Keyword Difficulty Checker1 Appends the extracted `keyword` and `keywordDifficultyIndex` to Google Sheets.      |
| Sticky Note5               | Sticky Note         | Documentation for Reformat 2                        | None                      | None                           | 📦 Reformat 2 Extracts the `serpResults` data array from the API response JSON.                                    |
| Sticky Note6               | Sticky Note         | Documentation for Google Sheets append SERP results| None                     | None                           | 📄 SERP Results Appends the SERP results to the "backlinks" sheet in Google Sheets.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure form title: "Keyword Difficulty Checker"  
   - Add two required fields: `keyword` and `country`  
   - Save and copy the webhook URL for external form integration  

2. **Create "Keyword Difficulty Checker" node**  
   - Type: HTTP Request  
   - Set method to POST  
   - URL: `https://keyword-difficulty-checker1.p.rapidapi.com/keywordDifficulty.php`  
   - Content Type: multipart-form-data  
   - Body Parameters:  
     - `keyword` = expression: `{{$json.keyword}}` from form node  
     - `country` = expression: `{{$json.country}}` from form node  
   - Header Parameters:  
     - `x-rapidapi-host` = `keyword-difficulty-checker1.p.rapidapi.com`  
     - `x-rapidapi-key` = your valid RapidAPI key (must be set in credentials or directly)  
   - Connect output of "On form submission" to this node  

3. **Create "Reformat 1" node**  
   - Type: Code (JavaScript)  
   - JS code: `return $input.first().json.data.semrushAPI.keywordDifficulty;`  
   - Connect output of "Keyword Difficulty Checker" to this node  

4. **Create "Keyword Difficulty Checker1" node**  
   - Type: Google Sheets  
   - Operation: Append  
   - Google Sheets Document ID: Set your target spreadsheet ID (must be accessible by service account)  
   - Sheet Name: Use sheet with GID `4590546` or actual sheet name as per your setup  
   - Map columns for `keyword` and `keywordDifficultyIndex` (make sure input JSON keys match these)  
   - Authentication: Use a Service Account credential with write access  
   - Connect output of "Reformat 1" to this node  

5. **Create "Reformat 2" node**  
   - Type: Code (JavaScript)  
   - JS code: `return $input.first().json.data.semrushAPI.serpResults;`  
   - Connect output of "Keyword Difficulty Checker" to this node  

6. **Create "SERP Results" node**  
   - Type: Google Sheets  
   - Operation: Append  
   - Google Sheets Document ID: `1BL8gZNhmaiZVThuEIFFra0XNQnnNrkxTsgEKeRoRHFM` (or your own)  
   - Sheet Name: Use `gid=0` or first sheet as per your spreadsheet  
   - Map columns automatically for the serpResults array fields  
   - Authentication: Use same Service Account credential  
   - Connect output of "Reformat 2" to this node  

7. **Validate credential access and permissions** for Google Sheets and RapidAPI before running  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| 📊 Keyword Difficulty Checker Workflow for SEO Analysis: Automates SEO keyword research by collecting difficulty scores and SERP data and storing them in Google Sheets. Useful for SEO strategy and reporting.                     | Workflow description (Sticky Note)  |
| ⚠️ Ensure your RapidAPI key is valid and has permissions for `keywordDifficulty.php` endpoint.                                                                                                                                      | API Authentication Note             |
| 📝 Use Google Service Account credentials with Editor access to the target Google Sheets documents to avoid permission errors.                                                                                                     | Google Sheets Credentials           |
| For more about n8n Google Sheets node setup and service account authentication, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                                                                | n8n Docs                           |
| The workflow requires n8n version supporting Form Trigger node v2.2 and Google Sheets node v4.6 or later.                                                                                                                         | Version compatibility               |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. All data processed is legal and public, respecting current content policies without including any illegal, offensive, or protected elements.