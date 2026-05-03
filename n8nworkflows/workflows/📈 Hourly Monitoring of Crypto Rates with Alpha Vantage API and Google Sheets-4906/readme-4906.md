📈 Hourly Monitoring of Crypto Rates with Alpha Vantage API and Google Sheets

https://n8nworkflows.xyz/workflows/---hourly-monitoring-of-crypto-rates-with-alpha-vantage-api-and-google-sheets-4906


# 📈 Hourly Monitoring of Crypto Rates with Alpha Vantage API and Google Sheets

### 1. Workflow Overview

This workflow performs an **hourly monitoring of cryptocurrency exchange rates** for Bitcoin (BTC) and Ethereum (ETH) against the Euro (EUR) by leveraging the Alpha Vantage API. The retrieved exchange rate data is then appended to a Google Sheet for historical tracking, followed by sending real-time notifications to a Telegram chat.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow execution every hour.
- **1.2 Data Retrieval:** Fetches current BTC and ETH exchange rates from the Alpha Vantage API.
- **1.3 Data Persistence:** Saves the fetched exchange rate data into designated Google Sheets.
- **1.4 Notification Dispatch:** Sends formatted Telegram messages to notify users of the latest rates.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution automatically every hour to ensure timely data retrieval and recording.

- **Nodes Involved:**  
  - Call Every Hour

- **Node Details:**

  - **Call Every Hour**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates the workflow on a recurring hourly interval.  
    - *Configuration:* Interval set to trigger every 1 hour.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Two parallel outputs feeding into BTC and ETH exchange rate HTTP requests.  
    - *Version:* 1.2  
    - *Potential Failures:* None expected unless system scheduling fails.  
    - *Notes:* No manual setup required; runs automatically on schedule.

#### 1.2 Data Retrieval

- **Overview:**  
  This block calls Alpha Vantage's `CURRENCY_EXCHANGE_RATE` API function twice to retrieve the current exchange rates of BTC and ETH against EUR.

- **Nodes Involved:**  
  - BTC Exchange Rate  
  - ETH Exchange Rate

- **Node Details:**

  - **BTC Exchange Rate**  
    - *Type:* HTTP Request  
    - *Role:* Queries Alpha Vantage API for BTC to EUR exchange rate.  
    - *Configuration:*  
      - HTTP GET to `https://www.alphavantage.co/query`.  
      - Query parameters:  
        - `function` = `CURRENCY_EXCHANGE_RATE`  
        - `from_currency` = `BTC`  
        - `to_currency` = `EUR`  
      - Authentication: HTTP Query Parameter Auth with API key (to be configured).  
    - *Inputs:* Trigger from "Call Every Hour" node.  
    - *Outputs:* JSON response forwarded to "Save Rate BTC" node.  
    - *Version:* 4.2  
    - *Potential Failures:*  
      - API key missing or invalid (authentication errors)  
      - Network timeouts or API rate limits  
      - Unexpected response format or empty data  
    - *Notes:* Requires Alpha Vantage API key configured in HTTP Request credentials.

  - **ETH Exchange Rate**  
    - *Type:* HTTP Request  
    - *Role:* Queries Alpha Vantage API for ETH to EUR exchange rate.  
    - *Configuration:* Similar to BTC node but with `from_currency` = `ETH`.  
    - *Inputs:* Trigger from "Call Every Hour" node.  
    - *Outputs:* JSON response forwarded to "Save Rate ETH" node.  
    - *Version:* 4.2  
    - *Potential Failures:* Same as BTC Exchange Rate node.  
    - *Notes:* Requires Alpha Vantage API key configured similarly.

#### 1.3 Data Persistence

- **Overview:**  
  This block appends the retrieved exchange rate data into specific Google Sheets for BTC and ETH respectively.

- **Nodes Involved:**  
  - Save Rate BTC  
  - Save Rate ETH

- **Node Details:**

  - **Save Rate BTC**  
    - *Type:* Google Sheets  
    - *Role:* Appends BTC exchange rate details as a new row in a designated Google Sheet tab.  
    - *Configuration:*  
      - Operation: Append  
      - Document ID: Specific Google Sheet file (ID provided)  
      - Sheet Name: `gid=0` (corresponding to BTC sheet)  
      - Columns mapped explicitly from the API response JSON fields:  
        - From_Currency_Code, From_Currency_Name, To_Currency_Code, To_Currency_Name, Exchange_Rate, Bid_Price, Ask_Price, Last_Refreshed, Time_Zone  
      - Mapping mode: Manual mapping of fields.  
    - *Inputs:* JSON data output from "BTC Exchange Rate".  
    - *Outputs:* Forward data to "Notification BTC" node.  
    - *Version:* 4.6  
    - *Potential Failures:*  
      - Google Sheets API authentication failure  
      - Sheet or document ID invalid or not accessible  
      - Data format mismatch or append failure  
    - *Notes:* Requires Google Sheets OAuth2 credentials with write permissions.

  - **Save Rate ETH**  
    - *Type:* Google Sheets  
    - *Role:* Same as "Save Rate BTC" but for ETH data.  
    - *Configuration:*  
      - Operation: Append  
      - Document ID: Same Google Sheet file as BTC  
      - Sheet Name: `1591416661` (corresponding to ETH sheet)  
      - Columns mapped similarly to BTC node.  
    - *Inputs:* JSON data output from "ETH Exchange Rate".  
    - *Outputs:* Forward data to "Notification ETH" node.  
    - *Version:* 4.6  
    - *Potential Failures:* Same as "Save Rate BTC".  
    - *Notes:* Shares same Google Sheets OAuth2 credentials.

#### 1.4 Notification Dispatch

- **Overview:**  
  This block sends formatted Telegram messages to a specified chat ID to notify about the latest BTC and ETH exchange rates.

- **Nodes Involved:**  
  - Notification BTC  
  - Notification ETH

- **Node Details:**

  - **Notification BTC**  
    - *Type:* Telegram  
    - *Role:* Sends a Telegram message with BTC rate details formatted as HTML.  
    - *Configuration:*  
      - Chat ID: Specific numeric Telegram chat ID (configured).  
      - Message text: HTML-formatted message dynamically populated with exchange rate details using expressions referencing incoming JSON.  
      - Additional fields: `parse_mode` set to `HTML`, disables attribution appending.  
    - *Inputs:* Data output from "Save Rate BTC".  
    - *Outputs:* None (end node).  
    - *Version:* 1.2  
    - *Potential Failures:*  
      - Telegram API authentication failure or invalid token  
      - Invalid chat ID  
      - Network errors or Telegram rate limits  
    - *Notes:* Requires Telegram Bot credentials configured correctly.

  - **Notification ETH**  
    - *Type:* Telegram  
    - *Role:* Sends a Telegram message with ETH rate details, same format as BTC notification.  
    - *Configuration:* Similar to "Notification BTC" but for ETH data.  
    - *Inputs:* Data output from "Save Rate ETH".  
    - *Outputs:* None.  
    - *Version:* 1.2  
    - *Potential Failures:* Same as "Notification BTC".  
    - *Notes:* Telegram credentials must be set up.

---

### 3. Summary Table

| Node Name          | Node Type          | Functional Role            | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                              |
|--------------------|--------------------|----------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------------------|
| Call Every Hour     | Schedule Trigger   | Hourly trigger             | None                  | BTC Exchange Rate, ETH Exchange Rate | "1. Workflow Trigger every hour\n\nTrigger the collection of ETH and BTC price every hour.\n\nHow to setup?\n*Nothing to do.*" |
| BTC Exchange Rate   | HTTP Request       | Fetch BTC to EUR rate      | Call Every Hour        | Save Rate BTC          | "2. Collect BTC and ETH price from Alpha Vantage Insight API\nDetails on API key setup and Google Sheets mapping.\nSee note for Telegram setup." |
| ETH Exchange Rate   | HTTP Request       | Fetch ETH to EUR rate      | Call Every Hour        | Save Rate ETH          | Same as BTC Exchange Rate node                                                                           |
| Save Rate BTC       | Google Sheets      | Append BTC data to sheet   | BTC Exchange Rate      | Notification BTC       | Same as BTC Exchange Rate node                                                                           |
| Save Rate ETH       | Google Sheets      | Append ETH data to sheet   | ETH Exchange Rate      | Notification ETH       | Same as BTC Exchange Rate node                                                                           |
| Notification BTC    | Telegram           | Send BTC rate notification| Save Rate BTC          | None                  | Same as BTC Exchange Rate node                                                                           |
| Notification ETH    | Telegram           | Send ETH rate notification| Save Rate ETH          | None                  | Same as BTC Exchange Rate node                                                                           |
| Sticky Note1        | Sticky Note        | Workflow trigger info      | None                  | None                  | "1. Workflow Trigger every hour\n\nTrigger the collection of ETH and BTC price every hour.\n\nHow to setup?\n*Nothing to do.*" |
| Sticky Note         | Sticky Note        | API, Google Sheets & Telegram setup instructions | None                  | None                  | "2. Collect BTC and ETH price from Alpha Vantage Insight API\nThis starts by calling the CURRENCY_EXCHANGE_RATE function of the **Alpha Vantage Insight API** to get the exchange rate to euros.\n\nHow to setup?\n- API key from Alpha Vantage\n- Google Sheets credentials and mapping\n- Telegram credentials and chat ID setup\n\n[Learn more about the Google Sheet Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets)" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger Node**  
   - Add a **Schedule Trigger** node.  
   - Set the interval to trigger every 1 hour (select "hours" and set value to 1).  
   - Name it "Call Every Hour".

2. **Create HTTP Request Node for BTC**  
   - Add an **HTTP Request** node.  
   - Name it "BTC Exchange Rate".  
   - Set Method to GET.  
   - URL: `https://www.alphavantage.co/query`  
   - Enable "Send Query" and add query parameters:  
     - `function`: `CURRENCY_EXCHANGE_RATE`  
     - `from_currency`: `BTC`  
     - `to_currency`: `EUR`  
   - Under Authentication, choose "HTTP Query Parameter Authentication". Provide your Alpha Vantage API key here.  
   - Connect "Call Every Hour" node output to this node input.

3. **Create HTTP Request Node for ETH**  
   - Add another **HTTP Request** node.  
   - Name it "ETH Exchange Rate".  
   - Configure similarly to BTC node but set `from_currency` to `ETH`.  
   - Connect "Call Every Hour" node output to this node input in parallel to the BTC node.

4. **Create Google Sheets Node for BTC Data**  
   - Add a **Google Sheets** node.  
   - Name it "Save Rate BTC".  
   - Set Operation to "Append".  
   - Provide Google Sheets OAuth2 credentials that have permission to edit your target spreadsheet.  
   - Set Document ID to your Google Sheet file ID (the one you want to store BTC data in).  
   - Set Sheet Name to `gid=0` or the appropriate sheet tab for BTC.  
   - Under Columns, map the following fields from the incoming JSON:  
     - From_Currency_Code: `{{$json["Realtime Currency Exchange Rate"]["1. From_Currency Code"]}}`  
     - From_Currency_Name: `{{$json["Realtime Currency Exchange Rate"]["2. From_Currency Name"]}}`  
     - To_Currency_Code: `{{$json["Realtime Currency Exchange Rate"]["3. To_Currency Code"]}}`  
     - To_Currency_Name: `{{$json["Realtime Currency Exchange Rate"]["4. To_Currency Name"]}}`  
     - Exchange_Rate: `{{$json["Realtime Currency Exchange Rate"]["5. Exchange Rate"]}}`  
     - Last_Refreshed: `{{$json["Realtime Currency Exchange Rate"]["6. Last Refreshed"]}}`  
     - Time_Zone: `{{$json["Realtime Currency Exchange Rate"]["7. Time Zone"]}}`  
     - Bid_Price: `{{$json["Realtime Currency Exchange Rate"]["8. Bid Price"]}}`  
     - Ask_Price: `{{$json["Realtime Currency Exchange Rate"]["9. Ask Price"]}}`  
   - Connect "BTC Exchange Rate" node output to this node input.

5. **Create Google Sheets Node for ETH Data**  
   - Add another **Google Sheets** node.  
   - Name it "Save Rate ETH".  
   - Configure similarly to the BTC Google Sheets node but select the sheet tab for ETH (e.g., `1591416661` or corresponding sheet name).  
   - Map the same fields as for BTC from the ETH exchange rate JSON.  
   - Connect "ETH Exchange Rate" node output to this node input.

6. **Create Telegram Node for BTC Notification**  
   - Add a **Telegram** node.  
   - Name it "Notification BTC".  
   - Provide Telegram Bot credentials (Bot token).  
   - Set "Chat ID" to the target chat's numeric ID.  
   - Set "Text" field with the following HTML message, using expressions to dynamically fill data:  
     ```
     <b>🔄 BTC to EUR Rate Update</b>

     <b>From:</b> {{ $json["From_Currency_Name"] }} ({{ $json["From_Currency_Code"] }})
     <b>To:</b> {{ $json["To_Currency_Name"] }} ({{ $json["To_Currency_Code"] }})

     <b>💱 Exchange Rate:</b> {{ $json["Exchange_Rate"] }}
     <b>📉 Bid:</b> {{ $json["Bid_Price"] }}
     <b>📈 Ask:</b> {{ $json["Ask_Price"] }}

     <b>🕒 Last Updated:</b> {{ $json["Last_Refreshed"] }} ({{ $json["Time_Zone"] }})
     ```
   - Set "Parse Mode" to `HTML`.  
   - Connect "Save Rate BTC" node output to this node input.

7. **Create Telegram Node for ETH Notification**  
   - Add another **Telegram** node.  
   - Name it "Notification ETH".  
   - Configure identically to the BTC Telegram node but adjust the message text header to "ETH to EUR Rate Update".  
   - Connect "Save Rate ETH" node output to this node input.

8. **Validate Credentials and Permissions**  
   - Ensure Alpha Vantage API key is valid and has enough free calls quota.  
   - Ensure Google Sheets OAuth2 credentials have edit access to the target spreadsheet.  
   - Ensure Telegram Bot token is valid and the bot is added to the target chat with permissions to send messages.

9. **Activate the Workflow**  
   - Save and activate the workflow.  
   - The workflow will run automatically every hour, fetching exchange rates, saving them to the Google Sheets, and sending Telegram notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Alpha Vantage API Key is required for accessing exchange rate data. Get a free API key at [Alpha Vantage](https://www.alphavantage.co/support/#api-key).                                                                                                                                                                                                          | Alpha Vantage API documentation and signup link                                                                                      |
| Google Sheets node requires OAuth2 credentials with edit permissions on the target spreadsheet. Refer to [n8n Google Sheets Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) for setup details.                                                                                                               | Official n8n Google Sheets node documentation                                                                                        |
| Telegram Bot must be created via BotFather, and the bot token should be configured in n8n credentials. The bot must be added to the target chat or channel with permission to send messages.                                                                                                                                                                     | Telegram Bot API documentation                                                                                                       |
| The workflow includes two sticky notes with setup instructions and explanations for easier maintenance and onboarding.                                                                                                                                                                                                                                          | Sticky notes content embedded in the workflow                                                                                         |
| Potential API rate limits on Alpha Vantage (typically 5 calls per minute for free tier) might cause failures if the workflow is triggered too frequently or if many requests are done. Consider upgrading or adding error handling for rate limit responses.                                                                                                       | Alpha Vantage API rate limiting details                                                                                              |
| The Telegram notification messages use HTML parse mode for rich formatting. Ensure that dynamic data is sanitized if extending the workflow.                                                                                                                                                                                                                     | Telegram parse mode details                                                                                                          |

---

**Disclaimer:**  
The text provided stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.