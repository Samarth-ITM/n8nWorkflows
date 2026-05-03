Generate Daily Bitcoin On-Chain Metrics PDF Report with NASDAQ Data

https://n8nworkflows.xyz/workflows/generate-daily-bitcoin-on-chain-metrics-pdf-report-with-nasdaq-data-7836


# Generate Daily Bitcoin On-Chain Metrics PDF Report with NASDAQ Data

### 1. Workflow Overview

This workflow automates the generation of a daily Bitcoin on-chain metrics PDF report enriched with NASDAQ financial data. It is designed for financial analysts, cryptocurrency researchers, or data teams needing an automated, formatted report combining Bitcoin metrics with market data on a daily basis. The workflow orchestrates data fetching, formatting, aggregation, template updating, and final report generation.

Logical blocks:

- **1.1 Manual Trigger Input:** Start execution manually to initiate the workflow.
- **1.2 API Key Setup:** Prepare API key to authorize data requests.
- **1.3 NASDAQ Data Retrieval:** Fetch Bitcoin-related financial data from NASDAQ Data Link API.
- **1.4 Data Formatting:** Transform raw API data into a structured format suitable for report generation.
- **1.5 Data Aggregation:** Combine multiple data items into a single unified array.
- **1.6 Template Fetch and Update:** Retrieve a report template from an API, inject the formatted data.
- **1.7 PDF Report Download:** Request the updated template to generate and download the final PDF report.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger Input

- **Overview:** This block initiates the workflow manually, allowing user-controlled execution on demand.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)
- **Node Details:**

| Node Name                  | Details                                                                                                          |
|----------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | - Type: Manual Trigger<br>- Role: Starts workflow execution on user command<br>- Configuration: Default manual trigger with no parameters<br>- Input: None<br>- Output: Triggers downstream nodes<br>- Failure cases: None expected<br>- Version: 1 |

#### 2.2 API Key Setup

- **Overview:** Sets up necessary API key credentials or variables for authenticating subsequent API calls.
- **Nodes Involved:**  
  - Add API Key (Set Node)
- **Node Details:**

| Node Name    | Details                                                                                                            |
|--------------|--------------------------------------------------------------------------------------------------------------------|
| Add API Key  | - Type: Set Node<br>- Role: Defines and adds API key variables for authorization<br>- Configuration: Likely sets environment variables or parameters for API keys (exact key content not shown)<br>- Input: Trigger from manual node<br>- Output: Passes data with API key included<br>- Failure cases: Missing or invalid API key could cause downstream HTTP request failures<br>- Version: 3.4 |

#### 2.3 NASDAQ Data Retrieval

- **Overview:** Fetches Bitcoin-related financial data from the NASDAQ Data Link API using the configured API key.
- **Nodes Involved:**  
  - Get Data From Nasdaq Data Link (HTTP Request)
- **Node Details:**

| Node Name                  | Details                                                                                                            |
|----------------------------|--------------------------------------------------------------------------------------------------------------------|
| Get Data From Nasdaq Data Link | - Type: HTTP Request<br>- Role: Calls NASDAQ Data Link API to fetch financial metrics<br>- Configuration: Uses API key from previous node to authenticate; request URL and parameters configured to retrieve Bitcoin on-chain metrics and NASDAQ data<br>- Input: Receives API key from Set node<br>- Output: Raw JSON or CSV data response from API<br>- Failure cases: Network errors, invalid API key, API rate limits, malformed requests<br>- Version: 4.2 |

#### 2.4 Data Formatting

- **Overview:** Processes raw data into a structured format aligned with report requirements.
- **Nodes Involved:**  
  - Format the Output (Code Node)
- **Node Details:**

| Node Name         | Details                                                                                                            |
|-------------------|--------------------------------------------------------------------------------------------------------------------|
| Format the Output  | - Type: Code Node (JavaScript)<br>- Role: Parses and restructures raw API data into a clean format for aggregation and template injection<br>- Configuration: Custom script likely extracts key metrics, reformats date/time, and normalizes fields<br>- Input: Raw data from HTTP Request node<br>- Output: Formatted array or object<br>- Failure cases: Parsing errors, data inconsistency, missing fields<br>- Version: 2 |

#### 2.5 Data Aggregation

- **Overview:** Combines multiple formatted data objects into a single array for batch processing.
- **Nodes Involved:**  
  - Combine All Items Into One Array (Code Node)
- **Node Details:**

| Node Name                   | Details                                                                                                            |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------|
| Combine All Items Into One Array | - Type: Code Node (JavaScript)<br>- Role: Merges multiple formatted data items into one consolidated array for template injection<br>- Configuration: Custom script concatenating arrays or objects<br>- Input: Formatted data from previous node<br>- Output: Single aggregated array containing all items<br>- Failure cases: Empty input arrays, type mismatches<br>- Version: 2 |

#### 2.6 Template Fetch and Update

- **Overview:** Downloads a predefined report template from an API and updates it with the aggregated data.
- **Nodes Involved:**  
  - Fetch, Edit and Update Template From APITemplate (HTTP Request)
- **Node Details:**

| Node Name                             | Details                                                                                                            |
|-------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Fetch, Edit and Update Template From APITemplate | - Type: HTTP Request<br>- Role: Retrieves template and sends updated data payload to API for report generation preparation<br>- Configuration: Calls template API endpoint, uses aggregated data as request body or parameters<br>- Input: Aggregated data array<br>- Output: Confirmation or intermediate resource for final report<br>- Failure cases: API errors, invalid template ID, update conflicts<br>- Version: 4.2 |

#### 2.7 PDF Report Download

- **Overview:** Initiates the generation and download of the final PDF report based on the updated template.
- **Nodes Involved:**  
  - Download PDF Report (HTTP Request)
- **Node Details:**

| Node Name              | Details                                                                                                            |
|------------------------|--------------------------------------------------------------------------------------------------------------------|
| Download PDF Report     | - Type: HTTP Request<br>- Role: Requests the final PDF report file from the server using the updated template<br>- Configuration: HTTP GET or POST to download endpoint; may handle file response type<br>- Input: Response or token from template update node<br>- Output: Binary PDF file or download link<br>- Failure cases: File not found, network errors, access denied<br>- Version: 4.2 |

---

### 3. Summary Table

| Node Name                             | Node Type          | Functional Role                                   | Input Node(s)                      | Output Node(s)                           | Sticky Note |
|-------------------------------------|--------------------|--------------------------------------------------|----------------------------------|-----------------------------------------|-------------|
| When clicking ‘Execute workflow’    | Manual Trigger     | Start workflow execution manually                 | None                             | Add API Key                             |             |
| Add API Key                         | Set                | Set API key credentials for authorization         | When clicking ‘Execute workflow’ | Get Data From Nasdaq Data Link          |             |
| Get Data From Nasdaq Data Link      | HTTP Request       | Retrieve Bitcoin and NASDAQ data                    | Add API Key                     | Format the Output                       |             |
| Format the Output                   | Code               | Format raw API data into structured format         | Get Data From Nasdaq Data Link  | Combine All Items Into One Array        |             |
| Combine All Items Into One Array    | Code               | Aggregate formatted data into one array             | Format the Output               | Fetch, Edit and Update Template From APITemplate |             |
| Fetch, Edit and Update Template From APITemplate | HTTP Request       | Fetch and update report template with data         | Combine All Items Into One Array | Download PDF Report                     |             |
| Download PDF Report                 | HTTP Request       | Download the final generated PDF report             | Fetch, Edit and Update Template From APITemplate | None                                    |             |
| Sticky Note                        | Sticky Note        | (Empty content)                                     | None                            | None                                    |             |
| Sticky Note1                       | Sticky Note        | (Empty content)                                     | None                            | None                                    |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: When clicking ‘Execute workflow’  
   - Purpose: Starts workflow manually

2. **Create Set Node for API Key**  
   - Type: Set  
   - Name: Add API Key  
   - Add a field for API key (e.g., `nasdaqApiKey`) with your NASDAQ Data Link API key as its value  
   - Connect output of Manual Trigger to this node

3. **Create HTTP Request Node to Fetch NASDAQ Data**  
   - Type: HTTP Request  
   - Name: Get Data From Nasdaq Data Link  
   - Method: GET (or as required by API)  
   - URL: NASDAQ Data Link API endpoint for Bitcoin metrics  
   - Authentication: Use API key from previous Set node (pass as header or query parameter)  
   - Connect output of Add API Key to this node

4. **Create Code Node to Format Raw Data**  
   - Type: Code  
   - Name: Format the Output  
   - Language: JavaScript  
   - Script: Parse incoming data, extract necessary metrics, convert timestamps, normalize fields  
   - Connect output of HTTP Request node to this node

5. **Create Code Node to Combine Items**  
   - Type: Code  
   - Name: Combine All Items Into One Array  
   - Language: JavaScript  
   - Script: Merge multiple formatted data items into a single array for uniform processing  
   - Connect output of Format the Output node to this node

6. **Create HTTP Request Node to Fetch and Update Template**  
   - Type: HTTP Request  
   - Name: Fetch, Edit and Update Template From APITemplate  
   - Method: POST or PUT as required by template API  
   - URL: Template API endpoint  
   - Body: Pass aggregated data array as JSON payload  
   - Authentication: Configure if required by API  
   - Connect output of Combine All Items Into One Array node to this node

7. **Create HTTP Request Node to Download PDF Report**  
   - Type: HTTP Request  
   - Name: Download PDF Report  
   - Method: GET or POST to download endpoint  
   - URL: Endpoint to retrieve the finalized PDF report generated from the updated template  
   - Configure binary data handling if needed for file download  
   - Connect output of Fetch, Edit and Update Template From APITemplate node to this node

8. **(Optional) Add Sticky Notes**  
   - Place sticky notes for documentation or future maintenance as needed

Ensure all nodes use compatible n8n versions (HTTP Request nodes version 4.2 or higher recommended). Credentials for APIs must be created and linked properly in n8n credentials manager.

---

### 5. General Notes & Resources

| Note Content                                                                             | Context or Link                                           |
|------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow requires a valid NASDAQ Data Link API key for data retrieval                | NASDAQ Data Link API documentation: https://data.nasdaq.com/ |
| Template API must support update and PDF generation endpoints                            | Check API specs of your template provider                  |
| Ensure proper handling of binary files in HTTP Request node for PDF download             | n8n HTTP Request node docs: https://docs.n8n.io/nodes/n8n-nodes-base.http-request/ |
| Manual trigger allows flexible execution timing; can be replaced with scheduled trigger | n8n Scheduling docs: https://docs.n8n.io/nodes/n8n-nodes-base.cron/ |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.