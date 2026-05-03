Track Crypto Prices & FX Rates with CoinGecko & ExchangeRate-API to Notion

https://n8nworkflows.xyz/workflows/track-crypto-prices---fx-rates-with-coingecko---exchangerate-api-to-notion-8387


# Track Crypto Prices & FX Rates with CoinGecko & ExchangeRate-API to Notion

---

### 1. Workflow Overview

This workflow automates the hourly tracking of cryptocurrency prices and foreign exchange (FX) rates, then logs the data into a Notion database for easy monitoring and analysis. It is designed for users who want to maintain an up-to-date dashboard of key financial indicators, specifically Bitcoin (BTC), Ethereum (ETH), and USD-based FX rates (USD→EUR and USD→NGN).

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every 60 minutes.
- **1.2 Data Retrieval:** Fetches real-time FX rates from ExchangeRate-API and crypto prices from CoinGecko.
- **1.3 Data Aggregation:** Merges the two data streams into a single dataset.
- **1.4 Data Preparation:** Builds the structured data payload formatted for Notion.
- **1.5 Data Storage:** Creates a new page entry in the Notion database with the latest data.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically every hour to ensure continuous data updates.

- **Nodes Involved:**  
  - Every 60 Minutes

- **Node Details:**

  - **Every 60 Minutes**  
    - **Type & Role:** Schedule Trigger — initiates workflow execution on a fixed time interval.  
    - **Configuration:** Set to trigger every 60 minutes (cron configurable in minutes field).  
    - **Expressions/Variables:** None.  
    - **Input/Output:** No input; outputs trigger signal to two HTTP request nodes.  
    - **Version Requirements:** Compatible with n8n version supporting scheduleTrigger v1.1.  
    - **Potential Failures:** Scheduler misconfiguration, node disabled, or platform downtime preventing trigger.

#### 1.2 Data Retrieval

- **Overview:**  
  Concurrently fetches live data from two external APIs: ExchangeRate-API for fiat currency rates and CoinGecko for cryptocurrency prices and 24h changes.

- **Nodes Involved:**  
  - Get Fiat Exchange Rates  
  - Get Crypto Prices

- **Node Details:**

  - **Get Fiat Exchange Rates**  
    - **Type & Role:** HTTP Request node — obtains USD-based FX rates.  
    - **Configuration:**  
      - URL: `https://api.exchangerate-api.com/v4/latest/USD`  
      - Method: GET (default)  
      - No authentication or additional headers required.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Input from Schedule Trigger; output JSON contains rates keyed by currency codes.  
    - **Potential Failures:** API downtime, network errors, rate limiting, invalid JSON response.

  - **Get Crypto Prices**  
    - **Type & Role:** HTTP Request node — obtains current BTC and ETH prices in USD plus 24-hour percent change.  
    - **Configuration:**  
      - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`  
      - Method: GET  
      - No authentication required.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Input from Schedule Trigger; output JSON includes price and 24h change for bitcoin and ethereum.  
    - **Potential Failures:** API limits, network issues, malformed response, or CoinGecko downtime.

#### 1.3 Data Aggregation

- **Overview:**  
  Merges the two parallel data streams into a single combined dataset to facilitate unified processing downstream.

- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - **Merge**  
    - **Type & Role:** Merge node — consolidates multiple inputs into one output stream.  
    - **Configuration:** Default merge mode, joining the two inputs on main input 0 and 1.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Inputs from “Get Crypto Prices” and “Get Fiat Exchange Rates”; output to code node for data structuring.  
    - **Potential Failures:** Input synchronization issues if one API call fails or delays; node may output incomplete data if one input missing.

#### 1.4 Data Preparation

- **Overview:**  
  Constructs a structured JSON object formatted to match the Notion database schema, including timestamped title and numeric fields for crypto and FX data.

- **Nodes Involved:**  
  - Build Notion Page

- **Node Details:**

  - **Build Notion Page**  
    - **Type & Role:** Code node (JavaScript) — transforms merged data into Notion-compatible page properties.  
    - **Configuration:**  
      - Custom JS script extracts rates from both inputs, maps fields to Notion property names:  
        - Title property with current date/time in Africa/Lagos timezone (format: "Crypto+FX — <date/time>").  
        - Numeric properties: BTC, BTC_24h, ETH, ETH_24h, USD_EUR, USD_NGN.  
    - **Expressions/Variables:**  
      - Uses `$input.all()` to access all incoming data; conditional checks on presence of `rates` or `bitcoin` keys to identify source data.  
      - Date localized with `toLocaleString()` specifying 'Africa/Lagos' timezone.  
    - **Input/Output:** Input from Merge; outputs single JSON object formatted for Notion.  
    - **Potential Failures:** Code errors if input data missing or malformed; timezone specification issues; null values if API data missing fields.

#### 1.5 Data Storage

- **Overview:**  
  Creates a new page entry in the specified Notion database using the prepared data object.

- **Nodes Involved:**  
  - Create in Notion

- **Node Details:**

  - **Create in Notion**  
    - **Type & Role:** Notion node — inserts a new page into a database.  
    - **Configuration:**  
      - Database ID set via URL mode (user must paste Notion database URL).  
      - “Simple” mode disabled to allow property mapping.  
      - No extra options enabled.  
    - **Expressions/Variables:** None in parameters, uses input JSON from the code node.  
    - **Input/Output:** Input from Build Notion Page; outputs the API response from Notion.  
    - **Version Requirements:** Requires valid Notion OAuth2 credential with write access to target database.  
    - **Potential Failures:** Invalid database ID, permission errors, API rate limits, network errors, schema mismatch in Notion database fields.

---

### 3. Summary Table

| Node Name             | Node Type                | Functional Role              | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                                                        |
|-----------------------|--------------------------|-----------------------------|----------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Every 60 Minutes      | Schedule Trigger         | Initiates workflow hourly   | —                          | Get Fiat Exchange Rates, Get Crypto Prices | ## 📊 Crypto + FX → Notion Dashboard<br>**What it does:** Creates a Notion page entry hourly with BTC/ETH prices, 24h % changes, USD→EUR / USD→NGN rates.<br>**Trigger:** Hourly (adjustable cron).<br>**Nodes:** Schedule → ExchangeRate-API → CoinGecko → Merge → Code (build properties) → Notion (Create Page)<br>**Setup:** Create Notion DB with fields: Title, BTC, BTC_24h, ETH, ETH_24h, USD_EUR, USD_NGN; paste DB ID in Notion node; run once to confirm.<br>**Author:** David Olusola |
| Get Fiat Exchange Rates | HTTP Request            | Fetches USD FX rates        | Every 60 Minutes           | Merge                   | (see above – shared sticky note)                                                                                                  |
| Get Crypto Prices      | HTTP Request             | Fetches BTC, ETH prices     | Every 60 Minutes           | Merge                   | (see above – shared sticky note)                                                                                                  |
| Merge                 | Merge                    | Combines FX and crypto data | Get Crypto Prices, Get Fiat Exchange Rates | Build Notion Page       | (see above – shared sticky note)                                                                                                  |
| Build Notion Page      | Code                     | Formats data for Notion     | Merge                      | Create in Notion         | (see above – shared sticky note)                                                                                                  |
| Create in Notion       | Notion                   | Creates new Notion page     | Build Notion Page           | —                       | (see above – shared sticky note)                                                                                                  |
| Sticky Note            | Sticky Note              | Documentation and instructions | —                          | —                       | See content in “Sticky Note” node above                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node named "Every 60 Minutes":**  
   - Set trigger interval to every 60 minutes (minutes field).  
   - No credentials needed.

2. **Add an HTTP Request node named "Get Fiat Exchange Rates":**  
   - Connect input from "Every 60 Minutes".  
   - Set method to GET.  
   - Set URL to `https://api.exchangerate-api.com/v4/latest/USD`.  
   - No authentication or headers needed.

3. **Add an HTTP Request node named "Get Crypto Prices":**  
   - Connect input from "Every 60 Minutes".  
   - Set method to GET.  
   - Set URL to `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`.  
   - No authentication or headers needed.

4. **Add a Merge node named "Merge":**  
   - Set mode to Merge Inputs (keep default).  
   - Connect “Get Crypto Prices” output to input 0 of Merge.  
   - Connect “Get Fiat Exchange Rates” output to input 1 of Merge.

5. **Add a Code node named "Build Notion Page":**  
   - Connect input from "Merge".  
   - Paste the following JavaScript code in the node’s code editor:

   ```javascript
   let fx, cg;
   for (const it of $input.all()) {
     if (it.json?.rates) fx = it.json;
     if (it.json?.bitcoin) cg = it.json;
   }
   const page = {
     properties: {
       Title: { title: [{ text: { content: `Crypto+FX — ${new Date().toLocaleString('en-US', { timeZone: 'Africa/Lagos' })}` } }] },
       BTC: { number: cg?.bitcoin?.usd ?? null },
       BTC_24h: { number: cg?.bitcoin?.usd_24h_change ?? null },
       ETH: { number: cg?.ethereum?.usd ?? null },
       ETH_24h: { number: cg?.ethereum?.usd_24h_change ?? null },
       USD_EUR: { number: fx?.rates?.EUR ?? null },
       USD_NGN: { number: fx?.rates?.NGN ?? null }
     }
   };
   return [{ json: page }];
   ```

6. **Add a Notion node named "Create in Notion":**  
   - Connect input from "Build Notion Page".  
   - Configure credentials: set up and select Notion OAuth2 credentials with write access to your Notion workspace.  
   - In parameters, paste your Notion database URL or ID (URL mode).  
   - Disable "Simple" mode to enable property mapping.  
   - Leave other options default.

7. **(Optional) Add a Sticky Note node for documentation:**  
   - Paste the workflow description and setup instructions as provided in the sticky note content.  
   - Position it clearly for reference.

8. **Save and activate the workflow.**  
   - Run manually once to verify data insertion in Notion.  
   - Verify your Notion database has the following properties (all Number type except Title):  
     - BTC, BTC_24h, ETH, ETH_24h, USD_EUR, USD_NGN.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow was authored by David Olusola, who offers trading and consulting services via sales@daexai.com              | Contact email for consulting: sales@daexai.com                                                  |
| Notion database must have a Title field and numeric fields: BTC, BTC_24h, ETH, ETH_24h, USD_EUR, USD_NGN                  | Ensures compatibility with the data payload structure                                          |
| Example Notion page title format: "Crypto+FX — Monday 09:00"                                                             | Timestamped for clarity                                                                         |
| CoinGecko API docs: https://www.coingecko.com/en/api/documentation                                                      | Reference for price and 24h change parameters                                                  |
| ExchangeRate-API docs: https://www.exchangerate-api.com/docs/free                                                        | Reference for currency rate data                                                               |
| Official n8n Notion node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion/         | For credential setup and usage details                                                         |

---

**Disclaimer:**  
This document is derived exclusively from an n8n workflow designed for automation purposes. It complies fully with content policies and contains no illegal or protected information. All processed data is public and legal.