Tesla Quant Technical Indicators Webhooks Tool

https://n8nworkflows.xyz/workflows/tesla-quant-technical-indicators-webhooks-tool-4095


# Tesla Quant Technical Indicators Webhooks Tool

### 1. Workflow Overview

This workflow, named **Tesla Quant Technical Indicators Webhooks Tool**, acts as a centralized data fetcher and formatter for Tesla (TSLA) trading technical indicators leveraging the Alpha Vantage Premium API. It targets use cases where downstream AI trading agents or analytical tools require clean, uniform, and recent technical indicator data across three distinct timeframes: `15min`, `1hour`, and `1day`.

---

The workflow is organized into three nearly identical logical blocks, one per timeframe:

- **1.1 Webhook Trigger (Input Reception)**: Entry points exposed as webhook endpoints (`/15minData`, `/1hourData`, `/1dayData`) that accept requests from other workflows or agents.

- **1.2 Alpha Vantage API Calls (Data Acquisition)**: For each timeframe, six HTTP Request nodes independently fetch raw JSON data for six key technical indicators: RSI, MACD, BBANDS, SMA, EMA, and ADX.

- **1.3 Data Formatting & Merging (Processing & Output)**: Each raw indicator response is passed through a dedicated JavaScript Code node which extracts, trims to the latest 20 data points, and reshapes the data into a consistent JSON structure. The formatted outputs are merged into a single array per timeframe, then returned to the caller via a Respond to Webhook node.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Trigger Block (Input Reception)

**Overview:**  
This block receives external calls via webhook endpoints that trigger the entire data fetching and formatting sequence for a specific timeframe.

**Nodes Involved:**  
- `15min Data Webhook`  
- `1hour Data Webhook`  
- `1day Data Webhook`

**Node Details:**

- **Type:** Webhook (n8n-nodes-base.webhook)  
- **Role:** Entry point for external HTTP requests, each listening on a unique path (`88e83c35-bba7-4ff8-a808-9626ae04b88c` for 15min, `1eb54436-6511-4c6c-8d27-2a6ff8f15411` for 1hour, `65f1d0f0-b165-45e7-9a99-ceb977a9abbf` for 1day)  
- **Configuration:** Each webhook node has `responseMode` set to `responseNode` to defer response until processing completes  
- **Input/Output:** No inputs; outputs trigger six HTTP Requests for the indicators of the related timeframe  
- **Edge Cases:**  
  - Incorrect webhook path usage causes no trigger  
  - Timeout if downstream nodes fail  
- **Sticky Notes:** Blue sticky notes near each webhook node explain their role as entry points and response orchestration

---

#### 2.2 Alpha Vantage API Calls Block (Data Acquisition)

**Overview:**  
This block consists of six HTTP Request nodes per timeframe, each querying a specific Alpha Vantage technical indicator API endpoint for Tesla stock at the given interval.

**Nodes Involved (per timeframe):**

- RSI
- MACD
- BBANDS
- SMA
- EMA
- ADX

**Node Details (Example - RSI 15min):**

- **Type:** HTTP Request (n8n-nodes-base.httpRequest)  
- **Role:** Fetch raw technical indicator JSON data from Alpha Vantage  
- **Configuration:**  
  - URL constructed with parameters: `function=RSI`, `symbol=TSLA`, `interval` matching timeframe (e.g., `15min`), `time_period=14` (or 20 for some), `series_type=close`  
  - Authentication via `httpQueryAuth` credential named `Alpha Vantage Premium` (query param `apikey`)  
- **Input/Output:** Triggered by webhook node; outputs raw JSON to formatter code node  
- **Edge Cases:**  
  - API rate limits or quota reached  
  - Authentication failure (invalid/missing API key)  
  - Network timeout  
  - Unexpected/missing data in response  
- **Sticky Notes:** Yellow color-coded notes detail API endpoint usage and parameters

**Note:** Time period parameter varies slightly per indicator and timeframe, e.g., 14 or 20.

---

#### 2.3 Data Formatting & Merge Block (Processing & Output)

**Overview:**  
This block processes raw API responses to extract, clean, and reformat the latest 20 data points per indicator into a standardized JSON structure. The results per timeframe are merged and sent back to the caller.

**Nodes Involved:**

- Six JavaScript Code nodes per timeframe (one per indicator, e.g., `Format Response - MACD 15min`)  
- One Merge node per timeframe (e.g., `Merge 15 min Indicators`)  
- One Respond to Webhook node per timeframe (e.g., `Respond to Webhook 15min Data`)

**Node Details:**

- **JavaScript Code Nodes:**  
  - **Role:** Parse raw JSON, check for presence of expected keys like `"Technical Analysis: MACD"`, trim to latest 20 entries, and reformat data  
  - **Configuration:** Custom JS code per indicator to extract fields such as MACD, RSI, BBANDS bands, SMA, EMA, ADX  
  - **Expressions/Variables:** Use of `Object.entries()`, array slicing for 20 records, mapping for extracting numeric values  
  - **Input:** Raw JSON from HTTP Request node  
  - **Output:** JSON object with keys: `indicator`, `timeframe`, `data` (cleaned)  
  - **Edge Cases:**  
    - Missing or empty indicator data returns JSON error object  
    - Parsing errors if data structure changes  
- **Merge Nodes:**  
  - **Type:** Merge (n8n-nodes-base.merge) with 6 inputs  
  - **Role:** Combine all formatted indicator JSON objects into a single array for the timeframe  
  - **Input:** All 6 formatted outputs  
  - **Output:** Array containing all indicator objects  
- **Respond to Webhook Nodes:**  
  - **Type:** Respond to Webhook (n8n-nodes-base.respondToWebhook)  
  - **Role:** Send merged JSON array as HTTP response to original webhook caller  
  - **Configuration:** `respondWith` set to `allIncomingItems` to return full merged data  
  - **Input:** Output from Merge node  
  - **Edge Cases:** Failure to respond if merge node fails or no data present

**Sticky Notes:** Purple notes explain formatting logic; green notes describe merge; blue notes mark response nodes.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                              | Input Node(s)                      | Output Node(s)                       | Sticky Note                                                                                          |
|----------------------------|----------------------------|----------------------------------------------|----------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------|
| 15min Data Webhook          | Webhook                    | Entry point for 15min timeframe data fetch  | None                             | MACD 15min, RSI 15min, ...          | Entry point for external workflows calling indicator batches; orchestrates API pull + formatting    |
| MACD 15min                 | HTTP Request               | Fetch MACD indicator raw data for 15min     | 15min Data Webhook               | Format Response - MACD 15min        | Alpha Vantage API call for MACD 15min; uses httpQueryAuth credential                               |
| RSI 15min                  | HTTP Request               | Fetch RSI indicator raw data for 15min      | 15min Data Webhook               | Format Response - RSI 15min         | Alpha Vantage API call for RSI 15min                                                             |
| BBands 15min               | HTTP Request               | Fetch BBANDS indicator raw data for 15min   | 15min Data Webhook               | Format Response - BBANDS 15min      | Alpha Vantage API call for BBANDS 15min                                                          |
| SMA 15min                  | HTTP Request               | Fetch SMA indicator raw data for 15min      | 15min Data Webhook               | Format Response - SMA 15min         | Alpha Vantage API call for SMA 15min                                                             |
| EMA 15min                  | HTTP Request               | Fetch EMA indicator raw data for 15min      | 15min Data Webhook               | Format Response - EMA 15min         | Alpha Vantage API call for EMA 15min                                                             |
| ADX 15min                  | HTTP Request               | Fetch ADX indicator raw data for 15min      | 15min Data Webhook               | Format Response - ADX 15min         | Alpha Vantage API call for ADX 15min                                                             |
| Format Response - MACD 15min| Code                      | Clean and format MACD 15min data             | MACD 15min                      | Merge 15 min Indicators             | Parses and extracts latest 20 MACD data points                                                    |
| Format Response - RSI 15min | Code                      | Clean and format RSI 15min data              | RSI 15min                      | Merge 15 min Indicators             | Parses and extracts latest 20 RSI data points                                                     |
| Format Response - BBANDS 15min| Code                    | Clean and format BBANDS 15min data           | BBands 15min                   | Merge 15 min Indicators             | Parses and extracts latest 20 BBANDS data points                                                  |
| Format Response - SMA 15min | Code                      | Clean and format SMA 15min data              | SMA 15min                      | Merge 15 min Indicators             | Parses and extracts latest 20 SMA data points                                                     |
| Format Response - EMA 15min | Code                      | Clean and format EMA 15min data              | EMA 15min                      | Merge 15 min Indicators             | Parses and extracts latest 20 EMA data points                                                     |
| Format Response - ADX 15min | Code                      | Clean and format ADX 15min data              | ADX 15min                      | Merge 15 min Indicators             | Parses and extracts latest 20 ADX data points                                                     |
| Merge 15 min Indicators     | Merge                     | Merge all 15min formatted indicators         | All 6 Format Response - * 15min | Respond to Webhook 15min Data       | Final merge of formatted 15min indicators                                                        |
| Respond to Webhook 15min Data| Respond to Webhook       | Return merged 15min indicators to caller     | Merge 15 min Indicators          | None                               | Returns cleaned batch via webhook response                                                        |
| 1hour Data Webhook          | Webhook                    | Entry point for 1hour timeframe data fetch  | None                             | MACD 1hour, RSI 1hour, ...          | Entry point for external workflows calling indicator batches; orchestrates API pull + formatting    |
| MACD 1hour                 | HTTP Request               | Fetch MACD indicator raw data for 1hour     | 1hour Data Webhook              | Format Response - MACD 1hour        | Alpha Vantage API call for MACD 1hour                                                           |
| RSI 1hour                  | HTTP Request               | Fetch RSI indicator raw data for 1hour      | 1hour Data Webhook              | Format Response - RSI 1hour         | Alpha Vantage API call for RSI 1hour                                                            |
| BBands 1hour               | HTTP Request               | Fetch BBANDS indicator raw data for 1hour   | 1hour Data Webhook              | Format Response - BBANDS 1hour      | Alpha Vantage API call for BBANDS 1hour                                                         |
| SMA 1hour                  | HTTP Request               | Fetch SMA indicator raw data for 1hour      | 1hour Data Webhook              | Format Response - SMA 1hour         | Alpha Vantage API call for SMA 1hour                                                            |
| EMA 1hour                  | HTTP Request               | Fetch EMA indicator raw data for 1hour      | 1hour Data Webhook              | Format Response - EMA 1hour         | Alpha Vantage API call for EMA 1hour                                                            |
| ADX 1hour                  | HTTP Request               | Fetch ADX indicator raw data for 1hour      | 1hour Data Webhook              | Format Response - ADX 1hour         | Alpha Vantage API call for ADX 1hour                                                            |
| Format Response - MACD 1hour| Code                      | Clean and format MACD 1hour data             | MACD 1hour                     | Merge 1hour Indicators              | Parses and extracts latest 20 MACD data points                                                    |
| Format Response - RSI 1hour | Code                      | Clean and format RSI 1hour data              | RSI 1hour                     | Merge 1hour Indicators              | Parses and extracts latest 20 RSI data points                                                     |
| Format Response - BBANDS 1hour| Code                    | Clean and format BBANDS 1hour data           | BBands 1hour                  | Merge 1hour Indicators              | Parses and extracts latest 20 BBANDS data points                                                  |
| Format Response - SMA 1hour | Code                      | Clean and format SMA 1hour data              | SMA 1hour                     | Merge 1hour Indicators              | Parses and extracts latest 20 SMA data points                                                     |
| Format Response - EMA 1hour | Code                      | Clean and format EMA 1hour data              | EMA 1hour                     | Merge 1hour Indicators              | Parses and extracts latest 20 EMA data points                                                     |
| Format Response - ADX 1hour | Code                      | Clean and format ADX 1hour data              | ADX 1hour                     | Merge 1hour Indicators              | Parses and extracts latest 20 ADX data points                                                     |
| Merge 1hour Indicators      | Merge                     | Merge all 1hour formatted indicators         | All 6 Format Response - * 1hour| Respond to Webhook 1hour Data       | Final merge of formatted 1hour indicators                                                       |
| Respond to Webhook 1hour Data| Respond to Webhook       | Return merged 1hour indicators to caller     | Merge 1hour Indicators          | None                               | Returns cleaned batch via webhook response                                                       |
| 1day Data Webhook           | Webhook                    | Entry point for 1day timeframe data fetch   | None                             | MACD 1day, RSI 1day, ...            | Entry point for external workflows calling indicator batches; orchestrates API pull + formatting    |
| MACD 1day                  | HTTP Request               | Fetch MACD indicator raw data for 1day      | 1day Data Webhook              | Format Response - MACD 1day         | Alpha Vantage API call for MACD 1day                                                            |
| RSI 1day                   | HTTP Request               | Fetch RSI indicator raw data for 1day       | 1day Data Webhook              | Format Response - RSI 1day          | Alpha Vantage API call for RSI 1day                                                             |
| BBands 1day                | HTTP Request               | Fetch BBANDS indicator raw data for 1day    | 1day Data Webhook              | Format Response - BBANDS 1day       | Alpha Vantage API call for BBANDS 1day                                                          |
| SMA 1day                   | HTTP Request               | Fetch SMA indicator raw data for 1day       | 1day Data Webhook              | Format Response - SMA 1day          | Alpha Vantage API call for SMA 1day                                                             |
| EMA 1day                   | HTTP Request               | Fetch EMA indicator raw data for 1day       | 1day Data Webhook              | Format Response - EMA 1day          | Alpha Vantage API call for EMA 1day                                                             |
| ADX 1day                   | HTTP Request               | Fetch ADX indicator raw data for 1day       | 1day Data Webhook              | Format Response - ADX 1day          | Alpha Vantage API call for ADX 1day                                                             |
| Format Response - MACD 1day | Code                      | Clean and format MACD 1day data              | MACD 1day                    | Merge 1day Indicators               | Parses and extracts latest 20 MACD data points                                                    |
| Format Response - RSI 1day  | Code                      | Clean and format RSI 1day data               | RSI 1day                    | Merge 1day Indicators               | Parses and extracts latest 20 RSI data points                                                     |
| Format Response - BBANDS 1day| Code                     | Clean and format BBANDS 1day data            | BBands 1day                 | Merge 1day Indicators               | Parses and extracts latest 20 BBANDS data points                                                  |
| Format Response - SMA 1day  | Code                      | Clean and format SMA 1day data               | SMA 1day                    | Merge 1day Indicators               | Parses and extracts latest 20 SMA data points                                                     |
| Format Response - EMA 1day  | Code                      | Clean and format EMA 1day data               | EMA 1day                    | Merge 1day Indicators               | Parses and extracts latest 20 EMA data points                                                     |
| Format Response - ADX 1day  | Code                      | Clean and format ADX 1day data               | ADX 1day                    | Merge 1day Indicators               | Parses and extracts latest 20 ADX data points                                                     |
| Merge 1day Indicators       | Merge                     | Merge all 1day formatted indicators          | All 6 Format Response - * 1day | Respond to Webhook 1day Data        | Final merge of formatted 1day indicators                                                        |
| Respond to Webhook 1day Data| Respond to Webhook        | Return merged 1day indicators to caller      | Merge 1day Indicators           | None                               | Returns cleaned batch via webhook response                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials**  
   - Go to Credentials in n8n  
   - Add a new credential of type `HTTP Query Auth`  
   - Name it `Alpha Vantage Premium`  
   - Set Query Parameter Key: `apikey`  
   - Set your Alpha Vantage Premium API key as the value  

2. **Setup Webhook Nodes (3 total)**  
   - Create Webhook node named `15min Data Webhook`  
     - Set path: unique (e.g., `/15minData` or generated UUID)  
     - Set response mode: `responseNode` (to delay response)  
   - Repeat for `1hour Data Webhook` and `1day Data Webhook` with respective unique paths  

3. **Create HTTP Request Nodes (18 total; 6 per timeframe)**  
   - For each timeframe (`15min`, `1hour`, `1day`), create 6 HTTP Request nodes named e.g., `RSI 15min`, `MACD 15min`, etc.  
   - Configure each node with:  
     - URL per Alpha Vantage API specification with parameters:  
       - `function` set to indicator name (RSI, MACD, BBANDS, SMA, EMA, ADX)  
       - `symbol=TSLA`  
       - `interval` set to timeframe (`15min`, `60min`, `daily`)  
       - `time_period` typically 14 (or 20 for BBANDS/SMA in 15min)  
       - `series_type=close`  
       - `entitlement=delayed` (optional per original)  
     - Authentication: select `Alpha Vantage Premium` credential (httpQueryAuth)  
   - Connect each timeframe’s webhook node to its 6 HTTP Request nodes in parallel  

4. **Create JavaScript Code Nodes for Formatting (18 total)**  
   - For each HTTP Request node, create a corresponding Code node named e.g., `Format Response - RSI 15min`  
   - Write JS code to:  
     - Extract the relevant key from the response JSON (e.g., `"Technical Analysis: RSI"`)  
     - Check if data exists; if not return error JSON  
     - Extract and slice latest 20 timestamps  
     - Format each entry into a simplified object with indicator name, timeframe, and data array/object  
   - Connect each HTTP Request node output to its formatting Code node input  

5. **Create Merge Nodes (3 total)**  
   - For each timeframe, create a Merge node with 6 inputs named e.g., `Merge 15 min Indicators`  
   - Connect all 6 formatter Code nodes of that timeframe as inputs to the merge node  

6. **Create Respond to Webhook Nodes (3 total)**  
   - Create one Respond to Webhook node per timeframe (e.g., `Respond to Webhook 15min Data`)  
   - Set parameter `respondWith` to `allIncomingItems` to return the merged array  
   - Connect each Merge node to its corresponding Respond node  

7. **Connect the Workflow**  
   - Webhook nodes trigger the 6 HTTP Request nodes (parallel fan-out)  
   - Each HTTP Request node connects to its formatting Code node  
   - Formatting Code nodes feed into the Merge node for their timeframe  
   - Merge node connects to Respond to Webhook node for response  

8. **Set Workflow Execution Order to `v1`** (optional but matches original)  

9. **Save and Activate Workflow**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Requires **Alpha Vantage Premium API Key** registered as an HTTP Query Auth credential named `Alpha Vantage Premium`.          | https://www.alphavantage.co/premium/                                                            |
| Workflow exposes 3 webhook endpoints `/15minData`, `/1hourData`, and `/1dayData` for external agents to fetch indicator data.  | Use case: Tesla Quant AI trading system indicator tools                                         |
| Each timeframe fetches 6 technical indicators: RSI, MACD, BBANDS, SMA, EMA, ADX from Alpha Vantage Premium API.                 | Alpha Vantage official API documentation                                                        |
| Cleaned JSON output format per indicator: `{ indicator, timeframe, data }` with `data` holding latest 20 points.               | Standardized output enables consistent processing downstream                                    |
| Workflow protected under © 2025 Treasurium Capital Limited Company; licensed use only.                                           | Support: Don Jayamaha LinkedIn https://linkedin.com/in/donjayamahajr                            |
| Sticky notes in workflow use color codes: Blue for Webhook nodes, Yellow for API calls, Purple for formatters, Green for merge.| Visual aids in n8n for workflow clarity                                                         |

---

This documentation provides a complete understanding of the Tesla Quant Technical Indicators Webhooks Tool workflow for both human operators and AI-based automation agents. It enables reproduction, customization, and error anticipation for smooth integration of Tesla technical indicator data into trading systems.