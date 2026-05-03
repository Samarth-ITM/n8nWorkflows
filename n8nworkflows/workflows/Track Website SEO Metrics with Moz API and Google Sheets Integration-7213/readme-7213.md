Track Website SEO Metrics with Moz API and Google Sheets Integration

https://n8nworkflows.xyz/workflows/track-website-seo-metrics-with-moz-api-and-google-sheets-integration-7213


# Track Website SEO Metrics with Moz API and Google Sheets Integration

### 1. Workflow Overview

This workflow automates the process of tracking important SEO metrics for websites by integrating a user input form, Moz API data retrieval, and Google Sheets storage. It is designed for digital marketers, SEO specialists, and webmasters who want to efficiently collect and log Domain Authority (DA), Page Authority (PA), Domain Rating (DR), Spam Score, and Organic Traffic metrics for websites submitted via a form.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures website URLs submitted through a web form.
- **1.2 SEO Metrics Retrieval:** Sends the URL to a Moz API via RapidAPI to fetch SEO-related metrics.
- **1.3 Validation and Filtering:** Checks if the API call was successful before processing data.
- **1.4 Data Cleaning:** Extracts and formats relevant SEO data from the API response.
- **1.5 Data Storage:** Appends the cleaned data into a Google Sheets document for record-keeping and analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user submits a form containing a website URL labeled "website". It collects the input to initiate the SEO metrics check.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  **On form submission**  
  - Type: `formTrigger`  
  - Role: Entry point; listens for form submissions with website URLs.  
  - Configuration: Form titled "DA PA Checker" with a single required field `website`. No additional options set.  
  - Expressions: None; outputs the submitted JSON including the `website` field.  
  - Input: External HTTP form submission webhook  
  - Output: JSON containing `{ "website": <submitted_url> }`  
  - Version: 2.2  
  - Potential failures: Missing or malformed website input, webhook availability issues.

#### 2.2 SEO Metrics Retrieval

- **Overview:**  
  This block sends a POST request to the Moz DA/PA API through RapidAPI, passing the submitted website URL and retrieving SEO metrics.

- **Nodes Involved:**  
  - DA PA API Request

- **Node Details:**  

  **DA PA API Request**  
  - Type: `httpRequest`  
  - Role: Sends POST request to external Moz API to fetch SEO metrics.  
  - Configuration:  
    - URL: `https://moz-da-pa-checker.p.rapidapi.com/dapa.php`  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Body: Includes a single parameter `website` with value from form submission (`{{$json.website}}`).  
    - Headers: Includes RapidAPI host and API key (requires user to set their own key).  
  - Expressions: Uses expression to inject `website` from previous node.  
  - Input: JSON with `website` field from form trigger  
  - Output: JSON response from API, expected to contain `success` boolean and `data` object with SEO metrics.  
  - Version: 4.2  
  - Potential failures: API key invalid or missing, network errors, API rate limiting, malformed responses.

#### 2.3 Validation and Filtering

- **Overview:**  
  Checks the API response for a `success` flag to ensure data was fetched correctly before proceeding.

- **Nodes Involved:**  
  - If

- **Node Details:**  

  **If**  
  - Type: `if`  
  - Role: Conditional gate; allows workflow continuation only if the API response indicates success.  
  - Configuration: Condition checks if `success === true` in the JSON response (`{{$json.success}}`).  
  - Input: API response JSON  
  - Output: Two branches - `true` branch continues with data processing, `false` branch stops or can be configured for error handling.  
  - Version: 2.2  
  - Potential failures: Missing `success` field, unexpected response structure.

#### 2.4 Data Cleaning

- **Overview:**  
  Extracts the `data` property from the API response, which contains the key SEO metrics, preparing the data for insertion into Google Sheets.

- **Nodes Involved:**  
  - Clean Output

- **Node Details:**  

  **Clean Output**  
  - Type: `code` (JavaScript)  
  - Role: Data transformation; extracts only the relevant `data` section from the API response.  
  - Configuration: JavaScript code `return $input.first().json.data;` which returns the `data` object of the first item.  
  - Input: API response JSON from the If node (filtered for success).  
  - Output: JSON with only the SEO metrics fields (e.g., da, pa, dr, spam_score, org_traffic).  
  - Version: 2  
  - Potential failures: If `data` field is missing or malformed, code could fail.

#### 2.5 Data Storage

- **Overview:**  
  Appends the cleaned SEO metrics along with the original website URL into a specified Google Sheets document.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  

  **Google Sheets**  
  - Type: `googleSheets`  
  - Role: Data storage; appends a new row with SEO metrics into Google Sheets.  
  - Configuration:  
    - Operation: Append  
    - Document ID: `1fR7VFM-vxVcJLc96ESXstah_4U4qJeiRRjZFk_xhL9g`  
    - Sheet Name: `Sheet1` (gid=0)  
    - Columns mapped:  
      - website: from `On form submission` node’s website input  
      - da, pa, dr, spam score, organic traffic: from cleaned API data  
    - Authentication: Service account credentials linked (`Google Docs account`)  
    - Mapping mode: Manually defined mapping with matching column `website`  
  - Input: Cleaned SEO metrics JSON + website URL  
  - Output: Success confirmation or error response from Google Sheets API  
  - Version: 4.6  
  - Potential failures: Authentication failure, sheet access permissions, API quota exceeded, invalid document or sheet ID.

---

### 3. Summary Table

| Node Name           | Node Type        | Functional Role                        | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                                   |
|---------------------|------------------|-------------------------------------|-----------------------|----------------------|-------------------------------------------------------------------------------------------------------------------------------|
| On form submission   | formTrigger      | Trigger workflow on form submission | External webhook      | DA PA API Request     | ## 1. On form submission<br>Triggers the workflow when a form is submitted. Collects the `website` field as input.            |
| DA PA API Request    | httpRequest      | Fetch SEO metrics from Moz API       | On form submission    | If                   | ## 2. DA PA API Request<br>Sends the website to the Moz DA/PA API via RapidAPI. Receives domain metrics in the response.      |
| If                  | if               | Check API success flag               | DA PA API Request     | Clean Output          | ## 3. If<br>Checks if the API response `success` is true. Continues only when data was fetched successfully.                   |
| Clean Output        | code             | Extract relevant SEO data            | If                    | Google Sheets         | ## 4. Clean Output<br>Extracts only the `data` section from the API result. Prepares clean key-value pairs for saving.         |
| Google Sheets        | googleSheets     | Append SEO data to Google Sheets     | Clean Output          | None                  | ## 5. Google Sheets<br>Appends the cleaned data into a Google Sheet. Stores website, DA, PA, spam score, DR, and organic traffic.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the `On form submission` node:**  
   - Type: `formTrigger`  
   - Set form title to "DA PA Checker"  
   - Add a required field named `website` (string)  
   - No additional options needed  
   - This node acts as the webhook trigger.

3. **Add the `DA PA API Request` node:**  
   - Type: `httpRequest`  
   - Set HTTP Method to `POST`  
   - Set URL to `https://moz-da-pa-checker.p.rapidapi.com/dapa.php`  
   - Content-Type: `multipart/form-data`  
   - In body parameters, add `website` parameter with expression `{{$json["website"]}}` from the form trigger node  
   - Add headers:  
     - `x-rapidapi-host`: `moz-da-pa-checker.p.rapidapi.com`  
     - `x-rapidapi-key`: your RapidAPI key (replace `"your key"`)  
   - Connect output of `On form submission` to this node.

4. **Add the `If` node:**  
   - Type: `if`  
   - Condition: Check if `{{$json.success}}` equals `true` (boolean)  
   - Connect output of `DA PA API Request` to this node.

5. **Add the `Clean Output` node:**  
   - Type: `code` (JavaScript)  
   - Code: `return $input.first().json.data;`  
   - Connect `true` output of `If` node to this node.

6. **Add the `Google Sheets` node:**  
   - Type: `googleSheets`  
   - Operation: Append  
   - Document ID: `1fR7VFM-vxVcJLc96ESXstah_4U4qJeiRRjZFk_xhL9g` (replace with your own if necessary)  
   - Sheet Name: `Sheet1` or gid=0  
   - Authentication: Use service account credentials configured in n8n (named "Google Docs account")  
   - Map columns as follows:  
     - `website`: `{{$node["On form submission"].json["website"]}}`  
     - `da`: `{{$json.da}}`  
     - `pa`: `{{$json.pa}}`  
     - `spam score`: `{{$json.spam_score}}`  
     - `dr`: `{{$json.dr}}`  
     - `organic traffic`: `{{$json.org_traffic}}`  
   - Connect output of `Clean Output` to this node.

7. **Test the workflow:**  
   - Submit a website URL via the form URL generated by the `On form submission` node.  
   - Check Google Sheets for appended data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow uses the Moz DA/PA API via RapidAPI; users must obtain their own RapidAPI key and set it in the `DA PA API Request` node header `x-rapidapi-key`.                                              | https://rapidapi.com/                                                                            |
| Google Sheets node requires service account credentials with access to the target spreadsheet. Ensure the Google Sheet is shared with the service account email.                                              | n8n Google Docs integration docs                                                                |
| Form submissions trigger webhooks; ensure the n8n instance is accessible via public URL or tunneling service for external form submissions to reach the workflow.                                            |                                                                                                |
| Useful for SEO specialists to automate bulk domain authority checks and maintain historical records in Google Sheets.                                                                                       |                                                                                                |
| Full description and explanation are embedded inside a sticky note within the workflow for quick reference during editing.                                                                                   |                                                                                                |

---

**Disclaimer:** The content provided is generated from an automated n8n workflow and strictly complies with content policies. All data processed is legal and publicly accessible.