Flexible Currency Rate Uploads for SAP B1 with AI Validation & Multiple Sources

https://n8nworkflows.xyz/workflows/flexible-currency-rate-uploads-for-sap-b1-with-ai-validation---multiple-sources-4931


# Flexible Currency Rate Uploads for SAP B1 with AI Validation & Multiple Sources

### 1. Workflow Overview

This workflow, titled **Flexible Currency Rate Uploads for SAP B1 with AI Validation & Multiple Sources**, automates the process of uploading currency exchange rates into SAP Business One (SAP B1). It supports multiple data input sources—JSON payloads, SQL queries, Google Sheets, and manual entries—and leverages AI to normalize and validate incoming data, especially date formats. The workflow ensures robust error handling by logging successes and failures into Google Sheets for traceability.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and SAP Authentication:** Receives data via webhook and authenticates with SAP B1.
- **1.2 Source Routing:** Routes incoming data based on source type (JSON, SQL, Google Sheets, Manual).
- **1.3 Data Normalization and Splitting (JSON Source):** Uses OpenAI to normalize variable JSON structures, then splits data into individual items.
- **1.4 Data Retrieval (SQL Source):** Extracts and executes SQL queries against Microsoft SQL Server.
- **1.5 Data Retrieval (Google Sheets Source):** Fetches currency rates from Google Sheets.
- **1.6 AI Date Validation and Formatting:** Validates and standardizes the `RateDate` field via OpenAI.
- **1.7 SAP Upload:** Sends currency rate data to SAP B1 via HTTP API.
- **1.8 Success and Failure Logging:** Records successful and failed operations into Google Sheets logs.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and SAP Authentication

**Overview:**  
This block receives incoming data through a webhook and authenticates with SAP B1 to obtain a session token necessary for subsequent API calls.

**Nodes Involved:**  
- Webhook  
- Conectar SAP

**Node Details:**

- **Webhook**  
  - Type: `Webhook`  
  - Role: Entry point for external POST requests carrying currency rate data.  
  - Config: HTTP POST method, custom path.  
  - Inputs: External HTTP POST request.  
  - Outputs: JSON payload with data and metadata.  
  - Failure Points: Invalid payload format, unreachable webhook URL, network failure.

- **Conectar SAP**  
  - Type: `HTTP Request`  
  - Role: Authenticates with SAP B1 REST service, retrieves session ID for API calls.  
  - Config: POST request to SAP login endpoint using environment variables for username, password, and company database.  
  - Inputs: Triggered by webhook node.  
  - Outputs: JSON containing `SessionId`.  
  - Failure Points: Authentication errors, network issues, invalid SAP credentials.

---

#### 1.2 Source Routing

**Overview:**  
Determines the data source type from webhook input and routes the workflow accordingly. Supports JSON, SQL, Google Sheets, and Manual input.

**Nodes Involved:**  
- Switch

**Node Details:**

- **Switch**  
  - Type: `Switch`  
  - Role: Branches workflow based on `origen` field in webhook payload.  
  - Config:  
    - Cases:  
      - "JSON" → JSON data processing path  
      - "SQL" → SQL query execution path  
      - "GoogleSheets" → Google Sheets data path  
      - "Manual" → Manual data upload path  
  - Inputs: From `Conectar SAP` node.  
  - Outputs: One output per case routing to respective processing blocks.  
  - Failure Points: Missing or incorrect `origen` value, case sensitivity issues.

---

#### 1.3 Data Normalization and Splitting (JSON Source)

**Overview:**  
Processes flexible or irregular JSON structures by normalizing them into a uniform array of rate entries, then splits them into individual items for downstream processing.

**Nodes Involved:**  
- OpenAI (Normalize JSON)  
- Split Out  
- Enviar SAP (JSON)  
- Success  
- Fallo

**Node Details:**

- **OpenAI (Normalize JSON)**  
  - Type: `OpenAI` node using LangChain integration  
  - Role: Converts variable JSON structures into a uniform JSON with a single array `rate`.  
  - Config: Uses GPT-4o-mini model; prompt explicitly requests JSON transformation without explanation or comments.  
  - Inputs: Raw JSON from webhook body.  
  - Outputs: Normalized JSON with `rate` array under `message.content.rate`.  
  - Failure Points: API rate limits, prompt misinterpretation, network issues.

- **Split Out**  
  - Type: `Split Out Items`  
  - Role: Splits array `message.content.rate` into individual workflow items.  
  - Inputs: Normalized JSON from OpenAI node.  
  - Outputs: Multiple items processed individually downstream.  
  - Failure Points: Empty or malformed array.

- **Enviar SAP (JSON)**  
  - Type: `HTTP Request`  
  - Role: Sends individual currency rate entries to SAP B1 API.  
  - Config: POST to `SBOBobService_SetCurrencyRate` SAP endpoint with JSON body containing `Currency`, `Rate`, `RateDate`.  
  - Headers: Includes SAP session cookie.  
  - Inputs: Individual rate items from Split Out.  
  - Outputs: SAP API response; success and error handled downstream.  
  - Failure Points: SAP API errors, session expiration, JSON mapping errors.

- **Success**  
  - Type: `Google Sheets` (append operation)  
  - Role: Logs successful SAP submissions.  
  - Inputs: Success output from SAP HTTP request.  
  - Outputs: None.  
  - Failure Points: Google Sheets API rate limits, credential issues.

- **Fallo**  
  - Type: `Google Sheets` (append operation)  
  - Role: Logs failed SAP submissions with error details.  
  - Inputs: Error output from SAP HTTP request.  
  - Outputs: None.  
  - Failure Points: Google Sheets API errors.

---

#### 1.4 Data Retrieval (SQL Source)

**Overview:**  
Extracts a SQL query from webhook input, executes it on Microsoft SQL Server, limits the results, and processes each record by validating dates and sending to SAP.

**Nodes Involved:**  
- Extraer Query  
- Microsoft SQL  
- Limit  
- Comprobar Fecha  
- Enviar SAP (SQL)  
- Success1  
- Fallo1

**Node Details:**

- **Extraer Query**  
  - Type: `Set`  
  - Role: Extracts SQL query string from webhook payload into variable `select_sql`.  
  - Inputs: From Switch node.  
  - Outputs: SQL query string.  
  - Failure Points: Missing or malformed SQL query in payload.

- **Microsoft SQL**  
  - Type: `Microsoft SQL` node  
  - Role: Executes the extracted SQL query.  
  - Config: Uses Microsoft SQL credentials; executes query from `select_sql`.  
  - Inputs: SQL query from Extraer Query node.  
  - Outputs: Query result set as JSON.  
  - Failure Points: SQL syntax errors, connection issues.

- **Limit**  
  - Type: `Limit`  
  - Role: Restricts processing to 10 items maximum per execution to avoid overload.  
  - Inputs: SQL query results.  
  - Outputs: Limited dataset.  
  - Failure Points: None significant; misconfiguration could truncate needed data.

- **Comprobar Fecha**  
  - Type: `OpenAI` node  
  - Role: Validates and converts `RateDate` into `yyyyMMdd` format regardless of input format.  
  - Config: GPT-4o-mini model; prompt designed to convert multiple date formats strictly.  
  - Inputs: Individual rate records from Limit node.  
  - Outputs: JSON with corrected `RateDate`.  
  - Failure Points: API limits, ambiguous date formats.

- **Enviar SAP (SQL)**  
  - Type: `HTTP Request`  
  - Role: Sends corrected currency rate data to SAP B1 API.  
  - Config: POST request with fields from AI-validated JSON.  
  - Inputs: Validated JSON from Comprobar Fecha.  
  - Outputs: SAP API response for success or error.  
  - Failure Points: SAP API errors, session expiration.

- **Success1** & **Fallo1**  
  - Type: `Google Sheets` (append)  
  - Role: Log success and failure cases respectively for SQL source processing.  
  - Inputs: Success and error outputs from SAP HTTP request.  
  - Failure Points: Google Sheets API limits.

---

#### 1.5 Data Retrieval (Google Sheets Source)

**Overview:**  
Reads currency rate data from specified Google Sheets document and uploads each entry to SAP, logging outcomes.

**Nodes Involved:**  
- Google Sheets  
- Enviar SAP (SHEET)  
- Success2  
- Fallo2

**Node Details:**

- **Google Sheets (Read)**  
  - Type: `Google Sheets` node  
  - Role: Retrieves rows from a Google Sheets document and sheet specified by ID and GID.  
  - Config: Uses OAuth2 credentials; reads from sheet named "RATE".  
  - Inputs: Routed from Switch node on "GoogleSheets" case.  
  - Outputs: Rows as JSON objects.  
  - Failure Points: Credential expiration, API limits, incorrect sheet ID.

- **Enviar SAP (SHEET)**  
  - Same role and config as other `Enviar SAP` nodes, adapted for Google Sheets data.  
  - Inputs: Rows from Google Sheets node.  
  - Outputs: Response for success or error.  
  - Failure Points: SAP API failures.

- **Success2** & **Fallo2**  
  - Log success and failure for each SAP upload from Google Sheets data.  
  - Same structure as previous logging nodes.

---

#### 1.6 Data Upload (Manual Source)

**Overview:**  
Processes manually provided currency rate data from webhook and uploads it directly to SAP with logging.

**Nodes Involved:**  
- Enviar SAP (MANUAL)  
- Success3  
- Fallo3

**Node Details:**

- **Enviar SAP (MANUAL)**  
  - Type: `HTTP Request`  
  - Role: Sends manually entered currency data to SAP API.  
  - Inputs: Directly extracts fields `currency`, `rate`, `rateDate` from webhook body.  
  - Outputs: SAP response.  
  - Failure Points: Incorrect manual data formatting, session issues.

- **Success3** & **Fallo3**  
  - Log success and failure cases for manual uploads.  
  - Same Google Sheets append configuration as others.

---

### 3. Summary Table

| Node Name           | Node Type              | Functional Role                                | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                   |
|---------------------|------------------------|------------------------------------------------|-------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook             | Webhook                | Receives external POST requests                 | -                       | Conectar SAP                  |                                                                                                               |
| Conectar SAP        | HTTP Request           | Authenticates with SAP B1                       | Webhook                 | Switch                       |                                                                                                               |
| Switch              | Switch                 | Routes processing based on data source          | Conectar SAP            | OpenAI, Extraer Query, Google Sheets, Enviar SAP (MANUAL) |                                                                                                               |
| OpenAI              | OpenAI (LangChain)     | Normalizes variable JSON to uniform `rate` array| Switch (JSON path)      | Split Out                    |                                                                                                               |
| Split Out           | Split Out Items        | Splits normalized JSON into individual items    | OpenAI                  | Enviar SAP (JSON)             |                                                                                                               |
| Enviar SAP (JSON)   | HTTP Request           | Sends JSON currency rate data to SAP API        | Split Out               | Success, Fallo                |                                                                                                               |
| Success             | Google Sheets          | Logs success of JSON uploads                      | Enviar SAP (JSON)       | -                             |                                                                                                               |
| Fallo               | Google Sheets          | Logs failure of JSON uploads                      | Enviar SAP (JSON)       | -                             |                                                                                                               |
| Extraer Query       | Set                    | Extracts SQL query string from webhook payload   | Switch (SQL path)       | Microsoft SQL                 |                                                                                                               |
| Microsoft SQL       | Microsoft SQL          | Executes extracted SQL query                      | Extraer Query           | Limit                         |                                                                                                               |
| Limit               | Limit                  | Limits number of items processed                   | Microsoft SQL           | Comprobar Fecha               |                                                                                                               |
| Comprobar Fecha     | OpenAI (LangChain)     | Validates and reformats `RateDate` field          | Limit                   | Enviar SAP (SQL)              |                                                                                                               |
| Enviar SAP (SQL)    | HTTP Request           | Sends SQL-retrieved currency rate data to SAP    | Comprobar Fecha         | Success1, Fallo1              |                                                                                                               |
| Success1            | Google Sheets          | Logs success of SQL uploads                        | Enviar SAP (SQL)        | -                             |                                                                                                               |
| Fallo1              | Google Sheets          | Logs failure of SQL uploads                        | Enviar SAP (SQL)        | -                             |                                                                                                               |
| Google Sheets       | Google Sheets          | Reads currency rates from Google Sheets           | Switch (GoogleSheets)   | Enviar SAP (SHEET)            |                                                                                                               |
| Enviar SAP (SHEET)  | HTTP Request           | Sends Google Sheets currency data to SAP          | Google Sheets           | Success2, Fallo2              |                                                                                                               |
| Success2            | Google Sheets          | Logs success of Google Sheets uploads              | Enviar SAP (SHEET)      | -                             |                                                                                                               |
| Fallo2              | Google Sheets          | Logs failure of Google Sheets uploads              | Enviar SAP (SHEET)      | -                             |                                                                                                               |
| Enviar SAP (MANUAL) | HTTP Request           | Sends manually input currency rate data to SAP    | Switch (Manual path)    | Success3, Fallo3              |                                                                                                               |
| Success3            | Google Sheets          | Logs success of manual uploads                      | Enviar SAP (MANUAL)     | -                             |                                                                                                               |
| Fallo3              | Google Sheets          | Logs failure of manual uploads                      | Enviar SAP (MANUAL)     | -                             |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `b81a1fa5-2138-4b70-80ee-945fa44e69ce` (or custom)  
   - Purpose: Receive currency rate payloads.

2. **Create HTTP Request Node "Conectar SAP"**  
   - URL: `${{ $vars.url_sap }}Login`  
   - Method: POST  
   - Body (JSON):  
     ```json
     {
       "UserName": "{{ $vars.user_sap }}",
       "Password": "{{ $vars.password_sap }}",
       "CompanyDB": "{{ $vars.company_db }}"
     }
     ```  
   - Allow unauthorized certificates: true  
   - Connect output of Webhook to this node.

3. **Add Switch Node**  
   - Input: Output of "Conectar SAP"  
   - Property to check: `{{$json.body.origen}}`  
   - Cases:  
     - JSON  
     - SQL  
     - GoogleSheets  
     - Manual

4. **JSON Path: Add OpenAI Node "Normalize JSON"**  
   - Model: GPT-4o-mini  
   - Input prompt: Transform variable JSON into uniform JSON with `rate` array (include exact prompt content).  
   - Input: `{{$json.body.json}}` from webhook.  
   - Output JSON enabled.  
   - Connect Switch JSON output to OpenAI node.

5. **Add Split Out Items Node**  
   - Field to split out: `message.content.rate`  
   - Connect from OpenAI node.

6. **Add HTTP Request Node "Enviar SAP (JSON)"**  
   - URL: `${{ $vars.url_sap }}SBOBobService_SetCurrencyRate`  
   - Method: POST  
   - Body:  
     ```json
     {
       "Currency": "{{ $json.Currency }}",
       "Rate": "{{ $json.Rate }}",
       "RateDate": "{{ $json.RateDate }}"
     }
     ```  
   - Headers:  
     - Cookie: `B1SESSION={{ $('Conectar SAP').item.json.SessionId }}`  
   - Allow unauthorized certificates: true  
   - Connect Split Out to this node.

7. **Add Google Sheets Node "Success"**  
   - Operation: Append  
   - Document and Sheet: Use appropriate Google Sheets document and worksheet for log storage.  
   - Columns: Map success data including `workflow`, `method`, `url`, `json`, `status_code`, `message`.  
   - Connect Success output of "Enviar SAP (JSON)" to this node.

8. **Add Google Sheets Node "Fallo"**  
   - Same as Success, but maps error data from SAP API response.  
   - Connect Error output of "Enviar SAP (JSON)" to this node.

9. **SQL Path: Add Set Node "Extraer Query"**  
   - Assign `select_sql` variable with SQL query from webhook: `{{$json.body.sql}}`  
   - Connect Switch SQL output to this node.

10. **Add Microsoft SQL Node**  
    - Credentials: Microsoft SQL credentials configured.  
    - Operation: Execute query, input `select_sql`.  
    - Connect from Set node.

11. **Add Limit Node**  
    - Max items: 10  
    - Connect from Microsoft SQL node.

12. **Add OpenAI Node "Comprobar Fecha"**  
    - Model: GPT-4o-mini  
    - Prompt: Validate and convert `RateDate` into `yyyyMMdd` format (use exact prompt).  
    - Input JSON: Currency, Rate, RateDate from limited items.  
    - Output JSON enabled.  
    - Connect from Limit node.

13. **Add HTTP Request Node "Enviar SAP (SQL)"**  
    - Same config as "Enviar SAP (JSON)", but body fields come from AI-validated JSON.  
    - Connect from OpenAI node.

14. **Add Google Sheets Nodes "Success1" and "Fallo1"**  
    - Append success and error logs as done previously.  
    - Connect success and error outputs of "Enviar SAP (SQL)" node.

15. **Google Sheets Path: Add Google Sheets Read Node**  
    - Credentials: Google Sheets OAuth2.  
    - Document ID and Sheet Name: Use the document with currency rates.  
    - Connect Switch GoogleSheets output to this node.

16. **Add HTTP Request Node "Enviar SAP (SHEET)"**  
    - Same SAP API configuration as before.  
    - Input from Google Sheets rows.  
    - Connect Google Sheets node to this.

17. **Add Google Sheets Nodes "Success2" and "Fallo2"**  
    - Log success and failure for Google Sheets data uploads.  
    - Connect outputs from "Enviar SAP (SHEET)".

18. **Manual Path: Add HTTP Request Node "Enviar SAP (MANUAL)"**  
    - Body fields extracted directly from webhook body fields `currency`, `rate`, `rateDate`.  
    - Connect Switch Manual output to this node.

19. **Add Google Sheets Nodes "Success3" and "Fallo3"**  
    - Log success and failure for manual submissions.  
    - Connect outputs from "Enviar SAP (MANUAL)".

20. **Set all necessary environment variables:**  
    - `url_sap` (SAP B1 URL)  
    - `user_sap` (SAP username)  
    - `password_sap` (SAP password)  
    - `company_db` (SAP company database name)

21. **Configure credentials:**  
    - OpenAI API credentials  
    - Microsoft SQL credentials  
    - Google Sheets OAuth2 credentials

22. **Test each path independently with relevant sample payloads to ensure correctness.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                         | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow utilizes OpenAI GPT-4o-mini for data normalization and date validation, ensuring flexible and robust handling of variable input formats. | OpenAI LangChain integration in n8n.                                                                        |
| SAP B1 API endpoint `SBOBobService_SetCurrencyRate` is used for setting currency rates. Session management is via login and cookie.                  | SAP Business One Service Layer API documentation.                                                           |
| Google Sheets are used both as a data source and as a logging mechanism for successes and failures, enabling audit and traceability.                  | Google Sheets API and OAuth2 credentials setup in n8n.                                                      |
| The workflow limits SQL query results to 10 items per execution to avoid overload or timeouts.                                                        | Limit node usage to control batch size.                                                                     |
| For date normalization, AI is prompted to convert multiple date formats into a strict `yyyyMMdd` string to ensure SAP API compatibility.              | AI prompt carefully designed to handle various date input formats robustly.                                 |
| Error handling is performed by continuing on error in HTTP Request nodes, allowing logging nodes to capture and store failure details.                | See HTTP Request node setting `onError: continueErrorOutput`.                                               |

---

This documentation provides a complete structured understanding of the workflow, enabling reproduction, modification, and error anticipation for advanced users and AI agents alike.

---

**Disclaimer:** The provided content is derived solely from an n8n automation workflow. All data and operations comply with legal and ethical standards.