Scheduled Currency Exchange Rate Updates to Telegram with Exchange Rate API

https://n8nworkflows.xyz/workflows/scheduled-currency-exchange-rate-updates-to-telegram-with-exchange-rate-api-6185


# Scheduled Currency Exchange Rate Updates to Telegram with Exchange Rate API

---

### 1. Workflow Overview

This workflow automates the retrieval and reporting of currency exchange rates, sending formatted updates to a Telegram chat every 15 minutes. It targets users or systems needing timely currency exchange information for specific currencies relative to a base currency.

The workflow is logically divided into these blocks:

- **1.1 Schedule Trigger:** Periodic initiation every 15 minutes to start the process.
- **1.2 Input Setup:** Defines the base currency, target currencies, and Telegram chat ID.
- **1.3 Currency Conversion API Call:** Fetches the latest exchange rates from an external API.
- **1.4 Report Preparation:** Processes API data to build a human-readable currency exchange report enriched with emojis.
- **1.5 Telegram Message Sending:** Sends the prepared report message to a specified Telegram chat.

Each block is connected sequentially to ensure smooth data flow from trigger to notification delivery.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  Automatically starts the workflow every 15 minutes to keep currency exchange information updated.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Name:** Schedule Trigger  
  - **Type:** Schedule Trigger (n8n-nodes-base.scheduleTrigger)  
  - **Configuration:**  
    - Interval set to trigger every 15 minutes.  
  - **Input/Output:**  
    - No input (trigger node).  
    - Output connected to the "Set Currency" node to initiate the currency request.  
  - **Version Requirements:**  
    - Uses n8n version supporting scheduleTrigger v1.2.  
  - **Potential Failures:**  
    - Workflow may not trigger if n8n scheduler is disabled or misconfigured.  
    - Timezone issues can affect execution timing but not critical here.  

---

#### 1.2 Input Setup

- **Overview:**  
  Sets up essential input parameters: base currency code, target currencies, and Telegram chat ID for message delivery.

- **Nodes Involved:**  
  - Set Currency

- **Node Details:**  
  - **Name:** Set Currency  
  - **Type:** Set (n8n-nodes-base.set)  
  - **Configuration:**  
    - Assigns three string variables:  
      - `currency`: base currency code set to "EGP" (Egyptian Pound).  
      - `to_currencies`: list of target currency codes ("USD,AED,SAR") — although not directly used downstream, useful for future extension.  
      - `telegram_chat_id`: dynamically set from environment variable `$env.telegram_chat_id`.  
  - **Key Expressions:** Uses expression to read env variable for chat ID.  
  - **Input/Output:**  
    - Input: from Schedule Trigger.  
    - Output: to Convert Currency HTTP Request node.  
  - **Version Requirements:**  
    - Uses Set node v3.4.  
  - **Potential Failures:**  
    - Missing or incorrect environment variable `telegram_chat_id` will cause message sending failure.  
    - Hardcoded currency "EGP" limits flexibility unless modified.  
  - **Sticky Note:** "## Set inputs - currency" (Instructional, clarifies purpose)

---

#### 1.3 Currency Conversion API Call

- **Overview:**  
  Retrieves the latest exchange rates for the base currency via an HTTP GET request from the Exchange Rate API.

- **Nodes Involved:**  
  - Convert Currency

- **Node Details:**  
  - **Name:** Convert Currency  
  - **Type:** HTTP Request (n8n-nodes-base.httpRequest)  
  - **Configuration:**  
    - Method: GET  
    - URL: Expression constructed dynamically as `https://open.er-api.com/v6/latest/{{ $json.currency }}` — uses the base currency from previous node.  
    - No additional headers or authentication configured (API is public).  
  - **Input/Output:**  
    - Input: from Set Currency node.  
    - Output: to Report: Prepare node.  
  - **Version Requirements:**  
    - HTTP Request node v4.2 or higher recommended for expression support.  
  - **Potential Failures:**  
    - API downtime or rate limiting can cause HTTP errors or empty responses.  
    - Invalid currency codes can lead to API errors or no data.  
    - Network issues causing timeouts.  
  - **Sticky Note:** "## Currrency" (Typo in sticky note content but denotes this is the currency data retrieval step)

---

#### 1.4 Report Preparation

- **Overview:**  
  Processes the raw exchange rate data to format a readable, emoji-enhanced report string suitable for Telegram's HTML parse mode.

- **Nodes Involved:**  
  - Report: Prepare

- **Node Details:**  
  - **Name:** Report: Prepare  
  - **Type:** Code (JavaScript) (n8n-nodes-base.code)  
  - **Configuration:**  
    - JavaScript code takes JSON from the API response.  
    - Extracts base currency and rates.  
    - Defines target currencies with associated flag emojis: USD 🇺🇸, EUR 🇪🇺, GBP 🇬🇧, SAR 🇸🇦, AED 🇦🇪.  
    - Constructs a multi-line report string showing the inverse exchange rate (1 target currency = ? base currency) with HTML bold tags (`<b>`) for emphasis.  
    - Adds a timestamp of last update in italic (`<i>`).  
    - Returns the report string as `telegram_text` in output JSON.  
  - **Key Expressions:** None outside the code; all logic internal to JS code node.  
  - **Input/Output:**  
    - Input: from Convert Currency node (API JSON).  
    - Output: to Send a text message node.  
  - **Version Requirements:**  
    - Code node v2 for JavaScript support.  
  - **Potential Failures:**  
    - API response structural changes can break code assumptions (missing fields).  
    - Date parsing issues if API returns unexpected date format.  
    - Empty or malformed rate data causes incomplete reports.  
  - **Sticky Note:** "## Report" (Denotes report formatting logic)

---

#### 1.5 Telegram Message Sending

- **Overview:**  
  Sends the formatted currency exchange report as an HTML message to a specified Telegram chat.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**  
  - **Name:** Send a text message  
  - **Type:** Telegram node (n8n-nodes-base.telegram)  
  - **Configuration:**  
    - Text: Uses expression to pass `telegram_text` from previous node output.  
    - Chat ID: Uses expression referencing `telegram_chat_id` from "Set Currency" node.  
    - Parse mode: HTML to render formatting tags properly.  
    - `appendAttribution` disabled to avoid extra footer.  
    - Credentials: Uses a preconfigured Telegram API credential (OAuth2 token or bot token), named "Telegram account".  
  - **Input/Output:**  
    - Input: from Report: Prepare node.  
    - Output: none (end of workflow).  
  - **Version Requirements:**  
    - Telegram node v1.2 or higher for stable chatId expressions and HTML parsing.  
  - **Potential Failures:**  
    - Invalid or expired Telegram bot token credentials cause auth failures.  
    - Incorrect chat ID leads to message delivery failure.  
    - Telegram API rate limits or downtime.  
  - **Sticky Note:** "## Send" (Indicates sending step)

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role              | Input Node(s)         | Output Node(s)       | Sticky Note                    |
|--------------------|---------------------|-----------------------------|-----------------------|----------------------|-------------------------------|
| Schedule Trigger    | Schedule Trigger    | Starts workflow every 15 min| —                     | Set Currency         |                               |
| Set Currency       | Set                 | Define base currency, targets, Telegram chat ID | Schedule Trigger     | Convert Currency     | ## Set inputs - currency       |
| Convert Currency   | HTTP Request        | Fetch exchange rates         | Set Currency           | Report: Prepare      | ## Currrency                  |
| Report: Prepare    | Code (JavaScript)   | Format exchange rate report  | Convert Currency       | Send a text message  | ## Report                     |
| Send a text message | Telegram            | Send report to Telegram chat | Report: Prepare        | —                    | ## Send                       |
| Sticky Note        | Sticky Note         | Documentation only           | —                      | —                    | ## Set inputs - currency       |
| Sticky Note3       | Sticky Note         | Documentation only           | —                      | —                    | ## Report                     |
| Sticky Note4       | Sticky Note         | Documentation only           | —                      | —                    | ## Send                       |
| Sticky Note6       | Sticky Note         | Documentation only           | —                      | —                    | ## Currrency                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 15 minutes.  
   - No authentication needed.  

2. **Create Set Currency Node**  
   - Type: Set  
   - Connect input from Schedule Trigger node.  
   - Assign variables:  
     - `currency` (string): `"EGP"`  
     - `to_currencies` (string): `"USD,AED,SAR"` (can be modified or extended)  
     - `telegram_chat_id` (string): Use expression `{{$env.telegram_chat_id}}` to fetch from environment variable.  
   - No credentials required.  

3. **Create HTTP Request Node "Convert Currency"**  
   - Type: HTTP Request  
   - Connect input from Set Currency node.  
   - Method: GET  
   - URL: Use expression `https://open.er-api.com/v6/latest/{{ $json.currency }}` to dynamically insert base currency.  
   - No authentication or headers needed (public API).  
   - Set response format to JSON (default).  

4. **Create Code Node "Report: Prepare"**  
   - Type: Code (JavaScript)  
   - Connect input from Convert Currency node.  
   - Paste the following JS code:  
     ```javascript
     const data = $json;
     const baseCode = data.base_code;
     const rates = data.rates;
     const lastUpdate = new Date(data.time_last_update_utc).toLocaleDateString('en-US', {
       month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit', hour12: true
     });

     const targets = [
       { code: 'USD', emoji: '🇺🇸' },
       { code: 'EUR', emoji: '🇪🇺' },
       { code: 'GBP', emoji: '🇬🇧' },
       { code: 'SAR', emoji: '🇸🇦' },
       { code: 'AED', emoji: '🇦🇪' },
     ];

     let report = `<b>💱 Currency Exchange Rates</b>\n\n`;

     for (const target of targets) {
       const rate = rates[target.code];
       if (rate) {
         const inverseRate = (1 / rate).toFixed(2);
         report += `${target.emoji} 1 ${target.code} = <b>${inverseRate} ${baseCode}</b>\n`;
       }
     }

     report += `\n<i>Last updated: ${lastUpdate}</i>`;

     return [{ json: { telegram_text: report } }];
     ```  
   - No credentials needed.  

5. **Create Telegram Node "Send a text message"**  
   - Type: Telegram  
   - Connect input from Report: Prepare node.  
   - Set Text field to expression: `={{ $json.telegram_text }}`  
   - Set Chat ID to expression: `={{ $('Set Currency').item.json.telegram_chat_id }}`  
   - Under Additional Fields:  
     - Parse Mode: HTML  
     - Disable appendAttribution (set false)  
   - Set up Telegram API credentials (bot token or OAuth2) named "Telegram account" in n8n credentials.  
   - Save and activate workflow for testing.  

6. **Optional:** Add Sticky Notes for documentation near each logical block with the content as described in section 2.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                      |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The Telegram chat ID is set via environment variable `telegram_chat_id` for security and flexibility. | Environment Variable Setup                                          |
| Uses public Exchange Rate API: https://open.er-api.com/                                         | Exchange Rate API Documentation                                    |
| Message formatting uses HTML parse mode in Telegram for bold and italic styling.                 | Telegram Bot API Parse Modes: https://core.telegram.org/bots/api#html-style  |
| Emoji flags add visual clarity for currency codes enhancing message readability.                 | Unicode Regional Indicator Symbols                                 |
| Schedule interval can be adjusted as needed for trade-off between update frequency and API limits.| n8n Schedule Trigger Documentation                                 |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---