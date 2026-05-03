Daily Crypto Yield Monitor: Track Top Binance Earn APY Rates with Earnings Calculator

https://n8nworkflows.xyz/workflows/daily-crypto-yield-monitor--track-top-binance-earn-apy-rates-with-earnings-calculator-11426


# Daily Crypto Yield Monitor: Track Top Binance Earn APY Rates with Earnings Calculator

### 1. Workflow Overview

This workflow, titled **Daily Crypto Yield Monitor: Track Top Binance Earn APY Rates with Earnings Calculator**, is designed to automate the daily retrieval and analysis of Binance Flexible Earn product data. It securely authenticates to the Binance API, fetches the latest annual percentage yields (APYs) for up to 100 flexible earn assets, identifies the top 15 by APY, calculates potential daily earnings based on a simulated investment of $10,000, and emails a formatted report summarizing these findings.

The workflow is logically divided into four main blocks:

- **1.1 Authentication & API Fetch:** Handles secure authentication by generating a timestamp and HMAC-SHA256 signature, then fetches live flexible earn rate data from Binance.
- **1.2 Data Processing:** Splits the retrieved data into individual asset entries, extracts relevant fields, sorts assets by APY in descending order, and limits the list to the top 15 assets.
- **1.3 Analysis & Report Generation:** Calculates estimated daily earnings per asset and compiles an HTML email report.
- **1.4 Scheduling & Notification:** Triggers the entire process once every 24 hours and sends the final report via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Authentication & API Fetch

**Overview:**  
This block generates the necessary timestamp and cryptographic signature to authenticate API requests securely, then retrieves the latest flexible earn product data from Binance.

**Nodes Involved:**  
- Daily Trigger1  
- Prepare Data1  
- Set Credentials1  
- Sign Request1  
- Get Earn Rates1

**Node Details:**

- **Daily Trigger1**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow once every 24 hours.  
  - *Configuration:* Interval set to 24 hours (daily trigger).  
  - *Inputs:* None  
  - *Outputs:* Starts flow to Prepare Data1  
  - *Edge cases:* Missed executions if n8n instance is down or paused.

- **Prepare Data1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Generates current timestamp and query string for API call.  
  - *Configuration:*  
    - Generates `timestamp` using `Date.now()`.  
    - Constructs query string in order: size=100, then timestamp.  
  - *Expressions:* Outputs `timestamp` and `queryString` used downstream.  
  - *Inputs:* Trigger from Daily Trigger1  
  - *Outputs:* To Set Credentials1  
  - *Edge cases:* If system clock is incorrect, may cause signature mismatch.

- **Set Credentials1**  
  - *Type:* Set  
  - *Role:* Stores Binance API key and secret key for signing and headers.  
  - *Configuration:*  
    - Two string fields: `API_KEY` and `SECRET_KEY`.  
    - User must replace placeholders with actual Binance API credentials.  
  - *Inputs:* From Prepare Data1  
  - *Outputs:* To Sign Request1  
  - *Edge cases:* Missing or incorrect credentials will cause API auth failures.

- **Sign Request1**  
  - *Type:* Crypto (HMAC SHA256)  
  - *Role:* Creates HMAC-SHA256 signature of the query string using the secret key.  
  - *Configuration:*  
    - Input value: query string from Prepare Data1.  
    - Secret: `SECRET_KEY` from Set Credentials1.  
    - Action: HMAC with SHA256.  
    - Output property: `signature`.  
  - *Inputs:* From Set Credentials1  
  - *Outputs:* To Get Earn Rates1  
  - *Edge cases:* Signature mismatch if query string or secret key is incorrect.

- **Get Earn Rates1**  
  - *Type:* HTTP Request  
  - *Role:* Calls Binance API endpoint to retrieve flexible earn rate data.  
  - *Configuration:*  
    - URL: `https://api.binance.com/sapi/v1/simple-earn/flexible/list`  
    - Query Parameters:  
      - `size=100` (max entries)  
      - `timestamp` from Prepare Data1  
      - `signature` from Sign Request1  
    - Headers include `X-MBX-APIKEY` set to API key.  
  - *Inputs:* From Sign Request1  
  - *Outputs:* To Split Out1  
  - *Edge cases:* API rate limits, invalid credentials, network timeouts, Binance API downtime.

---

#### 2.2 Data Processing

**Overview:**  
This block processes the raw API response by splitting the asset list, extracting and normalizing relevant fields, sorting by APY, and limiting to the top 15 assets.

**Nodes Involved:**  
- Split Out1  
- Edit Fields1  
- Sort1  
- Limit1

**Node Details:**

- **Split Out1**  
  - *Type:* Split Out  
  - *Role:* Splits the array of asset data (`rows` field) into individual items for downstream processing.  
  - *Configuration:*  
    - Field to split: `rows` (response data array).  
  - *Inputs:* From Get Earn Rates1  
  - *Outputs:* To Edit Fields1  
  - *Edge cases:* Empty or malformed `rows` field causes no output.

- **Edit Fields1**  
  - *Type:* Set  
  - *Role:* Extracts and renames relevant fields from each asset object.  
  - *Configuration:*  
    - Sets two fields:  
      - `asset` (string) from original JSON `asset` field  
      - `latestAnnualPercentageRate` (string) from original field with same name  
  - *Inputs:* From Split Out1  
  - *Outputs:* To Sort1  
  - *Edge cases:* Missing fields in API response cause empty values.

- **Sort1**  
  - *Type:* Sort  
  - *Role:* Sorts the assets descending by APY.  
  - *Configuration:*  
    - Sort field: `latestAnnualPercentageRate`  
    - Order: Descending  
  - *Inputs:* From Edit Fields1  
  - *Outputs:* To Limit1  
  - *Edge cases:* Non-numeric or malformed APYs might disrupt sorting.

- **Limit1**  
  - *Type:* Limit  
  - *Role:* Restricts the list to the top 15 assets.  
  - *Configuration:*  
    - Max items: 15  
  - *Inputs:* From Sort1  
  - *Outputs:* To Filter & Analyze1  
  - *Edge cases:* If less than 15 assets, outputs all available.

---

#### 2.3 Analysis & Report Generation

**Overview:**  
This block calculates estimated daily earnings for each top asset based on a $10,000 simulated investment and generates a formatted HTML email report.

**Nodes Involved:**  
- Filter & Analyze1  
- Send Email1

**Node Details:**

- **Filter & Analyze1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Calculates daily income per asset and formats APY percentage.  
  - *Configuration:*  
    - Defines capital as $10,000 (simulation amount).  
    - For each asset:  
      - Parses APY from string to float.  
      - Calculates daily income = (capital * APY) / 365.  
      - Formats APY as percentage string with two decimals.  
      - Formats daily income as dollar string with two decimals.  
  - *Inputs:* From Limit1  
  - *Outputs:* To Send Email1  
  - *Edge cases:* Parsing errors if APY is not numeric or missing.

- **Send Email1**  
  - *Type:* Gmail  
  - *Role:* Sends the HTML report email summarizing the top APY assets and estimated earnings.  
  - *Configuration:*  
    - Subject: "Binance Yield Update"  
    - Message: HTML list of assets with APY and daily earnings, dynamically generated from input data.  
    - Credentials: Uses Gmail OAuth2 credential (configured externally).  
    - Execute Once: true (runs one email per execution)  
  - *Inputs:* From Filter & Analyze1  
  - *Outputs:* None (end node)  
  - *Edge cases:* Email send failure due to invalid credentials, OAuth expiration, or network issues.

---

#### 2.4 Workflow Annotation (Sticky Notes)

- **Main Sticky:** Overview, setup instructions, and general description of the workflow purpose.  
- **Auth Group:** Highlights the authentication and API fetch logic.  
- **Processing Group:** Describes the data processing steps: splitting, field extraction, sorting, and limiting.  
- **Reporting Group:** Explains the analysis and report generation block.

---

### 3. Summary Table

| Node Name        | Node Type            | Functional Role                        | Input Node(s)       | Output Node(s)    | Sticky Note                                                                                                             |
|------------------|----------------------|-------------------------------------|---------------------|-------------------|-------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger1   | Schedule Trigger      | Triggers workflow daily              | None                | Prepare Data1     | ## 🔐 Authentication & API Fetch (group sticky note covers this node)                                                  |
| Prepare Data1    | Code                 | Generates timestamp & query string   | Daily Trigger1      | Set Credentials1  | ## 🔐 Authentication & API Fetch                                                                                         |
| Set Credentials1 | Set                   | Stores Binance API credentials       | Prepare Data1       | Sign Request1     | ## 🔐 Authentication & API Fetch                                                                                         |
| Sign Request1    | Crypto (HMAC SHA256)  | Signs query string for API call      | Set Credentials1    | Get Earn Rates1   | ## 🔐 Authentication & API Fetch                                                                                         |
| Get Earn Rates1  | HTTP Request          | Fetches Binance flexible earn rates  | Sign Request1       | Split Out1        | ## 🔐 Authentication & API Fetch                                                                                         |
| Split Out1       | Split Out             | Splits API response array into items | Get Earn Rates1     | Edit Fields1      | ## 🧹 Data Processing (group sticky note covers this and following nodes)                                                |
| Edit Fields1     | Set                   | Extracts relevant fields             | Split Out1          | Sort1             | ## 🧹 Data Processing                                                                                                    |
| Sort1            | Sort                  | Sorts assets by APY descending       | Edit Fields1        | Limit1            | ## 🧹 Data Processing                                                                                                    |
| Limit1           | Limit                 | Limits to top 15 assets              | Sort1               | Filter & Analyze1 | ## 🧹 Data Processing                                                                                                    |
| Filter & Analyze1| Code                  | Calculates earnings & formats APY    | Limit1              | Send Email1       | ## 📊 Analysis & Report (group sticky note covers this and Send Email1)                                                  |
| Send Email1      | Gmail                 | Sends HTML report email              | Filter & Analyze1   | None              | ## 📊 Analysis & Report                                                                                                  |
| Main Sticky      | Sticky Note           | Workflow overview and setup guide    | None                | None              | ## 🚀 High-Yield Crypto Monitor with setup steps                                                                        |
| Auth Group       | Sticky Note           | Authentication & API Fetch description| None                | None              | Covers Daily Trigger1 to Get Earn Rates1                                                                               |
| Processing Group | Sticky Note           | Data processing explanation          | None                | None              | Covers Split Out1 to Limit1                                                                                              |
| Reporting Group  | Sticky Note           | Reporting and analysis explanation   | None                | None              | Covers Filter & Analyze1 and Send Email1                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an empty new workflow in n8n.**

2. **Add a "Schedule Trigger" node named `Daily Trigger1`:**  
   - Set the trigger to execute every 24 hours (hours interval = 24).  
   - This node starts the daily monitoring process.

3. **Add a "Code" node named `Prepare Data1`:**  
   - Connect it to `Daily Trigger1`.  
   - JavaScript code:  
     ```js
     const timestamp = Date.now();
     const queryString = `size=100&timestamp=${timestamp}`;
     return {
       timestamp,
       queryString
     };
     ```  
   - This prepares the current timestamp and query string for API signing.

4. **Add a "Set" node named `Set Credentials1`:**  
   - Connect to `Prepare Data1`.  
   - Add two string fields:  
     - `API_KEY` with placeholder: `YOUR_BINANCE_API_KEY`  
     - `SECRET_KEY` with placeholder: `YOUR_BINANCE_SECRET_KEY`  
   - User must replace these with actual Binance API credentials.

5. **Add a "Crypto" node named `Sign Request1`:**  
   - Connect to `Set Credentials1`.  
   - Configure as:  
     - Type: SHA256  
     - Action: HMAC  
     - Value: Expression referencing query string from `Prepare Data1` (`{{$node["Prepare Data1"].json.queryString}}`)  
     - Secret: Expression referencing `SECRET_KEY` from current node (`{{$json.SECRET_KEY}}`)  
     - Store signature in property named `signature`.

6. **Add an "HTTP Request" node named `Get Earn Rates1`:**  
   - Connect to `Sign Request1`.  
   - Configure:  
     - Method: GET  
     - URL: `https://api.binance.com/sapi/v1/simple-earn/flexible/list`  
     - Query parameters:  
       - `size` = `100`  
       - `timestamp` = Expression from `Prepare Data1` (`{{$node["Prepare Data1"].json.timestamp}}`)  
       - `signature` = Expression from `Sign Request1` (`{{$json.signature}}`)  
     - Add header parameter: `X-MBX-APIKEY` with value from `API_KEY` (`{{$json.API_KEY}}`)  
     - Ensure query parameters and headers are sent correctly.

7. **Add a "Split Out" node named `Split Out1`:**  
   - Connect to `Get Earn Rates1`.  
   - Configure to split the field `rows` (which contains the array of earn products).

8. **Add a "Set" node named `Edit Fields1`:**  
   - Connect to `Split Out1`.  
   - Configure to keep and rename fields:  
     - `asset` from JSON field `asset`  
     - `latestAnnualPercentageRate` from JSON field `latestAnnualPercentageRate`

9. **Add a "Sort" node named `Sort1`:**  
   - Connect to `Edit Fields1`.  
   - Configure to sort descending by `latestAnnualPercentageRate` (numeric sorting recommended).

10. **Add a "Limit" node named `Limit1`:**  
    - Connect to `Sort1`.  
    - Set max items to 15.

11. **Add a "Code" node named `Filter & Analyze1`:**  
    - Connect to `Limit1`.  
    - JavaScript code:  
      ```js
      const capital = 10000; // Investment simulation amount
      const results = [];

      for (const item of $input.all()) {
        const apy = parseFloat(item.json.latestAnnualPercentageRate);
        results.push({
          asset: item.json.asset,
          apy_percent: (apy * 100).toFixed(2) + '%',
          daily_income: '$' + (capital * apy / 365).toFixed(2),
        });
      }

      return results;
      ```

12. **Add a "Gmail" node named `Send Email1`:**  
    - Connect to `Filter & Analyze1`.  
    - Configure with Gmail OAuth2 credentials (set up in n8n credentials manager).  
    - Email subject: `Binance Yield Update`  
    - Email message (HTML):  
      ```html
      <h3>Daily Yield Report</h3>
      <ul>
      {{ $input.all().map(i => `
        <li>
          <b>${i.json.asset}</b>: ${i.json.apy_percent} APY 
          (Est. ${i.json.daily_income}/day on $10k)
        </li>
      `).join('') }}
      </ul>
      ```  
    - Set "Execute Once" to true.

13. **Add sticky notes for documentation:**  
    - Add a large sticky note near the start with workflow overview and setup steps (API keys, Gmail credentials, optional investment amount).  
    - Add sticky notes grouping nodes by their logical roles:  
      - Authentication & API Fetch  
      - Data Processing  
      - Analysis & Reporting

14. **Test the workflow:**  
    - Replace placeholders with your Binance API key and secret.  
    - Configure Gmail OAuth2 credentials correctly.  
    - Execute manually or wait for the scheduled trigger.  
    - Verify email receipt and content accuracy.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow monitors Binance Flexible Earn products daily and emails a report of top APY assets and earnings.| Workflow purpose described in main sticky note.                                                        |
| Setup requires Binance API Key and Secret for authentication and Gmail OAuth2 credentials for emailing. | Setup instructions in main sticky note.                                                               |
| Investment simulation amount ($10,000) can be adjusted in `Filter & Analyze1` node JavaScript code.      | Allows customizing earnings projections.                                                              |
| Binance API requires precise query string ordering and signing with HMAC-SHA256 for authenticated calls.| See Binance API docs for signature process: https://binance-docs.github.io/apidocs/spot/en/#signed-endpoints |
| Gmail node must be authorized with OAuth2 to send emails securely.                                       | See n8n Gmail OAuth2 credential setup guide: https://docs.n8n.io/credentials/gmail/                      |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.