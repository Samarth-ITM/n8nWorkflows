Create a BTC/ETH Price & USD Exchange Rate API with CoinGecko & ExchangeRate-API

https://n8nworkflows.xyz/workflows/create-a-btc-eth-price---usd-exchange-rate-api-with-coingecko---exchangerate-api-8389


# Create a BTC/ETH Price & USD Exchange Rate API with CoinGecko & ExchangeRate-API

### 1. Workflow Overview

This workflow creates a lightweight micro-API endpoint that provides real-time cryptocurrency prices for Bitcoin (BTC) and Ethereum (ETH), including their 24-hour percentage changes, alongside current USD exchange rates for EUR and NGN. The API is designed for easy integration and quick retrieval of essential crypto and FX data in a single JSON response.

Logical blocks:

- **1.1 Input Reception:** Receives HTTP POST requests on a webhook endpoint.
- **1.2 Data Retrieval:** Concurrently fetches fiat exchange rates and cryptocurrency prices from two external APIs.
- **1.3 Data Aggregation:** Merges the responses from both API calls.
- **1.4 Data Transformation:** Processes and formats the merged data into a clean JSON structure.
- **1.5 Output Delivery:** Responds to the webhook request with the formatted JSON payload.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block exposes a webhook that serves as the entry point for external requests. It listens for HTTP POST requests at a specific path and triggers the workflow execution.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (n8n-nodes-base.webhook)  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `/crypto-fx`  
      - Response Mode: `responseNode` (the response is handled by a downstream Respond node)  
    - Key variables: None  
    - Input connections: None (entry point)  
    - Output connections: Connects to both "Get Fiat Exchange Rates" and "Get Crypto Prices" nodes in parallel  
    - Edge cases:  
      - Missing or malformed POST request data (though this workflow does not require body content)  
      - HTTP method other than POST will not trigger the workflow  
    - Version: v1

#### 1.2 Data Retrieval

- **Overview:**  
  This block makes two concurrent HTTP requests to external APIs: one to fetch fiat currency exchange rates from ExchangeRate-API, and another to fetch cryptocurrency prices from CoinGecko.

- **Nodes Involved:**  
  - Get Fiat Exchange Rates  
  - Get Crypto Prices

- **Node Details:**

  - **Get Fiat Exchange Rates**  
    - Type: HTTP Request (n8n-nodes-base.httpRequest)  
    - Configuration:  
      - URL: `https://api.exchangerate-api.com/v4/latest/USD`  
      - HTTP Method: GET (default)  
      - No authentication required  
    - Input: Receives trigger from Webhook node  
    - Output: JSON containing exchange rates of USD against various fiat currencies  
    - Edge cases:  
      - API downtime or network issues causing request failure or timeout  
      - Unexpected API response structure  
    - Version: v4.1

  - **Get Crypto Prices**  
    - Type: HTTP Request (n8n-nodes-base.httpRequest)  
    - Configuration:  
      - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`  
      - HTTP Method: GET (default)  
      - No authentication required  
    - Input: Receives trigger from Webhook node  
    - Output: JSON containing BTC and ETH prices in USD and their 24h percent changes  
    - Edge cases:  
      - API rate limiting or downtime  
      - Changes in API response format  
    - Version: v4.1

#### 1.3 Data Aggregation

- **Overview:**  
  This block merges the outputs from both API calls, combining fiat exchange rates and crypto prices into a single data set for further processing.

- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - **Merge**  
    - Type: Merge (n8n-nodes-base.merge)  
    - Configuration: Default settings (merge by index, combining two inputs)  
    - Inputs:  
      - Input 1: Output from "Get Crypto Prices"  
      - Input 2: Output from "Get Fiat Exchange Rates"  
    - Output: Combined data payload passed downstream  
    - Edge cases:  
      - Unequal or missing inputs could cause incomplete data merging  
      - Timing issues if one HTTP request is significantly delayed  
    - Version: v3.2

#### 1.4 Data Transformation

- **Overview:**  
  This block extracts relevant data from the merged inputs and constructs a clean JSON object containing BTC/ETH prices and 24h changes, USD→EUR and USD→NGN exchange rates, and a timestamp.

- **Nodes Involved:**  
  - Build JSON (Code node)

- **Node Details:**

  - **Build JSON**  
    - Type: Code (n8n-nodes-base.code)  
    - Configuration:  
      - JavaScript code extracts `rates` from fiat data and `bitcoin`/`ethereum` data from crypto response  
      - Constructs a JSON payload with:  
        - `btc`: { price, change_24h }  
        - `eth`: { price, change_24h }  
        - `usd_eur`, `usd_ngn` exchange rates  
        - `ts`: ISO timestamp of data retrieval  
    - Key expressions:  
      ```javascript
      let fx, cg;
      for(const it of $input.all()){
        if(it.json?.rates) fx=it.json;
        if(it.json?.bitcoin) cg=it.json;
      }
      const payload={
        btc:{ price: cg?.bitcoin?.usd ?? null, change_24h: cg?.bitcoin?.usd_24h_change ?? null },
        eth:{ price: cg?.ethereum?.usd ?? null, change_24h: cg?.ethereum?.usd_24h_change ?? null },
        usd_eur: fx?.rates?.EUR ?? null,
        usd_ngn: fx?.rates?.NGN ?? null,
        ts: new Date().toISOString()
      };
      return [{json: payload}];
      ```
    - Inputs: Merged data from Merge node  
    - Outputs: Single JSON object as described  
    - Edge cases:  
      - Missing fields default to `null` gracefully  
      - Unexpected data structure may cause `undefined` values  
    - Version: v2

#### 1.5 Output Delivery

- **Overview:**  
  This block sends the constructed JSON response back to the original webhook caller, completing the request-response cycle.

- **Nodes Involved:**  
  - Respond

- **Node Details:**

  - **Respond**  
    - Type: Respond to Webhook (n8n-nodes-base.respondToWebhook)  
    - Configuration: Default (responds with input data)  
    - Input: Receives the JSON payload from Build JSON node  
    - Output: HTTP response to the client with JSON content  
    - Edge cases:  
      - If upstream node fails or returns no data, response may be empty or error-prone  
    - Version: v1

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role               | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                         |
|-------------------------|---------------------------|------------------------------|---------------------------|-------------------------|---------------------------------------------------------------------------------------------------|
| Webhook                 | Webhook                   | Entry point for API requests | —                         | Get Fiat Exchange Rates, Get Crypto Prices |                                                                                                   |
| Get Fiat Exchange Rates | HTTP Request              | Fetch USD fiat exchange rates| Webhook                   | Merge                   |                                                                                                   |
| Get Crypto Prices       | HTTP Request              | Fetch BTC & ETH prices       | Webhook                   | Merge                   |                                                                                                   |
| Merge                   | Merge                     | Combine fiat and crypto data | Get Crypto Prices, Get Fiat Exchange Rates | Build JSON              |                                                                                                   |
| Build JSON              | Code                      | Format and build JSON output | Merge                     | Respond                 |                                                                                                   |
| Respond                 | Respond to Webhook        | Send JSON response to caller | Build JSON                | —                       |                                                                                                   |
| Sticky Note             | Sticky Note               | Documentation note           | —                         | —                       | 🌐 Crypto + FX Micro-API (Webhook JSON)  What it does: Exposes a lightweight GET endpoint returning BTC/ETH price + 24h % and USD→EUR / USD→NGN in clean JSON. Route: `/webhook/crypto-fx`. Flow: Webhook (GET) → ExchangeRate-API → CoinGecko → Merge → Code (build JSON) → Respond. Example Response provided.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Name: `Webhook`  
   - HTTP Method: `POST`  
   - Path: `crypto-fx`  
   - Response Mode: `responseNode` (so downstream Respond node handles the reply)  

2. **Create an HTTP Request node for fiat exchange rates**  
   - Name: `Get Fiat Exchange Rates`  
   - URL: `https://api.exchangerate-api.com/v4/latest/USD`  
   - Method: GET (default)  
   - Connect input from `Webhook` node  

3. **Create an HTTP Request node for crypto prices**  
   - Name: `Get Crypto Prices`  
   - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`  
   - Method: GET (default)  
   - Connect input from `Webhook` node  

4. **Create a Merge node**  
   - Name: `Merge`  
   - Default merge mode (Merge by index)  
   - Connect inputs from both `Get Fiat Exchange Rates` and `Get Crypto Prices` nodes  

5. **Create a Code node to build JSON output**  
   - Name: `Build JSON`  
   - Language: JavaScript  
   - Paste the following code:
     ```javascript
     let fx, cg;
     for(const it of $input.all()){
       if(it.json?.rates) fx=it.json;
       if(it.json?.bitcoin) cg=it.json;
     }
     const payload={
       btc:{ price: cg?.bitcoin?.usd ?? null, change_24h: cg?.bitcoin?.usd_24h_change ?? null },
       eth:{ price: cg?.ethereum?.usd ?? null, change_24h: cg?.ethereum?.usd_24h_change ?? null },
       usd_eur: fx?.rates?.EUR ?? null,
       usd_ngn: fx?.rates?.NGN ?? null,
       ts: new Date().toISOString()
     };
     return [{json: payload}];
     ```
   - Connect input from `Merge` node  

6. **Create a Respond to Webhook node**  
   - Name: `Respond`  
   - Connect input from `Build JSON` node  
   - No additional configuration needed (defaults to respond with input data)  

7. **Connect the nodes in this sequence:**  
   - `Webhook` → `Get Fiat Exchange Rates` and `Get Crypto Prices` (parallel)  
   - `Get Fiat Exchange Rates` + `Get Crypto Prices` → `Merge`  
   - `Merge` → `Build JSON`  
   - `Build JSON` → `Respond`  

8. **Credentials:**  
   - None required for this workflow as both APIs are public and do not need authentication.

9. **Testing:**  
   - Use n8n’s test webhook URL or production URL with POST request to `/webhook/crypto-fx`  
   - Expect a JSON response with BTC/ETH prices and USD exchange rates for EUR and NGN, including timestamp.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow exposes a lightweight GET endpoint returning combined crypto and fiat exchange data in a clean JSON format.                       | Sticky Note content in workflow; Route: `/webhook/crypto-fx`                                            |
| Example JSON response format is included in the sticky note for clarity and easy API consumer reference.                                        | Sticky Note                                                                                              |
| Both APIs used (CoinGecko and ExchangeRate-API) are public and require no authentication, simplifying deployment and usage.                   | Workflow design choice                                                                                   |
| This micro-API can be integrated into dashboards, bots, or financial analysis tools requiring near real-time crypto and FX data in one call.  | Use case context                                                                                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.