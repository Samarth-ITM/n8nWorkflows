Google SERP + Trends and Recommendations with Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/google-serp---trends-and-recommendations-with-bright-data---google-gemini-4861


# Google SERP + Trends and Recommendations with Bright Data & Google Gemini

### 1. Workflow Overview

This workflow automates the process of tracking Google Search Engine Results Pages (SERP) for a given query, extracting trends and recommendations related to the query, and storing the results as CSV files. It integrates Bright Data for web scraping, Google Gemini LLM (PaLM API) for structured data extraction and analysis, and processes the data into structured JSON formats for trends and recommendations.

The workflow consists of the following logical blocks:

- **1.1 Input Setup & Web Request**: Accepts manual trigger input, sets parameters including search query and Bright Data zone, and performs a web request to Google SERP via Bright Data proxy.
- **1.2 Google SERP Data Extraction**: Uses the Google Gemini LLM to analyze raw SERP data and extracts structured search results.
- **1.3 Data Loop and Parallel Analysis**: Iterates over each extracted search result item to run two parallel analyses:
  - **1.3.1 Trends Extraction**: Extracts current trends from the search result.
  - **1.3.2 Recommendations Extraction**: Generates recommendations based on the search result.
- **1.4 Data Post-Processing & Storage**: Converts trends and recommendations into files and writes them as CSVs to disk for further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Setup & Web Request

**Overview:**  
This block accepts manual input to define search parameters, constructs the request, and sends it to Bright Data’s API to scrape Google SERP.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set input fields  
- Perform Bright Data Web Request  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starting point to manually initiate the workflow.  
  - Configuration: No parameters; triggers flow on manual user action.  
  - Input/Output: No input; outputs trigger to "Set input fields".  
  - Edge Cases: Trigger not fired if user does not manually execute.  

- **Set input fields**  
  - Type: Set  
  - Role: Defines critical input parameters for the workflow.  
  - Configuration: Sets the following variables:  
    - `url`: Google Search URL (https://www.google.com/search)  
    - `webhook_notification_url`: URL to receive webhook notifications (used for testing)  
    - `search`: The query string ("best crm for the year 2025")  
    - `zone`: Bright Data proxy zone ("web_unlocker1")  
  - Key Expressions: Static string assignments.  
  - Input: Trigger from manual node.  
  - Output: Passes JSON with parameters to Bright Data request node.  
  - Edge Cases: Missing or incorrect zone or URL parameters will cause request failure.  

- **Perform Bright Data Web Request**  
  - Type: HTTP Request  
  - Role: Makes a POST request to Bright Data’s API to scrape the Google SERP page with the given search query, using the specified proxy zone.  
  - Configuration:  
    - URL: `https://api.brightdata.com/request`  
    - Method: POST  
    - Body: JSON including `zone`, `url` with encoded search query parameter, and `format` set to "raw"  
    - Headers: Content-Type: application/json  
    - Authentication: HTTP Header Auth with Bright Data API token  
  - Input: Receives search parameters from "Set input fields".  
  - Output: Raw SERP HTML/text data to next node.  
  - Edge Cases: Authentication failure, network timeout, invalid zone, or malformed query could cause request failure.  

---

#### 1.2 Google SERP Data Extraction

**Overview:**  
Processes the raw SERP response using Google Gemini LLM to extract structured data such as rank, title, URL, snippet, and type for each search result.

**Nodes Involved:**  
- Google Search Data Extractor (LangChain LLM Chain)  
- Code  
- Loop Over Items  

**Node Details:**

- **Google Search Data Extractor**  
  - Type: LangChain Chain LLM  
  - Role: Parses raw SERP data and extracts structured search results using a custom prompt.  
  - Configuration:  
    - Prompt: Extract rank, title, URL, snippet, and type (organic/ads/map) from raw SERP data. Return JSON.  
    - Output Parser: Enabled for structured output validation.  
    - Retry on Failure: Enabled for robustness.  
  - Input: Raw SERP response from Bright Data HTTP Request.  
  - Output: JSON with array of search results.  
  - Edge Cases: Model may fail to parse if input format is unexpected or if API limits are reached.  

- **Code (JavaScript)**  
  - Type: Code  
  - Role: Extracts the `results` array from the LLM output for further processing.  
  - Configuration: Returns `$input.first().json.output.results` to flatten data structure.  
  - Input: LLM chain output.  
  - Output: Array of search results.  
  - Edge Cases: If output format changes or is empty, this node could error or return empty array.  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each search result item individually to process trends and recommendations in parallel.  
  - Configuration: Default batching options.  
  - Input: Array of search result items.  
  - Output: Single item per iteration to two parallel branches.  
  - Edge Cases: Empty input array results in no iterations.  

---

#### 1.3 Data Loop and Parallel Analysis

**Overview:**  
For each search result item, two parallel LLM analyses are performed: one to extract trends and another to generate recommendations based on the title and snippet.

**Nodes Involved:**  
- Trends Data Extractor (LangChain Chain LLM)  
- Google Gemini Chat Model for Trend Data  
- Structured Output Parser for Trend Data  
- Code for Trends  
- Recommendation Data Extractor (LangChain Chain LLM)  
- Google Gemini Chat Model for Recommendation  
- Structured Output Parser for Recommendation  
- Code for Recommendations  

**Node Details:**

- **Trends Data Extractor**  
  - Type: LangChain Chain LLM  
  - Role: Extracts trend information from the title and snippet of each search result.  
  - Configuration: Prompt asks to extract trends as JSON from title and snippet.  
  - Input: Single search result item (title & snippet).  
  - Output: JSON with `trends` array.  
  - Edge Cases: Model errors, or poorly formatted input can cause parsing failures.  

- **Google Gemini Chat Model for Trend Data**  
  - Type: LangChain Google Gemini Chat LLM  
  - Role: Provides the underlying language model for the trends extraction chain.  
  - Configuration: Model name "models/gemini-2.0-flash-exp"  
  - Credentials: Google PaLM API credentials required.  
  - Input/Output: Connected as AI language model integration for trends extractor node.  
  - Edge Cases: API quota limits, authentication errors, or network instability.  

- **Structured Output Parser for Trend Data**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates and parses trends output into JSON schema format.  
  - Configuration: JSON schema defines an array of trend objects with `trend` and `description` fields.  
  - Input: Raw LLM trends output.  
  - Output: Validated JSON structure.  
  - Edge Cases: Schema violations or incomplete data cause parse errors.  

- **Code for Trends**  
  - Type: Code  
  - Role: Extracts the `trends` property from the structured output for file conversion.  
  - Configuration: Returns `$input.first().json.output.trends`  
  - Input: Structured output parser for trends.  
  - Output: Trends array.  
  - Edge Cases: Missing output or unexpected structure.  

- **Recommendation Data Extractor**  
  - Type: LangChain Chain LLM  
  - Role: Generates recommendations based on the title and snippet of the search result.  
  - Configuration: Prompt asks for recommendations in JSON format with specified fields.  
  - Input: Single search result item.  
  - Output: JSON with `recommendations` array.  
  - Edge Cases: Same as trends extractor regarding parsing and model output.  

- **Google Gemini Chat Model for Recommendation**  
  - Type: LangChain Google Gemini Chat LLM  
  - Role: Underlying model for recommendation extraction.  
  - Configuration: Same model and credentials as trends model.  
  - Edge Cases: Same as trends model.  

- **Structured Output Parser for Recommendation**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates recommendations against a schema defining types like Software, Action, etc.  
  - Input: Raw LLM recommendation output.  
  - Output: Validated recommendations JSON.  
  - Edge Cases: Schema misalignment or incomplete data.  

- **Code for Recommendations**  
  - Type: Code  
  - Role: Extracts `recommendations` from the structured output parser.  
  - Configuration: Returns `$input.first().json.output.recommendations`  
  - Edge Cases: Unexpected input structure.  

---

#### 1.4 Data Post-Processing & Storage

**Overview:**  
Converts the extracted trends and recommendations into file format and writes them as CSV files to disk with timestamped filenames.

**Nodes Involved:**  
- Convert to File for Trends  
- Write the trends csv file to disk  
- Convert to File for Recommendations  
- Write the recommendations csv file to disk  

**Node Details:**

- **Convert to File for Trends**  
  - Type: ConvertToFile  
  - Role: Converts trends JSON data into a file (likely CSV or JSON format) for storage.  
  - Input: Trends array from code node.  
  - Output: File data object for writing.  
  - Edge Cases: Data format inconsistencies may cause conversion errors.  

- **Write the trends csv file to disk**  
  - Type: ReadWriteFile  
  - Role: Writes the trends file to disk under path `d:\Google_SERP_Trends_Response_<timestamp>.csv`  
  - Configuration: Write operation with dynamic filename using current ISO timestamp.  
  - Input: File data from convert node.  
  - Edge Cases: File permission or disk space issues.  

- **Convert to File for Recommendations**  
  - Type: ConvertToFile  
  - Role: Converts recommendations JSON data into a file for saving.  
  - Input: Recommendations array.  
  - Output: File data object.  
  - Edge Cases: Same as trends file conversion.  

- **Write the recommendations csv file to disk**  
  - Type: ReadWriteFile  
  - Role: Writes the recommendations file to disk under path `d:\Google_SERP_Recommendations_Response_<timestamp>.csv`  
  - Configuration: Write operation with dynamic filename.  
  - Edge Cases: Same as trends file writing.  

---

### 3. Summary Table

| Node Name                          | Node Type                              | Functional Role                                  | Input Node(s)                       | Output Node(s)                         | Sticky Note                                                                                               |
|-----------------------------------|--------------------------------------|-------------------------------------------------|-----------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger                       | Workflow start trigger                           | -                                 | Set input fields                     |                                                                                                          |
| Set input fields                   | Set                                  | Define input parameters for search and proxy    | When clicking ‘Test workflow’      | Perform Bright Data Web Request      | ## Note Deals with the Google SERP Tracker by utilizing the Bright Data and Google Gemini LLM for transforming the profile into a structured JSON response. **Please make sure to set the input fields node with the filtering criteria, Bright Data zone name, Webhook notification URL** Test Webhook using - https://webhook.site/ |
| Perform Bright Data Web Request    | HTTP Request                        | Scrape Google SERP via Bright Data proxy         | Set input fields                   | Google Search Data Extractor         |                                                                                                          |
| Google Search Data Extractor       | LangChain Chain LLM                 | Extract structured SERP results from raw data    | Perform Bright Data Web Request    | Code                                | ## LLM Usages Google Gemini LLM is being utilized for the structured data extraction handling.           |
| Code                              | Code                                | Extract results array from LLM output            | Google Search Data Extractor       | Loop Over Items                     |                                                                                                          |
| Loop Over Items                   | SplitInBatches                      | Iterate over each search result item              | Code                              | Trends Data Extractor, Recommendation Data Extractor |                                                                                                          |
| Trends Data Extractor             | LangChain Chain LLM                 | Extract trends from search result item            | Loop Over Items                   | Code for Trends                    |                                                                                                          |
| Google Gemini Chat Model for Trend Data | LangChain Google Gemini Chat LLM  | Language model for trends extraction              | - (used as AI model)               | Trends Data Extractor               |                                                                                                          |
| Structured Output Parser for Trend Data | LangChain Structured Output Parser | Validate trends JSON output                        | Google Gemini Chat Model for Trend Data | Trends Data Extractor            |                                                                                                          |
| Code for Trends                  | Code                                | Extract trends array from structured output       | Structured Output Parser for Trend Data | Convert to File for Trends        |                                                                                                          |
| Convert to File for Trends       | ConvertToFile                      | Convert trends JSON to file format                 | Code for Trends                   | Write the trends csv file to disk   |                                                                                                          |
| Write the trends csv file to disk | ReadWriteFile                      | Save trends file to disk                            | Convert to File for Trends        | Loop Over Items                     |                                                                                                          |
| Recommendation Data Extractor     | LangChain Chain LLM                 | Extract recommendations from search result item   | Loop Over Items                   | Code for Recommendations          |                                                                                                          |
| Google Gemini Chat Model for Recommendation | LangChain Google Gemini Chat LLM  | Language model for recommendations extraction     | - (used as AI model)               | Recommendation Data Extractor       |                                                                                                          |
| Structured Output Parser for Recommendation | LangChain Structured Output Parser | Validate recommendations JSON output              | Google Gemini Chat Model for Recommendation | Recommendation Data Extractor  |                                                                                                          |
| Code for Recommendations          | Code                                | Extract recommendations array from structured output | Structured Output Parser for Recommendation | Convert to File for Recommendations |                                                                                                          |
| Convert to File for Recommendations | ConvertToFile                    | Convert recommendations JSON to file format       | Code for Recommendations          | Write the recommendations csv file to disk |                                                                                                          |
| Write the recommendations csv file to disk | ReadWriteFile                  | Save recommendations file to disk                   | Convert to File for Recommendations | -                                 |                                                                                                          |
| Sticky Note1                     | Sticky Note                        | Workflow general note and instructions            | -                                 | -                                 | ## Note Deals with the Google SERP Tracker by utilizing the Bright Data and Google Gemini LLM for transforming the profile into a structured JSON response. **Please make sure to set the input fields node with the filtering criteria, Bright Data zone name, Webhook notification URL** Test Webhook using - https://webhook.site/ |
| Sticky Note4                     | Sticky Note                        | Notes on LLM usage                                | -                                 | -                                 | ## LLM Usages Google Gemini LLM is being utilized for the structured data extraction handling.           |
| Sticky Note5                     | Sticky Note                        | Branding: Bright Data logo                         | -                                 | -                                 | ## Logo ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Type: Manual Trigger (no parameters)  

2. **Create Set Node for Input Fields**  
   - Name: "Set input fields"  
   - Assign values:  
     - `url`: `https://www.google.com/search`  
     - `webhook_notification_url`: `https://webhook.site/c9118da2-1c54-460f-a83a-e5131b7098db` (or your test webhook URL)  
     - `search`: `best crm for the year 2025`  
     - `zone`: `web_unlocker1` (Bright Data proxy zone)  
   - Connect from Manual Trigger node.  

3. **Create HTTP Request Node for Bright Data**  
   - Name: "Perform Bright Data Web Request"  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: HTTP Header Auth with Bright Data API token credentials  
   - Headers: Content-Type = application/json  
   - Body Parameters (JSON):  
     - `zone`: `={{ $json.zone }}`  
     - `url`: `={{ $json.url }}?q={{ encodeURI($json.search) }}`  
     - `format`: `raw`  
   - Connect from "Set input fields".  

4. **Create LangChain Chain LLM Node for Google SERP Extraction**  
   - Name: "Google Search Data Extractor"  
   - Model: Google Gemini via LangChain chain  
   - Prompt: Extract structured fields (rank, title, URL, snippet, type) from input raw SERP data. Return JSON.  
   - Enable Output Parser  
   - Retry on failure enabled  
   - Connect from HTTP Request node.  

5. **Create Code Node to Extract Results Array**  
   - Name: "Code"  
   - JavaScript: `return $input.first().json.output.results`  
   - Connect from "Google Search Data Extractor".  

6. **Create SplitInBatches Node to Loop Over Items**  
   - Name: "Loop Over Items"  
   - Default batching parameters  
   - Connect from "Code".  

7. **Create LangChain Chain LLM Node for Trends Extraction**  
   - Name: "Trends Data Extractor"  
   - Prompt: Extract trends from title and snippet fields, return JSON.  
   - Enable Output Parser  
   - Retry on failure  
   - Connect from first output of "Loop Over Items".  

8. **Create Google Gemini Chat Model Node for Trends**  
   - Name: "Google Gemini Chat Model for Trend Data"  
   - Model: "models/gemini-2.0-flash-exp"  
   - Credentials: Google PaLM API  
   - Connect as AI language model to "Trends Data Extractor".  

9. **Create Structured Output Parser Node for Trends**  
   - Name: "Structured Output Parser for Trend Data"  
   - Provide JSON schema defining `trends` array with `trend` and `description` fields  
   - Connect as AI output parser to "Trends Data Extractor".  

10. **Create Code Node for Trends Data Extraction**  
    - Name: "Code for Trends"  
    - JavaScript: `return $input.first().json.output.trends`  
    - Connect from "Structured Output Parser for Trend Data".  

11. **Create ConvertToFile Node for Trends**  
    - Name: "Convert to File for Trends"  
    - Default parameters for conversion to CSV or JSON file  
    - Connect from "Code for Trends".  

12. **Create ReadWriteFile Node to Write Trends CSV**  
    - Name: "Write the trends csv file to disk"  
    - Operation: Write  
    - File Name: `d:\Google_SERP_Trends_Response_{{ new Date().toISOString().replace(/[:.]/g, '-')}}.csv`  
    - Connect from "Convert to File for Trends".  

13. **Connect "Write the trends csv file to disk" back to "Loop Over Items"**  
    - This allows the loop to continue processing other items.  

14. **Create LangChain Chain LLM Node for Recommendations Extraction**  
    - Name: "Recommendation Data Extractor"  
    - Prompt: Provide recommendations based on title and snippet, return JSON.  
    - Enable Output Parser  
    - Retry on failure  
    - Connect from second output of "Loop Over Items".  

15. **Create Google Gemini Chat Model Node for Recommendations**  
    - Name: "Google Gemini Chat Model for Recommendation"  
    - Model: "models/gemini-2.0-flash-exp"  
    - Credentials: Google PaLM API  
    - Connect as AI language model to "Recommendation Data Extractor".  

16. **Create Structured Output Parser for Recommendations**  
    - Name: "Structured Output Parser for Recommendation"  
    - Provide JSON schema defining `recommendations` array with fields like type, name, description, reason  
    - Connect as AI output parser to "Recommendation Data Extractor".  

17. **Create Code Node for Recommendations Extraction**  
    - Name: "Code for Recommendations"  
    - JavaScript: `return $input.first().json.output.recommendations`  
    - Connect from "Structured Output Parser for Recommendation".  

18. **Create ConvertToFile Node for Recommendations**  
    - Name: "Convert to File for Recommendations"  
    - Default parameters for file conversion  
    - Connect from "Code for Recommendations".  

19. **Create ReadWriteFile Node to Write Recommendations CSV**  
    - Name: "Write the recommendations csv file to disk"  
    - Operation: Write  
    - File Name: `d:\Google_SERP_Recommendations_Response_{{ new Date().toISOString().replace(/[:.]/g, '-')}}.csv`  
    - Connect from "Convert to File for Recommendations".  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                               |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------|
| Bright Data logo image: ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) | Branding in Sticky Note5                       |
| Webhook testing URL recommendation: https://webhook.site/                                    | Mentioned in Sticky Note1 for webhook testing |
| Workflow uses Google Gemini LLM (PaLM API) for structured data extraction and analysis.       | Sticky Note4                                   |
| Use Google PaLM API credentials for Google Gemini model nodes to enable AI capabilities.      | Credential requirement for LLM nodes          |
| File write paths use Windows format (drive `d:\`); adjust paths if deploying on other OSes.  | File write nodes configuration                 |

---

This comprehensive documentation provides a clear understanding of the workflow’s architecture, node-level details, and a stepwise guide to reproduce it in n8n. It anticipates common failure modes such as authentication issues, API limits, and data parsing errors, enabling efficient debugging and extension.