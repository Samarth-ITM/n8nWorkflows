Receive Bitcoin, Etherium, Solana, Binance Data With Gecko Coin and Gmail

https://n8nworkflows.xyz/workflows/receive-bitcoin--etherium--solana--binance-data-with-gecko-coin-and-gmail-4244


# Receive Bitcoin, Etherium, Solana, Binance Data With Gecko Coin and Gmail

### 1. Workflow Overview

This n8n workflow automates the retrieval and email distribution of OHLC (Open, High, Low, Close) price data for four major cryptocurrencies: Bitcoin (BTC), Ethereum (ETH), Solana (SOL), and Binance Coin (BNB). It uses the Gecko Coin API to fetch market data and Gmail to send a compiled report.

The workflow is logically organized into these blocks:  
- **1.1 Scheduled Trigger:** Initiates the workflow on a schedule.  
- **1.2 Data Retrieval:** Four parallel HTTP Request nodes fetch OHLC data for BTC, ETH, SOL, and BNB from Gecko Coin.  
- **1.3 Data Formatting:** Each coin’s raw data is processed and formatted for readability using Code nodes.  
- **1.4 Data Merging:** The formatted data streams are merged stepwise, combining BTC and ETH first, then adding SOL, and finally BNB.  
- **1.5 Data Combination:** All merged data are consolidated into a single output.  
- **1.6 Email Dispatch:** The compiled data is sent via Gmail node to deliver the report.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

**Overview:**  
Starts the workflow execution at defined time intervals.

**Nodes Involved:**  
- Schedule

**Node Details:**  
- **Schedule**  
  - Type: Schedule Trigger  
  - Configuration: Default schedule parameters (likely cron or time interval trigger)  
  - Inputs: None (trigger node)  
  - Outputs: 1 output to four parallel HTTP request nodes  
  - Failures: Usually none; possible misconfiguration of schedule timing  
  - Notes: The schedule controls the automatic periodic data retrieval cycle.

---

#### 2.2 Data Retrieval

**Overview:**  
Fetches OHLC data for each cryptocurrency using Gecko Coin HTTP API endpoints.

**Nodes Involved:**  
- Get BITCOIN OHLC  
- Get ETHEREUM OHLC  
- Get SOLANA OHLC  
- Get BINANCECOIN OHLC

**Node Details:**  

- **Get BITCOIN OHLC**  
  - Type: HTTP Request  
  - Role: Retrieves Bitcoin market data  
  - Configuration: Calls Gecko Coin API endpoint configured to fetch BTC OHLC data  
  - Inputs: Trigger from Schedule  
  - Outputs: Raw JSON with BTC OHLC data to Format BTC node  
  - Failures: Network errors, API rate limits, invalid API URLs or parameters

- **Get ETHEREUM OHLC**  
  - Type: HTTP Request  
  - Role: Retrieves Ethereum market data  
  - Configuration: Similar to BTC node, fetching ETH data  
  - Inputs: Trigger from Schedule  
  - Outputs: Raw JSON with ETH OHLC data to Format ETH node  
  - Failures: Same as above

- **Get SOLANA OHLC**  
  - Type: HTTP Request  
  - Role: Retrieves Solana market data  
  - Configuration: Gecko Coin API call for SOL  
  - Inputs: Trigger from Schedule  
  - Outputs: Raw JSON with SOL OHLC data to Format SOL node  
  - Failures: Same as above

- **Get BINANCECOIN OHLC**  
  - Type: HTTP Request  
  - Role: Retrieves Binance Coin market data  
  - Configuration: Gecko Coin API call for BNB  
  - Inputs: Trigger from Schedule  
  - Outputs: Raw JSON with BNB OHLC data to Format BNB node  
  - Failures: Same as above

---

#### 2.3 Data Formatting

**Overview:**  
Processes and formats the raw OHLC JSON data into a clean, structured format for merging.

**Nodes Involved:**  
- Format BTC  
- Format ETH  
- Format SOL  
- Format BNB

**Node Details:**  

- **Format BTC**  
  - Type: Code (JavaScript)  
  - Role: Parses and formats BTC data  
  - Inputs: Raw BTC OHLC JSON from Get BITCOIN OHLC  
  - Outputs: Formatted BTC data to Merge 1 (BTC + ETH) node  
  - Key Logic: Extracts relevant OHLC values and formats them as human-readable text or object  
  - Failures: Parsing errors if input JSON is malformed, expression errors in code

- **Format ETH**  
  - Type: Code (JavaScript)  
  - Role: Parses and formats ETH data  
  - Inputs: Raw ETH OHLC JSON from Get ETHEREUM OHLC  
  - Outputs: Formatted ETH data to Merge 1 (BTC + ETH) node  
  - Key Logic: Same as Format BTC but for ETH  
  - Failures: Same as Format BTC

- **Format SOL**  
  - Type: Code (JavaScript)  
  - Role: Parses and formats SOL data  
  - Inputs: Raw SOL OHLC JSON from Get SOLANA OHLC  
  - Outputs: Formatted SOL data to Merge 2 (+SOL) node  
  - Key Logic: Same as above for SOL  
  - Failures: Same as above

- **Format BNB**  
  - Type: Code (JavaScript)  
  - Role: Parses and formats BNB data  
  - Inputs: Raw BNB OHLC JSON from Get BINANCECOIN OHLC  
  - Outputs: Formatted BNB data to Merge 3 (+BNB) node  
  - Key Logic: Same as above for BNB  
  - Failures: Same as above

---

#### 2.4 Data Merging

**Overview:**  
Incrementally merges the formatted data streams into a single combined dataset.

**Nodes Involved:**  
- Merge 1 (BTC + ETH)  
- Merge 2 (+SOL)  
- Merge 3 (+BNB)

**Node Details:**  

- **Merge 1 (BTC + ETH)**  
  - Type: Merge  
  - Role: Combines formatted BTC and ETH data  
  - Inputs: Format BTC (index 0), Format ETH (index 1)  
  - Outputs: Merged BTC + ETH data to Merge 2 (+SOL)  
  - Failures: If input data missing or malformed, the merge may fail or produce incomplete data

- **Merge 2 (+SOL)**  
  - Type: Merge  
  - Role: Adds formatted SOL data to previous merge  
  - Inputs: Merge 1 (BTC + ETH) (index 0), Format SOL (index 1)  
  - Outputs: Merged BTC + ETH + SOL data to Merge 3 (+BNB)  
  - Failures: Same as above

- **Merge 3 (+BNB)**  
  - Type: Merge  
  - Role: Adds formatted BNB data to the combined dataset  
  - Inputs: Merge 2 (+SOL) (index 0), Format BNB (index 1)  
  - Outputs: Complete merged data to Combine All Coins  
  - Failures: Same as above

---

#### 2.5 Data Combination

**Overview:**  
Consolidates all merged data into a final formatted message suitable for email.

**Nodes Involved:**  
- Combine All Coins

**Node Details:**  

- **Combine All Coins**  
  - Type: Code (JavaScript)  
  - Role: Joins all merged coin data into a single string or HTML email body  
  - Inputs: Fully merged data from Merge 3 (+BNB)  
  - Outputs: Final combined message to Send Gmail node  
  - Key Logic: Aggregates text or structured data into an email-friendly format  
  - Failures: Any expression failure or formatting error could cause message to be incomplete

---

#### 2.6 Email Dispatch

**Overview:**  
Sends the compiled cryptocurrency OHLC report via Gmail.

**Nodes Involved:**  
- Send Gmail

**Node Details:**  

- **Send Gmail**  
  - Type: Gmail  
  - Role: Sends an email with the combined OHLC data report  
  - Configuration: Uses Gmail OAuth2 credentials for authentication  
  - Inputs: Combined message from Combine All Coins  
  - Outputs: None (terminal node)  
  - Failures: Authentication errors, quota limits, invalid recipient address, network issues

---

### 3. Summary Table

| Node Name               | Node Type       | Functional Role                     | Input Node(s)                                   | Output Node(s)             | Sticky Note |
|-------------------------|-----------------|-----------------------------------|------------------------------------------------|----------------------------|-------------|
| Schedule                | Schedule Trigger| Initiates workflow on schedule    | None                                           | Get BITCOIN OHLC, Get ETHEREUM OHLC, Get SOLANA OHLC, Get BINANCECOIN OHLC |             |
| Get BITCOIN OHLC        | HTTP Request    | Fetch BTC OHLC data               | Schedule                                       | Format BTC                 |             |
| Get ETHEREUM OHLC       | HTTP Request    | Fetch ETH OHLC data               | Schedule                                       | Format ETH                 |             |
| Get SOLANA OHLC         | HTTP Request    | Fetch SOL OHLC data               | Schedule                                       | Format SOL                 |             |
| Get BINANCECOIN OHLC    | HTTP Request    | Fetch BNB OHLC data               | Schedule                                       | Format BNB                 |             |
| Format BTC              | Code            | Format BTC data                   | Get BITCOIN OHLC                               | Merge 1 (BTC + ETH)        |             |
| Format ETH              | Code            | Format ETH data                   | Get ETHEREUM OHLC                              | Merge 1 (BTC + ETH)        |             |
| Merge 1 (BTC + ETH)     | Merge           | Merge BTC and ETH formatted data | Format BTC, Format ETH                          | Merge 2 (+SOL)             |             |
| Format SOL              | Code            | Format SOL data                   | Get SOLANA OHLC                                | Merge 2 (+SOL)             |             |
| Merge 2 (+SOL)          | Merge           | Merge SOL data with BTC+ETH       | Merge 1 (BTC + ETH), Format SOL                 | Merge 3 (+BNB)             |             |
| Format BNB              | Code            | Format BNB data                   | Get BINANCECOIN OHLC                           | Merge 3 (+BNB)             |             |
| Merge 3 (+BNB)          | Merge           | Merge BNB data with previous coins| Merge 2 (+SOL), Format BNB                       | Combine All Coins          |             |
| Combine All Coins       | Code            | Combine all merged data into email| Merge 3 (+BNB)                                 | Send Gmail                 |             |
| Send Gmail              | Gmail           | Send compiled report by email    | Combine All Coins                              | None                      |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Set it to trigger at your desired interval (e.g., daily at a set time).
   - No credentials required.

2. **Create four HTTP Request nodes:**  
   For each cryptocurrency (BTC, ETH, SOL, BNB):
   - Set method to GET.
   - Configure URL to Gecko Coin API endpoint for OHLC data of the respective coin. (Example: `https://api.coingecko.com/api/v3/coins/{coin_id}/ohlc?vs_currency=usd&days=1`)
     - Replace `{coin_id}` with `bitcoin`, `ethereum`, `solana`, and `binancecoin` respectively.
   - Connect Schedule node’s output to all four HTTP Request nodes in parallel.

3. **Create four Code nodes (Format BTC, Format ETH, Format SOL, Format BNB):**  
   For each:
   - Set language to JavaScript.
   - Write code to parse the HTTP response JSON, extract OHLC values, and format them into a clean object or string.
   - Connect each HTTP Request node’s output to its corresponding Format node.

4. **Create three Merge nodes:**  
   - Merge 1: Set to merge two inputs — connect Format BTC (input 0) and Format ETH (input 1).  
   - Merge 2: Merge output of Merge 1 (input 0) and Format SOL (input 1).  
   - Merge 3: Merge output of Merge 2 (input 0) and Format BNB (input 1).

5. **Create Combine All Coins Code node:**  
   - Connect output of Merge 3 to this node.  
   - Write code to aggregate all merged data into a single string or formatted HTML suitable for email.  
   - Example: concatenate OHLC summaries per coin with line breaks.

6. **Create Send Gmail node:**  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient email address.  
   - Use the combined message from the previous node as the email body.  
   - Set subject line (e.g., “Daily Cryptocurrency OHLC Report”).

7. **Connect nodes as per the merging and flow described.**

8. **Test workflow:**  
   - Run manually or wait for scheduled trigger.  
   - Validate successful HTTP requests, data formatting, merging, and email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                               |
|------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Gecko Coin API documentation: https://www.coingecko.com/en/api                                  | Reference for HTTP request configuration and data structure   |
| Gmail node requires OAuth2 credentials setup in n8n credentials section                         | Setup Gmail integration carefully to avoid auth errors        |
| Merge nodes combine multiple JSON data streams based on input order                             | Useful for incremental data aggregation                        |
| Code nodes use JavaScript for data parsing and formatting                                      | Ensure code handles missing or malformed data gracefully      |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. It adheres strictly to content policies and contains no illegal or protected elements. All handled data is legal and public.