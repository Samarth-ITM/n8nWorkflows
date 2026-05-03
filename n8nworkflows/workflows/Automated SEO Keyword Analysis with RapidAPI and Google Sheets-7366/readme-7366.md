Automated SEO Keyword Analysis with RapidAPI and Google Sheets

https://n8nworkflows.xyz/workflows/automated-seo-keyword-analysis-with-rapidapi-and-google-sheets-7366


# Automated SEO Keyword Analysis with RapidAPI and Google Sheets

### 1. Workflow Overview

This workflow automates an SEO keyword analysis process by collecting keyword and country inputs via a form, querying a RapidAPI SEO service for broad match keywords, keyword difficulty, and SERP (Search Engine Results Page) data, then formatting and saving these results into a Google Sheets document. It is designed to simplify on-page SEO research by integrating data retrieval and storage into an automated flow.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Capture keyword and country from a web form submission.
- **1.2 Data Storage:** Store input variables internally for consistent reuse.
- **1.3 API Requests:** Send POST requests to RapidAPI endpoints for keyword insights and difficulty data.
- **1.4 Data Extraction and Formatting:** Extract relevant data arrays or objects from the API responses.
- **1.5 Data Saving:** Append processed data into specific sheets in a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Captures user inputs for keyword and country from a web form submission to start the workflow.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: `formTrigger`  
  - Configuration: Listens for submissions to a form titled "OnPage SEO ( Keyword)" with mandatory fields `keyword` and `country`.  
  - Expressions/Variables: Captures `$json.keyword` and `$json.country` from incoming form data.  
  - Input: External (web form submission webhook)  
  - Output: Passes captured data downstream to the "Global Storage" node.  
  - Edge Cases: Missing required fields will prevent trigger; webhook availability and authentication must be ensured.

#### 1.2 Data Storage

**Overview:**  
Stores the keyword and country values from the form to ensure consistent reference throughout the workflow.

**Nodes Involved:**  
- Global Storage

**Node Details:**  
- **Global Storage**  
  - Type: `set`  
  - Configuration: Stores two string variables — `keyword` and `country` — assigned from incoming JSON fields.  
  - Expressions: `={{ $json.keyword }}`, `={{ $json.country }}`  
  - Input: From "On form submission"  
  - Output: Splits flow to two HTTP Request nodes for API data retrieval.  
  - Edge Cases: Data type coercion errors unlikely if inputs are valid strings.

#### 1.3 API Requests

**Overview:**  
Fetches keyword data and SEO metrics from RapidAPI endpoints via two POST requests.

**Nodes Involved:**  
- Keyword Insights Request  
- KeyWord Difficulty Request

**Node Details:**  
- **Keyword Insights Request**  
  - Type: `httpRequest`  
  - Configuration: POSTs to `https://seo-on-page.p.rapidapi.com/keyword-tool.php` with multipart-form-data body including `keyword` and `country`.  
  - Headers: `x-rapidapi-host: seo-on-page.p.rapidapi.com`, `x-rapidapi-key: your key` (replace with valid key)  
  - Expressions: Body parameters use `={{ $json.keyword }}` and `={{ $json.country }}`  
  - Input: From "Global Storage"  
  - Output: Passes response JSON to "Re-Format" for broad match keywords extraction.  
  - Edge Cases: API key invalid, rate limiting, network errors, malformed responses.

- **KeyWord Difficulty Request**  
  - Type: `httpRequest`  
  - Configuration: POSTs to `https://seo-on-page.p.rapidapi.com/keywordDifficulty.php` with similar multipart-form-data parameters.  
  - Headers and expressions same as above.  
  - Input: From "Global Storage"  
  - Output: Passes response JSON to two re-formatting code nodes to extract difficulty index and SERP results.  
  - Edge Cases: Same as above, plus handling of response structure variations.

#### 1.4 Data Extraction and Formatting

**Overview:**  
Processes API responses to extract relevant data subsets for storage.

**Nodes Involved:**  
- Re-Format  
- Re-Format 2  
- Re -Format 5

**Node Details:**  
- **Re-Format**  
  - Type: `code`  
  - Configuration: JavaScript code returning `data.semrushAPI.broadMatchKeywords` from the keyword insights response.  
  - Input: From "Keyword Insights Request"  
  - Output: Sends extracted broad match keywords array to "Keyword Insights" Google Sheets node.  
  - Edge Cases: Missing or malformed `broadMatchKeywords` field causes runtime errors.

- **Re-Format 2**  
  - Type: `code`  
  - Configuration: JavaScript code returning the first element of `data.semrushAPI.keywordDifficulty` array from difficulty API response.  
  - Input: From "KeyWord Difficulty Request"  
  - Output: Sends difficulty index data to "KeyWord Difficulty" Google Sheets node.  
  - Edge Cases: Empty or missing array leads to undefined errors.

- **Re -Format 5**  
  - Type: `code`  
  - Configuration: JavaScript code returning `data.semrushAPI.serpResults` array from difficulty API response.  
  - Input: From "KeyWord Difficulty Request"  
  - Output: Sends SERP results data to "SERP Result" Google Sheets node.  
  - Edge Cases: Missing `serpResults` field or empty data.

#### 1.5 Data Saving

**Overview:**  
Appends the processed data arrays or objects into respective sheets within a Google Sheets document for record and further analysis.

**Nodes Involved:**  
- Keyword Insights  
- KeyWord Difficulty  
- SERP Result

**Node Details:**  
- **Keyword Insights**  
  - Type: `googleSheets`  
  - Operation: Append rows  
  - Target Sheet: "Keyword Insights" (gid=0) in document ID `1dCSO-gv_mPD3O0QACeJaBFtjYGqtF-iF2_6iCuvm_Xw`  
  - Authentication: Google Service Account  
  - Input: From "Re-Format" (broad match keywords)  
  - Edge Cases: Authentication failures, Google Sheets API quota limits, schema mismatch.

- **KeyWord Difficulty**  
  - Type: `googleSheets`  
  - Operation: Append rows  
  - Target Sheet: "KeyWord Difficulty" (gid=1445611850) in same document  
  - Columns schema includes `keyword` and `keywordDifficultyIndex` strings.  
  - Input: From "Re-Format 2" (difficulty data)  
  - Authentication: Google Service Account  
  - Edge Cases: Same as above.

- **SERP Result**  
  - Type: `googleSheets`  
  - Operation: Append rows  
  - Target Sheet: "Serp Analytics" (gid=684053301) in same document  
  - Input: From "Re -Format 5" (SERP data)  
  - Authentication: Google Service Account  
  - Edge Cases: Same as above.

---

### 3. Summary Table

| Node Name               | Node Type       | Functional Role                          | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                         |
|-------------------------|-----------------|----------------------------------------|-------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------|
| On form submission      | formTrigger     | Trigger on keyword/country form submit | External webhook               | Global Storage                    | ### 1. 🟢 **On form submission** - Triggers when a user submits the form with `keyword` and `country`. |
| Global Storage          | set             | Store keyword and country variables     | On form submission            | Keyword Insights Request, KeyWord Difficulty Request | ### 2. 📦 **Global Storage** - Stores the keyword and country values from the form.                 |
| Keyword Insights Request | httpRequest     | Fetch broad match keyword insights      | Global Storage                | Re-Format                        | ### 3. 🌐 **Keyword Insights Request** - Sends POST to RapidAPI `keyword-tool.php`.                |
| KeyWord Difficulty Request | httpRequest | Fetch keyword difficulty and SERP data  | Global Storage                | Re-Format 2, Re -Format 5        | ### 4. 🌐 **KeyWord Difficulty Request** - Sends POST to RapidAPI `keywordDifficulty.php`.         |
| Re-Format               | code            | Extract broadMatchKeywords array         | Keyword Insights Request      | Keyword Insights                 | ### 5. 🧾 **Re-Format** - Extracts `broadMatchKeywords` from API response.                         |
| Keyword Insights         | googleSheets    | Append broad match keywords to sheet    | Re-Format                    | —                               | ### 6. 📊 **Keyword Insights** - Appends broad match keyword data to "Keyword Insights" sheet.    |
| Re-Format 2             | code            | Extract keywordDifficultyIndex value     | KeyWord Difficulty Request   | KeyWord Difficulty              | ### 7. 🧮 **Re-Format 2** - Extracts `keywordDifficultyIndex` from API response.                   |
| KeyWord Difficulty       | googleSheets    | Append keyword difficulty data to sheet | Re-Format 2                  | —                               | ### 8. 📈 **KeyWord Difficulty** - Appends difficulty index to "KeyWord Difficulty" sheet.        |
| Re -Format 5            | code            | Extract serpResults array                 | KeyWord Difficulty Request   | SERP Result                    | ### 9. 🗂️ **Re -Format 5** - Extracts `serpResults` from API response.                           |
| SERP Result              | googleSheets    | Append SERP analytics data to sheet      | Re -Format 5                 | —                               | ### 10. 🔍 **SERP Result** - Appends SERP data to "Serp Analytics" sheet.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("On form submission")**  
   - Type: `formTrigger` (version 2.2)  
   - Configure with form title "OnPage SEO ( Keyword)"  
   - Add required fields:  
     - `keyword` (string, required)  
     - `country` (string, required)  
   - Set webhook to listen for submissions.

2. **Create a Set Node ("Global Storage")**  
   - Type: `set` (version 3.4)  
   - Add two string fields:  
     - `keyword` set to `={{ $json.keyword }}`  
     - `country` set to `={{ $json.country }}`  
   - Connect "On form submission" output to this node’s input.

3. **Create HTTP Request Node ("Keyword Insights Request")**  
   - Type: `httpRequest` (version 4.2)  
   - Method: POST  
   - URL: `https://seo-on-page.p.rapidapi.com/keyword-tool.php`  
   - Content Type: multipart-form-data  
   - Body Parameters (form-data):  
     - `keyword` = `={{ $json.keyword }}`  
     - `country` = `={{ $json.country }}`  
   - Headers:  
     - `x-rapidapi-host` = `seo-on-page.p.rapidapi.com`  
     - `x-rapidapi-key` = *Your RapidAPI key here*  
   - Connect from "Global Storage".

4. **Create HTTP Request Node ("KeyWord Difficulty Request")**  
   - Type: `httpRequest` (version 4.2)  
   - Method: POST  
   - URL: `https://seo-on-page.p.rapidapi.com/keywordDifficulty.php`  
   - Content Type: multipart-form-data  
   - Body Parameters: same as above  
   - Headers: same as above  
   - Connect from "Global Storage".

5. **Create Code Node ("Re-Format")**  
   - Type: `code` (version 2)  
   - JavaScript code:  
     ```js
     return $input.first().json.data.semrushAPI.broadMatchKeywords;
     ```  
   - Connect output of "Keyword Insights Request" to this node.

6. **Create Google Sheets Node ("Keyword Insights")**  
   - Type: `googleSheets` (version 4.6)  
   - Operation: Append  
   - Document ID: `1dCSO-gv_mPD3O0QACeJaBFtjYGqtF-iF2_6iCuvm_Xw`  
   - Sheet Name: Use GID 0 or name "Keyword Insights"  
   - Authentication: Use Google Service Account credentials configured for API access  
   - Connect output of "Re-Format" to this node.

7. **Create Code Node ("Re-Format 2")**  
   - Type: `code` (version 2)  
   - JavaScript code:  
     ```js
     return $input.first().json.data.semrushAPI.keywordDifficulty[0];
     ```  
   - Connect output of "KeyWord Difficulty Request" to this node.

8. **Create Google Sheets Node ("KeyWord Difficulty")**  
   - Type: `googleSheets` (version 4.6)  
   - Operation: Append  
   - Document ID: same as above  
   - Sheet Name: Use GID 1445611850 or name "KeyWord Difficulty"  
   - Define columns schema including `keyword` and `keywordDifficultyIndex` as strings  
   - Authentication: same Google Service Account  
   - Connect output of "Re-Format 2" to this node.

9. **Create Code Node ("Re -Format 5")**  
   - Type: `code` (version 2)  
   - JavaScript code:  
     ```js
     return $input.first().json.data.semrushAPI.serpResults;
     ```  
   - Connect output of "KeyWord Difficulty Request" to this node.

10. **Create Google Sheets Node ("SERP Result")**  
    - Type: `googleSheets` (version 4.6)  
    - Operation: Append  
    - Document ID: same as above  
    - Sheet Name: Use GID 684053301 or name "Serp Analytics"  
    - Authentication: same Google Service Account  
    - Connect output of "Re -Format 5" to this node.

**Credential Setup Notes:**  
- Create and configure a Google Service Account credential with access to the target Google Sheets document.  
- Obtain valid RapidAPI key for the SEO On-Page API and insert into the HTTP Request headers.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow automates keyword SEO analysis combining RapidAPI SEO endpoints and Google Sheets for data storage.         | Workflow description embedded in the large sticky note node.                                                  |
| Replace `"your key"` in HTTP Request headers with your actual RapidAPI key for seamless API access.                      | Required for API authentication and avoiding request failures.                                                |
| Google Sheets document used: https://docs.google.com/spreadsheets/d/1dCSO-gv_mPD3O0QACeJaBFtjYGqtF-iF2_6iCuvm_Xw       | Contains sheets "Keyword Insights," "KeyWord Difficulty," and "Serp Analytics" for data organization.           |
| Ensure formTrigger webhook is properly deployed and accessible to receive form submissions externally.                   | Critical for workflow initiation.                                                                             |
| API responses assume presence of nested `data.semrushAPI` object with specific keys (`broadMatchKeywords`, etc.)          | Errors may occur if API changes or data is missing; validation or error handling can improve robustness.       |

---

**Disclaimer:** The provided text is exclusively sourced from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.