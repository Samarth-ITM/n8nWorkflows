Automated Gold Price Tracker with Multiple Currency Conversion for Telegram 📈

https://n8nworkflows.xyz/workflows/automated-gold-price-tracker-with-multiple-currency-conversion-for-telegram----6178


# Automated Gold Price Tracker with Multiple Currency Conversion for Telegram 📈

### 1. Workflow Overview

This workflow automates the tracking of gold prices in multiple currencies and sends formatted updates to a specified Telegram chat every 15 minutes. It fetches the latest gold price in USD per ounce, converts it into grams and multiple currency values, formats this data into a neatly aligned Markdown message, and delivers it via Telegram.

Logical blocks include:
- **1.1 Input Setup:** Initialize input parameters such as target currency and Telegram chat ID.
- **1.2 Data Retrieval:** Fetch gold price in USD and current currency exchange rates.
- **1.3 Data Merging:** Combine gold pricing and exchange rate data for processing.
- **1.4 Report Preparation:** Calculate gold prices per gram and ounce in different karats and currencies; format the output as a Markdown message.
- **1.5 Message Sending:** Send the formatted message to Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Setup

- **Overview:**  
  Sets up the initial parameters required for the workflow: the target currency code and the Telegram chat ID where messages will be sent. These are sourced from environment variables.

- **Nodes Involved:**  
  - Set Currency  
  - Schedule Trigger  
  - Sticky Note (inputs description)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow every 15 minutes.  
    - Configuration: Interval set to trigger every 15 minutes.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Set Currency" node.  
    - Edge cases: Workflow will not run if disabled; scheduling misconfiguration could cause no triggers.

  - **Set Currency**  
    - Type: Set  
    - Role: Assigns workflow variables "currency" and "telegram_chat_id" from environment variables.  
    - Configuration:  
      - `currency` = `{{$env.currency}}`  
      - `telegram_chat_id` = `{{$env.telegram_chat_id}}`  
    - Inputs: From Schedule Trigger  
    - Outputs: Connects to "Fetch Price" and "Convert Currency" nodes.  
    - Edge cases: Missing or incorrect environment variables will cause downstream failures or incorrect data.

  - **Sticky Note** (positioned near input nodes)  
    - Purpose: Notes the input parameters expected for the workflow (`currency`).  
    - No technical function.

---

#### 1.2 Data Retrieval

- **Overview:**  
  Fetches current gold price data and currency exchange rates to enable multi-currency conversion of gold prices.

- **Nodes Involved:**  
  - Fetch Price  
  - Convert Currency  
  - Sticky Notes (pricing and currency descriptions)

- **Node Details:**

  - **Fetch Price**  
    - Type: HTTP Request  
    - Role: Fetches gold price data in USD per ounce from goldprice.org API.  
    - Configuration:  
      - URL: `https://data-asg.goldprice.org/dbXRates/USD`  
      - Timeout: 5 seconds  
      - Custom User-Agent header mimicking an iPhone Safari browser  
      - Retry enabled on failure  
    - Inputs: From "Set Currency"  
    - Outputs: Connects to "Merge" node  
    - Edge cases:  
      - Timeout or network failure triggers retry.  
      - API format changes could break parsing downstream.  
      - If API is down, no data fetched.

  - **Convert Currency**  
    - Type: HTTP Request  
    - Role: Fetches latest currency exchange rates relative to USD.  
    - Configuration:  
      - URL: `https://open.er-api.com/v6/latest/USD`  
      - No special headers or retries specified.  
    - Inputs: From "Set Currency"  
    - Outputs: Connects to "Merge" node  
    - Edge cases:  
      - API downtime or changes in response structure.  
      - If no data or partial data is returned, conversion may fail.

  - **Sticky Notes (Pricing and Currency)**  
    - Provide contextual labeling for the nodes fetching pricing and currency data.

---

#### 1.3 Data Merging

- **Overview:**  
  Combines the outputs of the gold price and currency exchange rate fetch operations into a single data structure for further processing.

- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines two input streams by position into a single output.  
    - Configuration: Mode set to "Combine", combining inputs by position (index-based).  
    - Inputs:  
      - Input 1: Gold price data from "Fetch Price"  
      - Input 2: Exchange rate data from "Convert Currency"  
    - Outputs: Connects to "Report: Prepare" node  
    - Edge cases:  
      - If one input lags or is missing, merge may delay or output incomplete data.  
      - Data mismatch could cause errors in subsequent nodes.

---

#### 1.4 Report Preparation

- **Overview:**  
  Processes the combined data to compute gold prices per gram and ounce in the target currency and USD, for various karat purities. Formats the results into a neatly aligned Markdown table suitable for Telegram messaging.

- **Nodes Involved:**  
  - Report: Prepare  
  - Sticky Note (report description)

- **Node Details:**

  - **Report: Prepare**  
    - Type: Code (JavaScript)  
    - Role:  
      - Extracts gold price per ounce in USD from data.  
      - Converts per ounce price to per gram.  
      - Calculates prices for 24k, 21k (87.5%), and 18k (75%) gold in both the target currency and USD.  
      - Formats numbers with thousand separators and fixed column widths for alignment.  
      - Escapes Markdown special characters for safe Telegram display.  
      - Builds a Markdown V2 formatted table with flags and currency codes.  
      - Adds an arrow emoji indicating price increase or decrease with percentage change.  
    - Key expressions/variables:  
      - `gPerOz = 31.1034768` (grams per ounce)  
      - `usdGram` computed from fetched price  
      - `rates` exchange rates from API  
      - `targets` array with target currency and USD for comparison  
      - `fmt(n, w)` formatting function for alignment and thousands separator  
      - `esc(txt)` escapes Markdown special chars  
    - Inputs: Combined JSON from "Merge" node  
    - Outputs: JSON object with key `telegram_text` containing formatted message  
    - Edge cases:  
      - Missing or malformed exchange rate for target currency defaults to 1 (USD).  
      - Unexpected input structure causes runtime errors.  
      - Markdown escaping might fail if unexpected characters appear.  
      - Percentage change formatting assumes numeric input.

---

#### 1.5 Message Sending

- **Overview:**  
  Sends the prepared Markdown message to the configured Telegram chat.

- **Nodes Involved:**  
  - Send a text message  
  - Sticky Note (send description)

- **Node Details:**

  - **Send a text message**  
    - Type: Telegram  
    - Role: Sends the text message to Telegram chat ID specified in inputs.  
    - Configuration:  
      - Text: `{{$json.telegram_text}}` (output from Report: Prepare)  
      - Chat ID: `{{$('Set Currency').item.json.telegram_chat_id}}`  
      - Parse mode: Markdown (Telegram Markdown Mode)  
      - Append attribution: false (no attribution text)  
    - Credentials: Telegram API credentials configured with OAuth2 or API token.  
    - Inputs: From "Report: Prepare"  
    - Outputs: None (terminal node)  
    - Edge cases:  
      - Invalid chat ID or revoked bot permissions cause message failure.  
      - Markdown parsing errors could cause message rejection.  
      - Telegram API rate limits and downtime.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                  | Input Node(s)        | Output Node(s)         | Sticky Note                                      |
|---------------------|-----------------------|--------------------------------|----------------------|------------------------|-------------------------------------------------|
| Schedule Trigger    | Schedule Trigger      | Starts workflow every 15 mins  | None                 | Set Currency           | ## Set inputs - currency                         |
| Set Currency        | Set                   | Sets currency, chat ID from env| Schedule Trigger     | Fetch Price, Convert Currency | ## Set inputs - currency                      |
| Fetch Price         | HTTP Request          | Gets gold price in USD/oz      | Set Currency          | Merge                  | ## Pricing                                      |
| Convert Currency    | HTTP Request          | Gets currency exchange rates   | Set Currency          | Merge                  | ## Currrency                                    |
| Merge               | Merge                 | Combines gold price and rates  | Fetch Price, Convert Currency | Report: Prepare    |                                                 |
| Report: Prepare     | Code (JavaScript)     | Formats price report for Telegram| Merge                | Send a text message     | ## Report                                       |
| Send a text message | Telegram              | Sends formatted message        | Report: Prepare       | None                   | ## Send                                         |
| Sticky Note         | Sticky Note           | Notes input parameters         | None                  | None                   | ## Set inputs - currency                         |
| Sticky Note3        | Sticky Note           | Notes report section           | None                  | None                   | ## Report                                       |
| Sticky Note4        | Sticky Note           | Notes send section             | None                  | None                   | ## Send                                         |
| Sticky Note5        | Sticky Note           | Notes pricing section          | None                  | None                   | ## Pricing                                      |
| Sticky Note6        | Sticky Note           | Notes currency section         | None                  | None                   | ## Currrency                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set interval to trigger every 15 minutes.

2. **Create a Set node named "Set Currency":**  
   - Add two string fields:  
     - `currency` with value `{{$env.currency}}`  
     - `telegram_chat_id` with value `{{$env.telegram_chat_id}}`  
   - Connect Schedule Trigger output to this node.

3. **Create an HTTP Request node named "Fetch Price":**  
   - Method: GET  
   - URL: `https://data-asg.goldprice.org/dbXRates/USD`  
   - Timeout: 5000ms  
   - Add HTTP header:  
     - `User-Agent`: `Mozilla/5.0 (iPhone; CPU iPhone OS 16_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.6 Mobile/15E148 Safari/604.1`  
   - Enable retry on failure.  
   - Connect "Set Currency" output to this node.

4. **Create an HTTP Request node named "Convert Currency":**  
   - Method: GET  
   - URL: `https://open.er-api.com/v6/latest/USD`  
   - Connect "Set Currency" output to this node.

5. **Create a Merge node named "Merge":**  
   - Mode: Combine  
   - Combine By: Position  
   - Connect "Fetch Price" output to input 1, "Convert Currency" output to input 2.

6. **Create a Code node named "Report: Prepare":**  
   - Language: JavaScript  
   - Paste the provided code (see Node Details in section 2.4) that:  
     - Extracts gold price and exchange rates  
     - Converts prices per gram and per karat  
     - Formats a Markdown V2 table string  
     - Outputs JSON with key `telegram_text` containing the formatted message  
   - Connect "Merge" output to this node.

7. **Create a Telegram node named "Send a text message":**  
   - Configure Telegram credentials (API token or OAuth2) with required permissions.  
   - Set Chat ID as expression: `{{$('Set Currency').item.json.telegram_chat_id}}`  
   - Set Text as expression: `{{$json.telegram_text}}`  
   - Set Parse Mode to Markdown.  
   - Disable Append Attribution.  
   - Connect "Report: Prepare" output to this node.

8. **Save and activate the workflow.**

9. **Environment Variables Setup:**  
   - Define `currency` environment variable with the target currency code (e.g., "EGP").  
   - Define `telegram_chat_id` environment variable with the Telegram chat ID for message delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                     |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Markdown V2 formatting with escaping is critical for proper Telegram message display without errors.    | See Telegram Markdown V2 documentation for message formatting.    |
| Gold price API uses ounces; conversion to grams uses precise factor 31.1034768.                         | Ensures accurate metric conversions for global users.            |
| Currency conversion API endpoint: https://open.er-api.com/v6/latest/USD                                | Reliable free exchange rate API used for conversion.              |
| Telegram API credential setup requires a bot token with permissions to send messages to the target chat.| https://core.telegram.org/bots/api#authorizing-your-bot           |
| Workflow is designed to run every 15 minutes; adjust Schedule Trigger node as needed for different frequency.|                                                             |

---

**Disclaimer:** The provided content is extracted exclusively from an automated workflow created in n8n, adhering strictly to current content policies and containing no illegal or offensive elements. All handled data is legal and public.