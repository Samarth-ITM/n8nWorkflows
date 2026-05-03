TW-Request-Agri Data Open Platform-Daily Market Sheep Pricing

https://n8nworkflows.xyz/workflows/tw-request-agri-data-open-platform-daily-market-sheep-pricing-2687


# TW-Request-Agri Data Open Platform-Daily Market Sheep Pricing

### 1. Workflow Overview

This workflow automates the retrieval of daily sheep market pricing data from the Taiwan Agricultural Products Open Data Platform and appends the processed data into a Google Sheets document for subsequent analysis. It is designed for agricultural analysts, market researchers, or data engineers who need to maintain up-to-date transaction records for sheep pricing in a structured spreadsheet format.

The workflow consists of four main logical blocks:

- **1.1 Manual Trigger**: Initiates the workflow manually, allowing controlled execution.
- **1.2 Data Retrieval via HTTP Request**: Sends a GET request to the Taiwan Agricultural Products Open Data Platform API to fetch sheep quotation data for a specified date range and market.
- **1.3 Data Processing (Split Out)**: Splits the array of transaction records returned by the API into individual items for granular processing.
- **1.4 Data Storage (Google Sheets Append)**: Appends each individual record into a designated Google Sheets document, mapping relevant fields for easy analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger Block

- **Overview:**  
  This block provides a manual start point for the workflow, enabling users to execute the data fetching process on demand rather than on a schedule.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)

- **Node Details:**  
  - **Name:** When clicking ‘Test workflow’  
  - **Type:** Manual Trigger (`n8n-nodes-base.manualTrigger`)  
  - **Configuration:** No parameters required; triggers workflow execution when manually activated.  
  - **Input Connections:** None (entry point)  
  - **Output Connections:** Connected to HTTP Request node  
  - **Version Requirements:** Compatible with n8n v1+  
  - **Potential Failures:** None expected; user-dependent trigger  
  - **Sub-Workflow:** None

#### 2.2 Data Retrieval via HTTP Request Block

- **Overview:**  
  This block sends an HTTP GET request to the Taiwan Agricultural Products Open Data Platform API to retrieve sheep market transaction data filtered by date range and market name.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **Name:** HTTP Request  
  - **Type:** HTTP Request (`n8n-nodes-base.httpRequest`)  
  - **Configuration:**  
    - **Method:** GET  
    - **URL:** `https://data.moa.gov.tw/api/v1/SheepQuotation`  
    - **Query Parameters:**  
      - `Start_time`: `2024/12/01`  
      - `End_time`: `2024/12/31`  
      - `MarketName`: `台北二` (Taipei No. 2 Market)  
      - `api_key`: `3AFID4BGE9PDQ2WTFDO1X61H4RNQLE` (example key; replace with valid key)  
    - **Headers:**  
      - `accept`: `application/json`  
    - **Response Format:** JSON expected  
  - **Input Connections:** From Manual Trigger  
  - **Output Connections:** To Split Out node  
  - **Version Requirements:** HTTP Request node v4.2 or higher recommended for query parameter support  
  - **Potential Failures:**  
    - API key invalid or expired → authentication error  
    - Network timeout or connectivity issues  
    - API rate limiting or quota exceeded  
    - Unexpected response format or empty data  
  - **Sub-Workflow:** None

#### 2.3 Data Processing (Split Out) Block

- **Overview:**  
  This block takes the array of transaction records from the API response and splits it into individual items so that each record can be processed separately downstream.

- **Nodes Involved:**  
  - Split Out

- **Node Details:**  
  - **Name:** Split Out  
  - **Type:** Split Out (`n8n-nodes-base.splitOut`)  
  - **Configuration:**  
    - **Field to Split Out:** `Data` (the array field in the API response containing transaction records)  
  - **Input Connections:** From HTTP Request  
  - **Output Connections:** To Google Sheets node  
  - **Version Requirements:** Split Out node v1+  
  - **Potential Failures:**  
    - If `Data` field is missing or empty → no items to split, downstream nodes receive no input  
    - Malformed JSON response causing parsing errors  
  - **Sub-Workflow:** None

#### 2.4 Data Storage (Google Sheets Append) Block

- **Overview:**  
  This block appends each individual transaction record into a Google Sheets document, mapping specific fields from the API data to corresponding columns in the sheet.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Name:** Google Sheets  
  - **Type:** Google Sheets (`n8n-nodes-base.googleSheets`)  
  - **Configuration:**  
    - **Operation:** Append (adds new rows)  
    - **Document ID:** `17EJTOetBsfoGkzADCUHPoXaQW7FLQziYmQxKNJNnDIU` (example; replace with your own)  
    - **Sheet Name:** `Sheet1` (gid=0)  
    - **Columns Mapped (Auto Map Input Data):**  
      - `TransDate`  
      - `TcType`  
      - `CropCode`  
      - `CropName`  
      - `MarketCode`  
      - `MarketName`  
      - `Upper_Price`  
      - `Middle_Price`  
      - `Lower_Price`  
      - `Avg_Price`  
      - `Trans_Quantity`  
    - **Credentials:** Google Sheets OAuth2 account configured  
  - **Input Connections:** From Split Out  
  - **Output Connections:** None (end node)  
  - **Version Requirements:** Google Sheets node v4.5+ recommended for stable OAuth2 integration  
  - **Potential Failures:**  
    - Authentication failure due to expired or invalid OAuth2 token  
    - Google Sheets API quota exceeded  
    - Document ID or Sheet name incorrect or inaccessible  
    - Data type mismatches or missing fields causing append errors  
  - **Sub-Workflow:** None

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                  | Input Node(s)                | Output Node(s)       | Sticky Note                                                                                   |
|---------------------------|---------------------|---------------------------------|-----------------------------|----------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Manual start of workflow         | None                        | HTTP Request          | Allows manual execution of the workflow to control when data is fetched.                      |
| HTTP Request              | HTTP Request        | Fetch agricultural data          | When clicking ‘Test workflow’ | Split Out             | Sends request to Taiwan Agricultural Products Open Data Platform API with date and market filters. |
| Split Out                 | Split Out           | Split API response array into items | HTTP Request                | Google Sheets         | Processes each record individually, ensuring accurate handling of every data entry.          |
| Google Sheets             | Google Sheets       | Append data to Google Sheets     | Split Out                   | None                 | Appends data into structured Google Sheets document for easy access and analysis.            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.  
   - No configuration needed. This node will start the workflow manually.

2. **Create HTTP Request Node**  
   - Add an **HTTP Request** node named `HTTP Request`.  
   - Set **HTTP Method** to `GET`.  
   - Set **URL** to `https://data.moa.gov.tw/api/v1/SheepQuotation`.  
   - Enable **Send Query Parameters** and add the following parameters:  
     - `Start_time`: `2024/12/01`  
     - `End_time`: `2024/12/31`  
     - `MarketName`: `台北二`  
     - `api_key`: `<your_api_key>` (replace with your valid API key)  
   - Enable **Send Headers** and add header:  
     - `accept`: `application/json`  
   - Connect output of Manual Trigger node to input of this HTTP Request node.

3. **Create Split Out Node**  
   - Add a **Split Out** node named `Split Out`.  
   - Set **Field to Split Out** to `Data` (the field in the HTTP response containing the array of records).  
   - Connect output of HTTP Request node to input of Split Out node.

4. **Create Google Sheets Node**  
   - Add a **Google Sheets** node named `Google Sheets`.  
   - Set **Operation** to `Append`.  
   - Set **Document ID** to your Google Sheets document ID (e.g., `17EJTOetBsfoGkzADCUHPoXaQW7FLQziYmQxKNJNnDIU`).  
   - Set **Sheet Name** to `Sheet1` (or the appropriate sheet name/gid).  
   - Set **Mapping Mode** to `Auto Map Input Data`.  
   - Ensure the following columns exist in your sheet and are mapped:  
     - `TransDate`, `TcType`, `CropCode`, `CropName`, `MarketCode`, `MarketName`, `Upper_Price`, `Middle_Price`, `Lower_Price`, `Avg_Price`, `Trans_Quantity`  
   - Configure Google Sheets OAuth2 credentials for this node.  
   - Connect output of Split Out node to input of Google Sheets node.

5. **Save and Test**  
   - Save the workflow.  
   - Click “Execute Workflow” or “Test Workflow” to manually trigger and verify data retrieval and insertion.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Use the **Curl Import** feature in n8n to quickly create HTTP Request nodes from example curl commands.       | Example curl: `curl -X GET "https://data.moa.gov.tw/api/v1/AgriProductsTransType/?Start_time=114.01.01&End_time=114.01.01&MarketName=%E5%8F%B0%E5%8C%97%E4%BA%8C" -H "accept: application/json"` |
| Official documentation for Taiwan Agricultural Products Open Data Platform API.                               | [https://data.moa.gov.tw/api.aspx](https://data.moa.gov.tw/api.aspx)                            |
| Ensure Google Sheets document has proper sharing permissions for the OAuth2 account used in n8n.               | Google Sheets API requires the authenticated user to have edit access to the target document.  |
| The API key used in HTTP Request must be valid and authorized for the data endpoint; replace placeholder key. | API key management is handled via Taiwan Agricultural Products Open Data Platform.             |

---

This documentation fully describes the workflow’s structure, node configurations, and reproduction steps, enabling both human operators and automation agents to understand, replicate, and maintain the workflow effectively.