Fetch Live ETF Metrics from JustETF to Excel with One-Click Updates

https://n8nworkflows.xyz/workflows/fetch-live-etf-metrics-from-justetf-to-excel-with-one-click-updates-2519


# Fetch Live ETF Metrics from JustETF to Excel with One-Click Updates

### 1. Workflow Overview

This n8n workflow automates the process of fetching live ETF (Exchange-Traded Fund) metrics from the JustETF website and updating an Excel table with fresh, structured data upon demand. It is designed for investors and analysts who maintain an Excel workbook with ETF data and want to keep it continuously updated with minimal manual intervention.

The workflow is triggered via a webhook that is linked to an Excel macro button ("Update Table"). Once triggered, it performs the following logical blocks:

- **1.1 Trigger Reception:** Receives the webhook call initiated by the Excel macro button.
- **1.2 Excel Data Handling:** Updates a timestamp in the Excel table and retrieves all ETF rows from a specific table named "Div study".
- **1.3 Data Fetching from JustETF:** For each ETF ISIN code retrieved, sends an HTTP GET request to the JustETF profile page.
- **1.4 HTML Data Extraction:** Parses the returned HTML content to extract key ETF metrics such as dividends, fees, and 5-year performance using CSS selectors.
- **1.5 Data Formatting:** Processes and structures the extracted raw HTML data into clean numeric and textual values.
- **1.6 Excel Table Update:** Writes the cleaned data back into the Excel worksheet, matching on the ISIN to update existing rows.
- **1.7 Item Looping:** Loops over each ETF item to sequentially process multiple ETFs in batches.

This modular design allows for real-time updates of ETF metrics in Excel with a single click, supporting efficient portfolio monitoring and analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Reception

- **Overview:** Listens for a webhook call triggered by an Excel macro button. This is the entry point of the workflow.
- **Nodes Involved:**  
  - When called by Excel Macro (Webhook node)

- **Node Details:**  
  - Name: When called by Excel Macro  
  - Type: Webhook  
  - Configuration:  
    - Path: `/ETF`  
    - Triggered by HTTP POST (default webhook behavior)  
  - Input: External HTTP request from Excel macro  
  - Output: Triggers next node "Logs the date & time"  
  - Edge Cases: Webhook not reachable, incorrect path, Excel macro misconfiguration  
  - Version: n8n v2+ compatible

#### 2.2 Excel Data Handling

- **Overview:** Updates the “Last updated” timestamp in the Excel table and fetches all ETF rows from the "Div study" table.
- **Nodes Involved:**  
  - Logs the date & time (Microsoft Excel node)  
  - Gets rows from table (Microsoft Excel node)

- **Node Details:**  
  - Logs the date & time:  
    - Type: Microsoft Excel (Update operation on a table)  
    - Configuration: Updates the column "Dernière mise à jour" with current date/time in GMT-2 timezone, format `dd/mm/yyyy, HH:MM:SS` 24-hour  
    - Input: Trigger from webhook node  
    - Output: Feeds into "Gets rows from table"  
    - Edge Cases: Excel API auth issues, network errors, table/column missing  
  - Gets rows from table:  
    - Type: Microsoft Excel (Get Rows operation)  
    - Configuration: Retrieves all rows from "Div study" table in the configured workbook and worksheet  
    - Input: From "Logs the date & time"  
    - Output: Feeds into HTTP request node for each ISIN  
    - Edge Cases: Empty table, Excel API timeout, workbook/worksheet access issues

#### 2.3 Data Fetching from JustETF

- **Overview:** For each ETF ISIN, makes an HTTP GET request to JustETF to fetch the ETF profile HTML page.
- **Nodes Involved:**  
  - Forge a Get request with ISIN Values (HTTP Request node)

- **Node Details:**  
  - Type: HTTP Request (GET)  
  - Configuration: Constructs URL dynamically using ISIN, e.g., `https://www.justetf.com/fr/etf-profile.html?isin={{ $json.ISIN }}`  
  - Input: Rows from Excel with ISIN values  
  - Output: Raw HTML response of the ETF profile page  
  - Edge Cases: HTTP 404 if ISIN invalid, network errors, rate limiting by JustETF, malformed ISIN values

#### 2.4 HTML Data Extraction

- **Overview:** Parses the ETF profile HTML to extract specific data points using CSS selectors.
- **Nodes Involved:**  
  - Extracts defined values with css selector (HTML Extract node)

- **Node Details:**  
  - Type: HTML Extract  
  - Configuration: Extracts these keys with CSS selectors:  
    - Dividends  
    - Frais (Fees)  
    - Performance depuis 5 ans (5-year performance)  
    - Name (ETF name)  
  - Input: HTTP response HTML content  
  - Output: JSON with extracted text fields for further processing  
  - Edge Cases: Website structure changes breaking selectors, empty content, unexpected HTML format

#### 2.5 Data Formatting

- **Overview:** Processes raw extracted HTML text to parse dividend history, fees, performance, and ETF name into clean structured data.
- **Nodes Involved:**  
  - Extracts defined values in better format (Code node)  
  - Loop Over Items (Split in Batches node)  
  - Edit Fields (Set node)

- **Node Details:**  
  - Extracts defined values in better format:  
    - Type: Code (JavaScript)  
    - Logic: Uses regex to extract dividends per year, current distribution yield, fees, 5-year performance, and cleans ETF name  
    - Input: JSON from HTML Extract node  
    - Output: Structured JSON with fields: historicDividends array, performance5Years, rendementActuelDeDistribution, frais, nameOnly  
    - Edge Cases: Regex failures, missing data, unexpected text format  
  - Loop Over Items:  
    - Type: SplitInBatches  
    - Logic: Loops over each ETF item to process sequentially  
    - Input: Output array from code node  
    - Output: Passes single item to "Edit Fields"  
  - Edit Fields:  
    - Type: Set  
    - Logic: Converts string percentages and numbers to numeric values; maps extracted fields to Excel columns with proper data types  
    - Input: Single ETF structured JSON  
    - Output: Prepared JSON for Excel update  
    - Edge Cases: Parsing errors on numeric conversions, missing fields

#### 2.6 Excel Table Update

- **Overview:** Updates the original Excel worksheet rows with the freshly extracted and processed ETF metrics, matching rows on ISIN.
- **Nodes Involved:**  
  - Updates my table (Microsoft Excel node)

- **Node Details:**  
  - Type: Microsoft Excel (Update operation on worksheet)  
  - Configuration: Updates columns such as Frais, Rendement de départ, Performance depuis 5 ans, Dividende 12 mois, Dividende année précédente, etc. using values from "Edit Fields" node  
  - Matching: Uses ISIN column to match rows to update  
  - Input: Processed data from "Edit Fields" node  
  - Output: Final update confirmation  
  - Edge Cases: Row not found, Excel API errors, data type mismatches

---

### 3. Summary Table

| Node Name                         | Node Type                | Functional Role                      | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                                                                    |
|----------------------------------|--------------------------|------------------------------------|----------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When called by Excel Macro        | Webhook                  | Trigger entry from Excel macro     | -                          | Logs the date & time        |                                                                                                                                                |
| Logs the date & time              | Microsoft Excel          | Update timestamp in Excel table    | When called by Excel Macro  | Gets rows from table        | ### Excel data\n- start by logging the date and time of execution\n- Retrieve the rows of the table with the ETF ISIN\n- Forge a GET request |
| Gets rows from table              | Microsoft Excel          | Fetch all ETF rows from Excel      | Logs the date & time        | Forge a Get request with ISIN Values |                                                                                                                                               |
| Forge a Get request with ISIN Values | HTTP Request             | Fetch ETF profile HTML from JustETF | Gets rows from table        | Extracts defined values with css selector |                                                                                                                                              |
| Extracts defined values with css selector | HTML Extract             | Extract key ETF data from HTML     | Forge a Get request with ISIN Values | Loop Over Items             | ### Html content extraction\n- Extract html content into human readable text from the css selectors on just etf website\n- append or update data to your table |
| Extracts defined values in better format | Code                     | Parse and structure extracted data | Extracts defined values with css selector | Loop Over Items             |                                                                                                                                                |
| Loop Over Items                  | SplitInBatches           | Process each ETF item sequentially | Extracts defined values in better format | Edit Fields                |                                                                                                                                               |
| Edit Fields                     | Set                      | Convert and map fields for Excel   | Loop Over Items            | Updates my table            |                                                                                                                                               |
| Updates my table                | Microsoft Excel          | Update Excel worksheet rows         | Edit Fields                | -                          |                                                                                                                                               |
| Sticky Note                     | Sticky Note              | Documentation and overview          | -                          | -                          | # 📊 Automate Your ETF Comparison: Real-Time Data & Analysis 📈\n\nThis workflow automates ETF research by pulling fresh profile data into Excel whenever you click “Update Table.” ... |
| Sticky Note1                   | Sticky Note              | Trigger explanation                 | -                          | -                          | ### Trigger \n- Trigger manually \nor \n- Trigger using a web hook (called with a macro in excel for my part)                                   |
| Sticky Note2                   | Sticky Note              | Excel data explanation              | -                          | -                          | ### Excel data\n- start by logging the date and time of execution\n- Retrieve the rows of the table with the ETF ISIN\n- Forge a GET request   |
| Sticky Note3                   | Sticky Note              | HTML extraction explanation         | -                          | -                          | ### Html content extraction\n- Extract html content into human readable text from the css selectors on just etf website\n- append or update data |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Name: "When called by Excel Macro"  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `ETF`  
   - This will serve as the entry point triggered by the Excel macro.

2. **Add a Microsoft Excel Node to Update Timestamp**  
   - Name: "Logs the date & time"  
   - Operation: Update row in table  
   - Workbook: Select your Excel workbook (e.g., "My_investandearnings3")  
   - Worksheet: Select "Div study"  
   - Table: Select the relevant table (e.g., "MAJ")  
   - Update the column "Dernière mise à jour" with the current date/time in GMT-2 timezone using expression:  
     `={{ new Date().toLocaleString('en-GB', { timeZone: 'Etc/GMT-2', hour12: false }) }}`  
   - Connect Webhook node output to this node.

3. **Add Microsoft Excel Node to Get Rows from Table**  
   - Name: "Gets rows from table"  
   - Operation: Get Rows  
   - Workbook: Same as above  
   - Worksheet: "Div study"  
   - Table: The table containing ETF data (e.g., "DivComp")  
   - Return All: True  
   - Connect output of "Logs the date & time" node to this node.

4. **Add HTTP Request Node to Fetch ETF Profiles**  
   - Name: "Forge a Get request with ISIN Values"  
   - HTTP Method: GET  
   - URL:  
     `https://www.justetf.com/fr/etf-profile.html?isin={{ $json.ISIN }}`  
   - Connect output of "Gets rows from table" node to this node.

5. **Add HTML Extract Node**  
   - Name: "Extracts defined values with css selector"  
   - Operation: extractHtmlContent  
   - Extraction Values (key and CSS selector pairs):  
     - Dividends: `#etf-profile-body > div:nth-child(20)`  
     - Frais: `#etf-profile-body > div:nth-child(1) > div > div:nth-child(3) > div > div:nth-child(1) > div.val.bold`  
     - Performance depuis 5 ans: `#etf-profile-body > div:nth-child(18) > div.columns-2 > div:nth-child(1)`  
     - Name: `#etf-profile-body > div:nth-child(1) > div > div.e_head > div:nth-child(2)`  
   - Connect output of HTTP Request node to this node.

6. **Add Code Node to Format Extracted Data**  
   - Name: "Extracts defined values in better format"  
   - Paste the JavaScript code to parse dividends, fees, performance, and name from extracted HTML text (as per the workflow).  
   - Connect output of HTML Extract node to this node.

7. **Add SplitInBatches Node**  
   - Name: "Loop Over Items"  
   - Purpose: To process each ETF item one by one.  
   - Connect output of Code node to this node.

8. **Add Set Node to Edit Fields**  
   - Name: "Edit Fields"  
   - Configure to convert strings to numbers and map fields to Excel columns:  
     - Frais → number (parse percentage string to decimal)  
     - Rendement de départ → number (distribution yield)  
     - Performance depuis 5 ans → number (5 year performance)  
     - Dividende 12 mois, année précédente, il y a 2, 3, 4 ans → numbers (historic dividends)  
     - Nom → string (ETF name)  
   - Use expressions to parse and convert values accordingly.  
   - Connect output of "Loop Over Items" node to this node.

9. **Add Microsoft Excel Node to Update Table Rows**  
   - Name: "Updates my table"  
   - Operation: Update rows in worksheet  
   - Workbook and Worksheet: Same as previous Excel nodes  
   - Table: "Div study"  
   - Match rows on column "ISIN" with value `={{ $('Gets rows from table').item.json.ISIN }}`  
   - Update columns with values from "Edit Fields" node, e.g., Frais, Rendement de départ, Performance depuis 5 ans, Dividende fields, Nom  
   - Connect output of "Edit Fields" node to this node.

10. **Test Workflow**  
    - Add Manual Trigger or test webhook call from Excel macro button linked to the webhook URL.  
    - Verify that the "Dernière mise à jour" timestamp updates, ETF data is fetched and parsed correctly, and Excel rows update accordingly.

11. **Excel Macro Setup**  
    - Create an Excel macro button named "Update Table".  
    - Configure the macro to send an HTTP POST request to the webhook URL:  
      `https://<your-n8n-instance>/webhook/ETF`  
    - This triggers the entire workflow on demand.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| This workflow was designed to automate real-time ETF data updates by integrating Excel and web scraping with n8n. It leverages an Excel macro button to trigger updates easily.                                                       | Workflow description and design context                                                                         |
| The CSS selectors used for HTML extraction are dependent on the JustETF website’s current structure. Monitor for website changes that may require updates to selectors.                                                              | Website structure dependency                                                                                     |
| The workflow uses Microsoft Excel node with OAuth2 credentials linked to OneDrive-hosted Excel files. Ensure OAuth credentials are properly configured and have appropriate permissions.                                              | Credential setup for Microsoft Excel                                                                            |
| For regex parsing in the code node, be aware that dividend formats or performance texts may vary. Adjust regex patterns accordingly if data extraction fails.                                                                        | Data parsing considerations                                                                                      |
| Useful links for n8n Microsoft Excel node documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.microsoftExcel/                                                                                            | Official documentation                                                                                           |
| The workflow includes sticky notes within the editor to explain each major block and provide user instructions. Refer to these notes for quick contextual understanding during maintenance or handoff.                              | Internal workflow documentation                                                                                  |
| Video demonstration and project credits are not included but can be supplemented by workflow authors for training or sharing purposes.                                                                                              | Additional resources                                                                                            |

---

**Disclaimer:**  
The provided content is generated solely from an automated n8n workflow JSON export. It complies strictly with content policies and includes no illegal or protected elements. All data handled is legal and public.