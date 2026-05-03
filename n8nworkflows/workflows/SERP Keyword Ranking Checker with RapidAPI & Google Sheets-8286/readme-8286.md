SERP Keyword Ranking Checker with RapidAPI & Google Sheets

https://n8nworkflows.xyz/workflows/serp-keyword-ranking-checker-with-rapidapi---google-sheets-8286


# SERP Keyword Ranking Checker with RapidAPI & Google Sheets

---

### 1. Workflow Overview

This workflow automates the process of checking keyword rankings in Google Search Engine Results Pages (SERP) using a RapidAPI SERP Keyword Ranking Checker API and logs the results into Google Sheets. It is designed for SEO professionals, digital marketers, and analysts who want to track keyword visibility across different countries.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures user input via a form with fields for a keyword and country code.
- **1.2 Data Preparation:** Stores the input data for use across the workflow.
- **1.3 API Request to SERP Keyword Ranking Checker:** Sends the keyword and country to the RapidAPI endpoint to fetch SERP data.
- **1.4 Conditional Evaluation:** Checks if the API returned valid SERP results.
- **1.5 Data Logging:** Based on the condition, either logs the retrieved SERP data or logs a "No result found" message into Google Sheets.
- **1.6 Flow Control:** Wait nodes introduce delays to manage flow timing and API pacing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives user input from a web form submission containing the keyword and country code for the SERP check.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - *Type & Role:* `formTrigger` node; triggers the workflow when the form is submitted.  
    - *Configuration:* Form titled "SERP Keyword Ranking Checker" with two required fields: "keyword" (e.g., "labubu") and "Country" (e.g., "us").  
    - *Expressions:* Outputs JSON with properties `keyword` and `Country` based on user input.  
    - *Connections:* Outputs to `Global Storage` node.  
    - *Edge Cases:* Form validation ensures required fields are present; no input means the workflow won't start.  
    - *Version:* 2.2

#### 2.2 Data Preparation

- **Overview:**  
  Stores the input keyword and country in workflow variables for consistent access downstream.

- **Nodes Involved:**  
  - Global Storage

- **Node Details:**

  - **Global Storage**  
    - *Type & Role:* `set` node; stores input values into named variables.  
    - *Configuration:* Assigns `keyword` as `{{$json.keyword}}` and `country` as `{{$json.Country}}`.  
    - *Expressions:* Uses expressions to map from input JSON.  
    - *Connections:* Receives input from `On form submission`; outputs to `SERP Keyword Ranking Checker`.  
    - *Edge Cases:* If input is missing or malformed, variables may be empty, potentially causing downstream API errors.  
    - *Version:* 3.4

#### 2.3 API Request to SERP Keyword Ranking Checker

- **Overview:**  
  Sends a POST request with the keyword and country to the RapidAPI SERP Keyword Ranking Checker endpoint to fetch keyword ranking data.

- **Nodes Involved:**  
  - SERP Keyword Ranking Checker

- **Node Details:**

  - **SERP Keyword Ranking Checker**  
    - *Type & Role:* `httpRequest` node; makes HTTP POST request to the RapidAPI endpoint.  
    - *Configuration:*  
      - URL: `https://rapidapi.com/PrineshPatel/api/serp-keyword-ranking-checker`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameters: `keyword` and `country` from stored variables.  
      - Headers: includes `x-rapidapi-host` and `x-rapidapi-key` (user must supply their own API key).  
    - *Expressions:* Uses `={{ $json.keyword }}` and `={{ $json.country }}` for body parameters.  
    - *Connections:* Receives input from `Global Storage`; outputs to `If` node.  
    - *Edge Cases:*  
      - Invalid API key or quota exceeded leads to authentication errors.  
      - Network issues or API downtime may cause timeouts.  
      - Unexpected API response structure may cause downstream expression failures.  
    - *Version:* 4.2

#### 2.4 Conditional Evaluation

- **Overview:**  
  Evaluates whether the API response contains non-empty SERP results. This determines which branch the workflow follows next.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - *Type & Role:* `if` node; conditional branching based on data presence.  
    - *Configuration:* Checks if `data.semrushAPI.serpResults` in the JSON response is an array and not empty.  
    - *Expressions:* Uses expression `={{ $json.data.semrushAPI.serpResults }}` with `notEmpty` operation.  
    - *Connections:* Receives from `SERP Keyword Ranking Checker`.  
      - True branch connects to `Wait1`.  
      - False branch connects to `Wait`.  
    - *Edge Cases:*  
      - If the response has unexpected structure or missing `serpResults`, might evaluate incorrectly.  
      - Case sensitivity and strict type validation enabled reduce false positives.  
    - *Version:* 2.2

#### 2.5 Flow Control and Data Logging

- **Overview:**  
  Introduces timing delays before logging data to Google Sheets. Depending on the condition, either actual SERP data or a "No result found" message is appended to the sheet.

- **Nodes Involved:**  
  - Wait1  
  - Google Sheets  
  - Wait  
  - Google Sheets1

- **Node Details:**

  - **Wait1**  
    - *Type & Role:* `wait` node; delays workflow for a fixed interval (default 5 seconds).  
    - *Configuration:* No parameters explicitly set; default delay (typically 5 seconds).  
    - *Connections:* Receives true branch from `If`; outputs to `Google Sheets`.  
    - *Edge Cases:* If wait is interrupted or times out, the workflow may halt.  
    - *Version:* 1.1

  - **Google Sheets (true branch)**  
    - *Type & Role:* `googleSheets` node; appends SERP data to Google Sheet.  
    - *Configuration:*  
      - Operation: Append row  
      - Sheet name: "Sheet1" (gid=0)  
      - Document ID: must be set by user  
      - Columns: `Country`, `Keyword`, and `Json data` (mapped from response `data.semrushAPI.serpResults`)  
      - Authentication: Service Account with configured Google API credentials  
    - *Expressions:* Uses expressions like `={{ $('Global Storage').item.json.country }}`, `={{ $('Global Storage').item.json.keyword }}`, and `{{$json.data.semrushAPI.serpResults}}`.  
    - *Connections:* Input from `Wait1`.  
    - *Edge Cases:*  
      - Invalid or missing credentials cause auth errors.  
      - Sheet or document ID misconfiguration prevents data writing.  
      - Large JSON data may cause API limits or formatting issues.  
    - *Version:* 4.6

  - **Wait**  
    - *Type & Role:* `wait` node; delays workflow before logging no-result message.  
    - *Configuration:* Default delay, similar to `Wait1`.  
    - *Connections:* Receives false branch from `If`; outputs to `Google Sheets1`.  
    - *Edge Cases:* Same as `Wait1`.  
    - *Version:* 1.1

  - **Google Sheets1 (false branch)**  
    - *Type & Role:* `googleSheets` node; appends a "No result found. Please try another keyword..." message to Google Sheet when no API data is returned.  
    - *Configuration:*  
      - Operation: Append row  
      - Sheet name: "Sheet1" (gid=0)  
      - Document ID: must be set by user  
      - Columns: `Country`, `Keyword`, and static string for `Json data` column  
      - Authentication: Service Account with Google API credentials  
    - *Expressions:* `={{ $('Global Storage').item.json.country }}`, `={{ $('Global Storage').item.json.keyword }}`, and a fixed message for `Json data`.  
    - *Connections:* Input from `Wait`.  
    - *Edge Cases:* Same as `Google Sheets` node; also ensures no data scenarios are logged for audit.  
    - *Version:* 4.6

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                                  | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                                                |
|-----------------------------|---------------------|-------------------------------------------------|---------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission           | formTrigger         | Captures user keyword and country input          | —                         | Global Storage           | 1. 🟢 **On form submission**: Triggers when form submitted with Keyword & Country fields.                                                                  |
| Global Storage              | set                 | Stores input values for use in later nodes       | On form submission         | SERP Keyword Ranking Checker | 2. 📦 **Global Storage**: Stores input values for later use.                                                                                                |
| SERP Keyword Ranking Checker | httpRequest         | Sends API request to RapidAPI to fetch SERP data | Global Storage             | If                       | 3. 🌐 **SERP Keyword Ranking Checker**: POST request to RapidAPI's SERP Keyword Ranking Checker endpoint.                                                  |
| If                          | if                  | Checks if API response contains valid SERP data | SERP Keyword Ranking Checker | Wait1 (true), Wait (false) | 4. ⚖️ **Condition Checking**: Evaluates if response contains non-empty data to decide next step.                                                           |
| Wait1                       | wait                | Delays flow before logging data                   | If (true branch)           | Google Sheets             | 7. ⏳ **Wait Node (5-Second Interval)**: Introduces delay before proceeding.                                                                                |
| Google Sheets               | googleSheets        | Appends valid SERP response data to Google Sheet | Wait1                      | —                        | 6. 📊 **Google Sheets - Insert Record from Response Body**: Logs response data into Google Sheets.                                                         |
| Wait                        | wait                | Delays flow before logging "No result" message   | If (false branch)          | Google Sheets1            | 5. ⏳ **Wait Node (5-Second Interval)**: Introduces delay before proceeding.                                                                                |
| Google Sheets1              | googleSheets        | Appends "No result found" message to Google Sheet| Wait                       | —                        | 8. 📊 **Google Sheets - Insert 'No result found' Record**: Logs message when no data found in API response.                                                |
| Sticky Note1 to Sticky Note8| stickyNote          | Informational comments attached to various nodes | —                         | —                        | See detailed notes in Section 2.                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add `formTrigger` node:**
   - Name: "On form submission"
   - Configuration:
     - Form Title: "SERP Keyword Ranking Checker"
     - Form Description: "Enter your keyword for research the market."
     - Form Fields:
       - Field 1: Label = "keyword", Placeholder = "e.g. labubu", Required = true
       - Field 2: Label = "Country", Placeholder = "e.g. us", Required = true
   - This node triggers on form submission.

3. **Add `set` node:**
   - Name: "Global Storage"
   - Connect the output of "On form submission" to this node.
   - Assign two variables:
     - `keyword` = `{{$json.keyword}}`
     - `country` = `{{$json.Country}}`

4. **Add `httpRequest` node:**
   - Name: "SERP Keyword Ranking Checker"
   - Connect from "Global Storage".
   - Configuration:
     - URL: `https://rapidapi.com/PrineshPatel/api/serp-keyword-ranking-checker`
     - Method: POST
     - Content-Type: multipart/form-data
     - Send Body: true
     - Body Parameters:
       - `keyword` = `={{ $json.keyword }}`
       - `country` = `={{ $json.country }}`
     - Header Parameters:
       - `x-rapidapi-host` = `serp-keyword-ranking-checker.p.rapidapi.com`
       - `x-rapidapi-key` = *Your RapidAPI key here* (must be replaced with valid key)
   - Ensure no additional options are set.

5. **Add `if` node:**
   - Name: "If"
   - Connect from "SERP Keyword Ranking Checker"
   - Condition:
     - Left Value: `={{ $json.data.semrushAPI.serpResults }}`
     - Operator: `notEmpty` (array operation)
     - Right Value: empty
   - Set combinator to "and" with strict type validation and case sensitive.

6. **Add two `wait` nodes:**

   - "Wait1":
     - Connect from "If" node's true branch.
     - Use default wait time (5 seconds).

   - "Wait":
     - Connect from "If" node's false branch.
     - Use default wait time (5 seconds).

7. **Add two `googleSheets` nodes:**

   - "Google Sheets" (true branch):
     - Connect from "Wait1".
     - Operation: Append
     - Sheet Name: "Sheet1" (gid=0)
     - Document ID: *Set your Google Sheet ID*
     - Columns mapping:
       - `Country` = `={{ $('Global Storage').item.json.country }}`
       - `Keyword` = `={{ $('Global Storage').item.json.keyword }}`
       - `Json data` = `={{ $json.data.semrushAPI.serpResults }}`
     - Authentication: Use Google API credentials (Service Account)

   - "Google Sheets1" (false branch):
     - Connect from "Wait".
     - Operation: Append
     - Sheet Name: "Sheet1" (gid=0)
     - Document ID: *Set your Google Sheet ID*
     - Columns mapping:
       - `Country` = `={{ $('Global Storage').item.json.country }}`
       - `Keyword` = `={{ $('Global Storage').item.json.keyword }}`
       - `Json data` = `=No result found. Please try another keyword...`
     - Authentication: Same Google API credentials as above.

8. **Configure credentials:**
   - For the Google Sheets nodes, set up Google API service account credentials with appropriate permissions to append rows to the target sheet.
   - For the HTTP Request node, add your RapidAPI key under header parameters.

9. **Test the workflow:**
   - Submit the form with valid `keyword` and `country`.
   - Confirm that data is logged correctly in Google Sheets.
   - Test with keywords that yield no results to verify logging of "No result found" message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                  | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow automates keyword SERP ranking lookups and logs data into Google Sheets, ideal for SEO and marketing professionals.                                                              | Workflow description and use case.                                                                              |
| Uses RapidAPI’s SERP Keyword Ranking Checker API by PrineshPatel: https://rapidapi.com/PrineshPatel/api/serp-keyword-ranking-checker                                                            | API documentation and access.                                                                                    |
| Requires valid Google Sheets document ID and service account credentials with write permissions to append data to sheets.                                                                     | Google Sheets API setup.                                                                                         |
| Introduces 5-second wait nodes to avoid hitting API rate limits or overwhelming Google Sheets API.                                                                                            | Flow control best practices in API integrations.                                                                |
| FormTrigger node requires form creation and deployment in n8n to capture user inputs.                                                                                                          | n8n formTrigger node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.formTrigger/                        |
| Ensure to replace "your key" in the HTTP request headers with an active RapidAPI key to avoid authentication errors.                                                                           | RapidAPI key management.                                                                                         |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created with n8n, a no-code automation tool. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.