Import CSV from URL to Google Sheets

https://n8nworkflows.xyz/workflows/import-csv-from-url-to-google-sheets-1969


# Import CSV from URL to Google Sheets

### 1. Workflow Overview

This workflow automates the import of COVID-19 testing data from a publicly available CSV file hosted at a specified URL and updates a Google Sheets document with this data. It is designed for manual execution and processes the data to focus on a specific regional subset (DACH countries for 2023), optimizing for Google Sheets API rate limits.

Logical blocks included:

- **1.1 Manual Trigger:** Starts the workflow manually.
- **1.2 Data Acquisition:** Downloads the CSV file from the remote URL.
- **1.3 CSV Parsing:** Converts the downloaded CSV file into JSON format.
- **1.4 Data Enrichment:** Adds a unique composite key to each data record.
- **1.5 Data Filtering:** Filters records to include only those from Germany (DE), Austria (AT), and Switzerland (CH) for the year 2023.
- **1.6 Data Upload:** Appends or updates the filtered data into a specified Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:** This block initiates the workflow manually when the user clicks "Execute Workflow".
- **Nodes Involved:** 
  - When clicking "Execute Workflow"
- **Node Details:**
  - **Type and Role:** Manual Trigger node; triggers the workflow on user command.
  - **Configuration:** No parameters; default manual trigger.
  - **Expressions/Variables:** None.
  - **Input/Output:** No inputs; outputs trigger to the next node "Download CSV".
  - **Version:** 1.
  - **Edge Cases:** None expected; user must manually start workflow.
  - **Sub-workflow:** None.

#### 1.2 Data Acquisition

- **Overview:** Downloads the CSV file from the external URL.
- **Nodes Involved:** 
  - Download CSV
- **Node Details:**
  - **Type and Role:** HTTP Request node; performs a HTTP GET request to download the CSV file.
  - **Configuration:** 
    - URL: `https://opendata.ecdc.europa.eu/covid19/testing/csv/data.csv`
    - Response format: File (binary).
  - **Expressions/Variables:** Static URL.
  - **Input/Output:** Input from manual trigger; output is binary CSV file to "Import CSV".
  - **Version:** 4.1.
  - **Edge Cases:** Possible HTTP errors (404, 500), network timeouts, or response not being a valid file.
  - **Sub-workflow:** None.

#### 1.3 CSV Parsing

- **Overview:** Converts the downloaded CSV binary file into structured JSON data.
- **Nodes Involved:** 
  - Import CSV
- **Node Details:**
  - **Type and Role:** Spreadsheet File node; imports CSV content into JSON.
  - **Configuration:** 
    - File format set to CSV.
    - Options specify that the first row is a header row.
  - **Expressions/Variables:** None.
  - **Input/Output:** Input: CSV file from "Download CSV"; output: JSON data to "Add unique field".
  - **Version:** 2.
  - **Edge Cases:** Malformed CSV, missing headers, empty file.
  - **Sub-workflow:** None.

#### 1.4 Data Enrichment

- **Overview:** Creates a unique key for each record by concatenating 'country_code' and 'year_week' fields.
- **Nodes Involved:** 
  - Add unique field
- **Node Details:**
  - **Type and Role:** Set node; adds or modifies fields in JSON data.
  - **Configuration:** 
    - Adds a new field `unique_key` with value constructed as `{{$json.country_code}}-{{$json.year_week}}`.
  - **Expressions/Variables:** Expression used to concatenate two JSON fields.
  - **Input/Output:** Input from "Import CSV"; output to "Keep only DACH in 2023".
  - **Version:** 3.
  - **Edge Cases:** Missing or empty `country_code` or `year_week` fields may result in incomplete keys.
  - **Sub-workflow:** None.

#### 1.5 Data Filtering

- **Overview:** Filters the enriched dataset to keep only records for Germany, Austria, and Switzerland in 2023.
- **Nodes Involved:** 
  - Keep only DACH in 2023
- **Node Details:**
  - **Type and Role:** Filter node; applies multiple conditions to filter dataset.
  - **Configuration:** 
    - String condition: `year_week` starts with "2023".
    - Boolean condition: `country_code` is in the set ['DE', 'AT', 'CH'].
  - **Expressions/Variables:** 
    - Expression to check `year_week` prefix.
    - Expression to check inclusion in country codes.
  - **Input/Output:** Input from "Add unique field"; output to "Upload to spreadsheet".
  - **Version:** 1.
  - **Edge Cases:** Records missing these fields will be filtered out; if dataset is empty after filtering, subsequent upload will be empty.
  - **Sub-workflow:** None.
  - **Sticky Note:** Warns about Google API rate-limits and recommends batch processing with delay nodes for large datasets.

#### 1.6 Data Upload

- **Overview:** Appends or updates filtered data into Google Sheets using a unique key for matching rows.
- **Nodes Involved:** 
  - Upload to spreadsheet
- **Node Details:**
  - **Type and Role:** Google Sheets node; writes data to a Google Sheets document.
  - **Configuration:** 
    - Operation: `appendOrUpdate`.
    - Document specified by URL: `https://docs.google.com/spreadsheets/d/13YYuEJ1cDf-t8P2MSTFWnnNHCreQ6Zo8oPSp7WeNnbY`.
    - Sheet ID (gid): 383583634.
    - Columns schema is explicitly defined, including `unique_key` and all other CSV fields.
    - Matching column: `unique_key` (used to update existing rows or append new).
    - Cell format set to "USER_ENTERED" to allow Google Sheets to interpret values (e.g., dates, numbers).
  - **Expressions/Variables:** None dynamic; document and sheet ID are static.
  - **Input/Output:** Input filtered JSON from "Keep only DACH in 2023".
  - **Version:** 4.
  - **Credentials:** Uses Google Sheets OAuth2 credentials.
  - **Edge Cases:** Google Sheets API rate limits, permission errors, document or sheet not found, schema mismatch.
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role            | Input Node(s)                 | Output Node(s)            | Sticky Note                                                                                         |
|----------------------------|----------------------|----------------------------|------------------------------|---------------------------|---------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger       | Workflow start trigger     | -                            | Download CSV              |                                                                                                   |
| Download CSV               | HTTP Request         | Downloads CSV file          | When clicking "Execute Workflow" | Import CSV               |                                                                                                   |
| Import CSV                | Spreadsheet File     | Parses CSV to JSON          | Download CSV                 | Add unique field          |                                                                                                   |
| Add unique field           | Set                  | Adds composite unique key   | Import CSV                   | Keep only DACH in 2023    |                                                                                                   |
| Keep only DACH in 2023     | Filter               | Filters data by country & year | Add unique field            | Upload to spreadsheet     | Google API has rate-limits for read and write operations, that's why we take only a subset of data. To import the whole dataset please add Split In Batches and a Wait node with a sufficient delay. |
| Upload to spreadsheet      | Google Sheets        | Uploads data to Google Sheets | Keep only DACH in 2023       | -                         |                                                                                                   |
| Sticky Note                | Sticky Note          | Informational note          | -                            | -                         | Google API has rate-limits for read and write operations, that's why we take only a subset of the data. To import the whole dataset please add Split In Batches and a Wait node with a sufficient delay. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a node of type **Manual Trigger** (`n8n-nodes-base.manualTrigger`).  
   - Leave default parameters. This node will start the workflow manually.

2. **Add HTTP Request Node to Download CSV**  
   - Add node **HTTP Request** (`n8n-nodes-base.httpRequest`).  
   - Configure:  
     - HTTP Method: GET  
     - URL: `https://opendata.ecdc.europa.eu/covid19/testing/csv/data.csv`  
     - Response Format: File (enable binary response)  
   - Connect output of Manual Trigger to this node.

3. **Add Spreadsheet File Node to Import CSV**  
   - Add node **Spreadsheet File** (`n8n-nodes-base.spreadsheetFile`).  
   - Configure:  
     - File Format: CSV  
     - Options: Enable "Header row" (first row contains column headers)  
   - Connect output of HTTP Request node to this node.  
   - Use the binary file data output from the HTTP Request as input.

4. **Add Set Node to Create Unique Key**  
   - Add node **Set** (`n8n-nodes-base.set`).  
   - Configure:  
     - Add a new field named `unique_key`  
     - Value expression: `={{ $json.country_code + "-" + $json.year_week }}`  
   - Connect output of Spreadsheet File node to this node.

5. **Add Filter Node to Keep Only DACH Countries in 2023**  
   - Add node **Filter** (`n8n-nodes-base.filter`).  
   - Configure conditions:  
     - String condition: `year_week` starts with `"2023"`  
     - Boolean condition: `country_code` is included in array `['DE', 'AT', 'CH']`  
       Use expression: `={{ ['DE', 'AT', 'CH'].includes($json.country_code) }}`  
   - Connect output of Set node to this node.

6. **Add Google Sheets Node to Upload Data**  
   - Add node **Google Sheets** (`n8n-nodes-base.googleSheets`).  
   - Configure:  
     - Operation: `appendOrUpdate`  
     - Document ID: Use the URL `https://docs.google.com/spreadsheets/d/13YYuEJ1cDf-t8P2MSTFWnnNHCreQ6Zo8oPSp7WeNnbY` (n8n extracts the ID internally)  
     - Sheet: Select sheet by ID `383583634` or sheet name if preferred ("COVID-weekly")  
     - Columns: Define columns schema explicitly matching the CSV fields plus the new `unique_key`. Include: `unique_key`, `country`, `country_code`, `year_week`, `level`, `region`, `region_name`, `new_cases`, `tests_done`, `population`, `testing_rate`, `positivity_rate`, `testing_data_source`.  
     - Matching columns: `unique_key` (to update existing rows)  
     - Cell format: USER_ENTERED (allows Google Sheets to parse numbers, dates)  
   - Connect output of Filter node to this node.  
   - Set Google Sheets OAuth2 credentials before executing.

7. **(Optional) Add Sticky Note for Documentation**  
   - Add a sticky note describing Google API rate limits and recommending batch processing with wait nodes for large datasets.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Google API has rate-limits for read and write operations, that's why only a subset of data is imported. To import the whole dataset, add Split In Batches and a Wait node with a sufficient delay. | Sticky note attached to "Keep only DACH in 2023" node                                                          |
| The CSV source is updated weekly by ECDC and covers COVID-19 testing data for European countries.              | Source URL: https://opendata.ecdc.europa.eu/covid19/testing/csv/data.csv                                        |
| Ensure Google Sheets OAuth2 credentials have proper scopes to read/write the targeted spreadsheet.            | Credential setup required for "Upload to spreadsheet" node                                                     |

---

This documentation enables comprehensive understanding, reproduction, and modification of the workflow without reference to the original JSON. It highlights potential failure points and provides best practice notes for handling API rate limits.