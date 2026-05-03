Track and Analyze Forex News Trading Results with MyFxBook and Google Sheets

https://n8nworkflows.xyz/workflows/track-and-analyze-forex-news-trading-results-with-myfxbook-and-google-sheets-8522


# Track and Analyze Forex News Trading Results with MyFxBook and Google Sheets

### 1. Workflow Overview

This workflow automates the tracking and analysis of Forex news trading results by integrating data from Google Sheets and MyFxBook. It is designed to evaluate the profitability of trades based on news events, using historical high and low price data for currency pairs and precious metals, and updating the trade outcomes in a Google Sheets document.

Logical blocks of the workflow:

- **1.1 Schedule Trigger and Data Retrieval:** Periodically triggers the workflow, reads Forex trade data rows from a Google Sheet where certain price fields are empty.
- **1.2 MyFxBook Data Fetch and Parsing:** For each trade row, fetches historical high/low price data from MyFxBook via HTTP requests, then parses the HTML response into Markdown format.
- **1.3 Data Availability Check and Extraction:** Checks if new data for the next trading day (Date+1) is available, extracts the date and calculates multipliers based on currency pairs.
- **1.4 High/Low Price Extraction and Calculation:** Extracts high and low price values from the fetched data, then calculates points gained or lost based on the trade position.
- **1.5 Profit/Loss Determination and Sheet Update:** Determines if each trade resulted in a profit or loss based on position and points, updates the Google Sheet accordingly.
- **1.6 No Operation Nodes:** Several no-op nodes serve to gracefully end or branch the workflow without further processing.
- **1.7 Documentation Sticky Note:** Contains detailed instructions, use cases, limitations, and links related to the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger and Data Retrieval

- **Overview:** This block initiates the workflow on a scheduled interval and retrieves relevant rows from Google Sheets that need updating.
- **Nodes Involved:** Schedule Trigger, Get row(s) in sheet
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Periodically triggers the workflow (default: runs every minute/hour/day depending on configuration)
    - Configuration: Interval set to default (unspecified interval, depends on user setup)
    - Inputs: None (trigger node)
    - Outputs: Initiates downstream nodes
    - Failure considerations: If scheduling fails or is disabled, workflow won't run.

  - **Get row(s) in sheet**
    - Type: Google Sheets node (Read rows)
    - Role: Reads rows from Google Sheets where columns HIGH and LOW are empty (filters with "=" on HIGH and LOW)
    - Configuration:
      - Document ID: Google Sheets document "ForexFactory News Data"
      - Sheet ID: gid=0 (Sheet1)
      - Filters: HIGH = "" and LOW = ""
      - Credentials: Google Sheets OAuth2
    - Inputs: Trigger from Schedule Trigger
    - Outputs: Retrieved rows for which price data is missing
    - Failure considerations: Auth errors, API quota exceeded, missing or invalid credentials

#### 2.2 MyFxBook Data Fetch and Parsing

- **Overview:** Fetches historical high/low price data for currency pairs from MyFxBook and converts HTML to Markdown for easier parsing.
- **Nodes Involved:** HTTP Request, Markdown
- **Node Details:**

  - **HTTP Request**
    - Type: HTTP Request node
    - Role: Fetches historical price data from MyFxBook for the currency pair specified in the current row
    - Configuration:
      - URL Template: `https://www.myfxbook.com/forex-market/currencies/{{ $json.PAIRS }}-historical-data`
      - Method: GET (default)
      - No authentication required
    - Inputs: Rows from Google Sheets node
    - Outputs: HTML response from MyFxBook
    - Failure considerations: Network errors, 404 if currency pair not found, rate limiting by MyFxBook

  - **Markdown**
    - Type: Markdown node
    - Role: Converts the HTML response body from MyFxBook into Markdown text for easier extraction of data
    - Configuration:
      - Input: `={{ $json.data }}` (the HTML content from HTTP Request)
    - Inputs: HTTP Request output
    - Outputs: Markdown formatted text
    - Failure considerations: Parsing errors if HTML structure changes

#### 2.3 Data Availability Check and Extraction

- **Overview:** Checks if new high/low data for the next trading day is present in the fetched data, extracts the next date and calculates the multiplier for points calculation.
- **Nodes Involved:** Date+1 Data Available (If), Date, Multiplier (Set), No Operation, do nothing2 (NoOp)
- **Node Details:**

  - **Date+1 Data Available**
    - Type: If node
    - Role: Determines if the Markdown data contains the next trading day’s date (Date+1)
    - Condition: Checks that the substring around "Change (%) | | " does not contain the current row's DATE value, indicating new data
    - Inputs: Markdown output
    - Outputs: True branch proceeds with data extraction; false branch goes to no-op
    - Failure considerations: Expression errors if string indices invalid or missing data

  - **Date, Multiplier**
    - Type: Set node
    - Role: Extracts the next trading day’s date (DATE+1) from the Markdown text and assigns the appropriate points multiplier based on the currency pair
    - Configuration:
      - DATE+1: Extracted substring after "Change (%) | | ", parsed to get date string
      - POINTS MULTIPLIER: Conditional number – 1000 for USDJPY, 100 for XAUUSD, else 100000
    - Inputs: True output from If node
    - Outputs: Sets enriched data for next steps
    - Failure considerations: Parsing errors if expected strings are missing or malformed

  - **No Operation, do nothing2**
    - Type: No Operation node
    - Role: Terminates the false branch gracefully when no new data is available
    - Inputs: False output from If node
    - Outputs: None (end of branch)
    - Failure considerations: None (safe pass-through)

#### 2.4 High/Low Price Extraction and Calculation

- **Overview:** Extracts high and low price values from the Markdown data for DATE+1, then calculates points gained or lost based on the trade entry price and multiplier.
- **Nodes Involved:** High Low Price (Set), High, Low, Points Up, Points Down (Google Sheets update)
- **Node Details:**

  - **High Low Price**
    - Type: Set node
    - Role: Extracts numeric HIGH and LOW prices for DATE+1 from the Markdown text
    - Configuration:
      - HIGH: Extracted by substring search and split logic from the Markdown data
      - LOW: Similarly extracted from the same substring
    - Inputs: Output from Date, Multiplier node
    - Outputs: Sets HIGH and LOW values for update
    - Failure considerations: Parsing errors if data format changes or substring indices invalid

  - **High, Low, Points Up, Points Down**
    - Type: Google Sheets node (Update row)
    - Role: Updates the Google Sheet row with HIGH, LOW, and calculates POINTS UP and POINTS DOWN values using formulas:
      - POINTS UP = (HIGH - Entry PRICE) * POINTS MULTIPLIER
      - POINTS DOWN = (Entry PRICE - LOW) * POINTS MULTIPLIER
    - Configuration:
      - Matching column: row_number (to update correct row)
      - Document and Sheet IDs same as previous Google Sheets nodes
      - Columns updated: LOW, HIGH, POINTS UP, POINTS DOWN, and PROFIT/LOSS (empty at this stage)
    - Inputs: Output from High Low Price node
    - Outputs: Updated row data for downstream evaluation
    - Failure considerations: Auth issues, sheet locking, invalid row numbers

#### 2.5 Profit/Loss Determination and Sheet Update

- **Overview:** Determines the profitability of each trade based on the position (BUY/SELL) and points values, then updates the Google Sheet with "Profit" or "Loss" accordingly.
- **Nodes Involved:** 
  - Buy & Points Up > 0 = Profit (If)
  - Buy & Points Up <= 0 = Loss (If)
  - Sell & Points Down > 0 = Profit (If)
  - Sell & Points Down <= 0 = Loss (If)
  - Profit (Google Sheets update)
  - Loss (Google Sheets update)
  - No Operation, do nothing (NoOp)
- **Node Details:**

  - **Buy & Points Up > 0 = Profit**
    - Type: If node
    - Role: Checks if position is BUY and POINTS UP > 0, indicating profit
    - Inputs: High, Low, Points Up, Points Down node output
    - Outputs: True branch to Profit update, false to NoOp
    - Failure considerations: Expression evaluation errors

  - **Buy & Points Up <= 0 = Loss**
    - Type: If node
    - Role: Checks if position is BUY and POINTS UP ≤ 0, indicating loss
    - Inputs: Same as above
    - Outputs: True branch to Loss update, false to NoOp
    - Failure considerations: Expression evaluation errors

  - **Sell & Points Down > 0 = Profit**
    - Type: If node
    - Role: Checks if position is SELL and POINTS DOWN > 0, indicating profit
    - Inputs: Same as above
    - Outputs: True branch to Profit update, false to NoOp
    - Failure considerations: Expression evaluation errors

  - **Sell & Points Down <= 0 = Loss**
    - Type: If node
    - Role: Checks if position is SELL and POINTS DOWN ≤ 0, indicating loss
    - Inputs: Same as above
    - Outputs: True branch to Loss update, false to NoOp
    - Failure considerations: Expression evaluation errors

  - **Profit**
    - Type: Google Sheets node (Update row)
    - Role: Updates the PROFIT/LOSS column with "Profit" for the matching row
    - Configuration: Matches row_number, sets PROFIT/LOSS = "Profit"
    - Inputs: True outputs from profit If nodes
    - Outputs: Passes to No Operation node
    - Failure considerations: Auth/API issues

  - **Loss**
    - Type: Google Sheets node (Update row)
    - Role: Updates the PROFIT/LOSS column with "Loss" for the matching row
    - Configuration: Matches row_number, sets PROFIT/LOSS = "Loss"
    - Inputs: True outputs from loss If nodes
    - Outputs: Passes to No Operation node
    - Failure considerations: Auth/API issues

  - **No Operation, do nothing**
    - Type: No Operation node
    - Role: Ends branches gracefully after Profit/Loss update or when no condition matches
    - Inputs: From Profit/Loss updates or false branches of If nodes
    - Outputs: None
    - Failure considerations: None

#### 2.6 Documentation Sticky Note

- **Overview:** Contains detailed documentation embedded as a sticky note node in the workflow canvas.
- **Nodes Involved:** Sticky Note7
- **Node Details:**

  - **Sticky Note7**
    - Type: Sticky Note node
    - Role: Provides comprehensive documentation including:
      - The workflow purpose and usage
      - Use cases and limitations
      - Currency pairs supported
      - How the workflow operates step-by-step
      - Requirements like Google Drive API and credentials
      - Links to example Google Sheets and n8n community
    - Position: Off to side for information only
    - Failure considerations: None (informational)

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                                   | Input Node(s)               | Output Node(s)                                  | Sticky Note                                                  |
|------------------------------|---------------------|-------------------------------------------------|-----------------------------|-------------------------------------------------|--------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger    | Periodic workflow trigger                        | None                        | Get row(s) in sheet                             |                                                              |
| Get row(s) in sheet          | Google Sheets       | Retrieve sheet rows missing HIGH and LOW data   | Schedule Trigger            | HTTP Request                                   |                                                              |
| HTTP Request                | HTTP Request       | Fetch historical price data from MyFxBook       | Get row(s) in sheet         | Markdown                                       |                                                              |
| Markdown                    | Markdown           | Convert HTML response to Markdown for parsing    | HTTP Request               | Date+1 Data Available                          |                                                              |
| Date+1 Data Available       | If                 | Check if new Date+1 price data is available      | Markdown                   | Date, Multiplier; No Operation, do nothing2    |                                                              |
| Date, Multiplier            | Set                | Extract Date+1 and determine points multiplier   | Date+1 Data Available (true) | High Low Price                                 |                                                              |
| No Operation, do nothing2   | No Operation       | End branch if no new data                         | Date+1 Data Available (false) | None                                          |                                                              |
| High Low Price              | Set                | Extract HIGH and LOW prices from Markdown        | Date, Multiplier           | High, Low, Points Up, Points Down              |                                                              |
| High, Low, Points Up, Points Down | Google Sheets       | Update sheet with HIGH, LOW, and calculated points | High Low Price             | Buy & Points Up > 0 = Profit; Buy & Points Up <= 0 = Loss; Sell & Points Down > 0 = Profit; Sell & Points Down <= 0 = Loss |                                                              |
| Buy & Points Up > 0 = Profit | If                 | Check BUY position with positive points up (profit) | High, Low, Points Up, Points Down | Profit; No Operation, do nothing                |                                                              |
| Buy & Points Up <= 0 = Loss  | If                 | Check BUY position with zero or negative points up (loss) | High, Low, Points Up, Points Down | Loss; No Operation, do nothing                  |                                                              |
| Sell & Points Down > 0 = Profit | If                 | Check SELL position with positive points down (profit) | High, Low, Points Up, Points Down | Profit; No Operation, do nothing                |                                                              |
| Sell & Points Down <= 0 = Loss | If                 | Check SELL position with zero or negative points down (loss) | High, Low, Points Up, Points Down | Loss; No Operation, do nothing                  |                                                              |
| Profit                      | Google Sheets       | Update sheet row with "Profit" in PROFIT/LOSS   | Buy & Points Up > 0 = Profit; Sell & Points Down > 0 = Profit | No Operation, do nothing                         |                                                              |
| Loss                        | Google Sheets       | Update sheet row with "Loss" in PROFIT/LOSS     | Buy & Points Up <= 0 = Loss; Sell & Points Down <= 0 = Loss | No Operation, do nothing                         |                                                              |
| No Operation, do nothing    | No Operation       | Graceful branch termination after updates       | Profit; Loss; false branches of If nodes | None                                           |                                                              |
| Sticky Note7                | Sticky Note        | Embedded workflow documentation and instructions | None                        | None                                           | Contains detailed workflow explanation, use cases, and links |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Configure interval as desired (e.g., every hour or daily)
   - No inputs

2. **Add Google Sheets Node ("Get row(s) in sheet")**
   - Operation: Read rows
   - Document ID: Use your Google Sheets "ForexFactory News Data"
   - Sheet ID: `gid=0` (Sheet1)
   - Filters: HIGH = "" AND LOW = "" (to select rows missing price data)
   - Credentials: Set up Google Sheets OAuth2 credentials
   - Connect Schedule Trigger output to this node input

3. **Add HTTP Request Node**
   - URL: `https://www.myfxbook.com/forex-market/currencies/{{ $json.PAIRS }}-historical-data`
   - Method: GET
   - No authentication necessary
   - Connect output of Google Sheets node to this node

4. **Add Markdown Node**
   - Input expression: `={{ $json.data }}`
   - Converts HTML response to Markdown
   - Connect output of HTTP Request node to this node

5. **Add If Node ("Date+1 Data Available")**
   - Condition: Check that substring around "Change (%) |  | " in Markdown data does NOT contain current row's DATE value (indicating new date data)
   - Expression example:
     ```
     $json.data.substring($json.data.indexOf("Change (%) |  | "), $json.data.indexOf("Change (%) |  | ") + 100).includes($('Get row(s) in sheet').item.json.DATE) === false
     ```
   - Connect output of Markdown node to this If node

6. **Add Set Node ("Date, Multiplier")**
   - Assign two new fields:
     - `DATE+1`: Extract substring after "Change (%) |  | ", parsed to get next date string
     - `POINTS MULTIPLIER`: Conditional numeric value:
       - 1000 if PAIRS = "USDJPY"
       - 100 if PAIRS = "XAUUSD"
       - 100000 otherwise
   - Connect True output of If node to this node

7. **Add No Operation Node ("No Operation, do nothing2")**
   - Connect False output of If node here to end branch gracefully

8. **Add Set Node ("High Low Price")**
   - Assign:
     - `HIGH`: Extracted price from Markdown substring for DATE+1
     - `LOW`: Extracted price similarly
   - Connect output of "Date, Multiplier" node to this node

9. **Add Google Sheets Node ("High, Low, Points Up, Points Down")**
   - Operation: Update row
   - Document ID and Sheet ID as before
   - Matching column: `row_number`
   - Columns to update:
     - LOW: `={{ $json.LOW }}`
     - HIGH: `={{ $json.HIGH }}`
     - POINTS UP: `={{ ($json.HIGH - $('Get row(s) in sheet').item.json.PRICE) * $('Date, Multiplier').item.json['POINTS MULTIPLIER'] }}`
     - POINTS DOWN: `={{ ($('Get row(s) in sheet').item.json.PRICE - $json.LOW) * $json['POINTS MULTIPLIER'] }}`
     - PROFIT/LOSS: leave blank for now
   - Connect output of "High Low Price" node to this node

10. **Add Four If Nodes for Profit/Loss Determination:**

    - **"Buy & Points Up > 0 = Profit"**
      - Condition: POSITION = "BUY" AND POINTS UP > 0
      - Connect output of Google Sheets update node to this node

    - **"Buy & Points Up <= 0 = Loss"**
      - Condition: POSITION = "BUY" AND POINTS UP ≤ 0

    - **"Sell & Points Down > 0 = Profit"**
      - Condition: POSITION = "SELL" AND POINTS DOWN > 0

    - **"Sell & Points Down <= 0 = Loss"**
      - Condition: POSITION = "SELL" AND POINTS DOWN ≤ 0

11. **Add Two Google Sheets Update Nodes ("Profit" and "Loss")**

    - Both update the same sheet and match on `row_number`
    - "Profit" node sets PROFIT/LOSS column to "Profit"
    - "Loss" node sets PROFIT/LOSS column to "Loss"
    - Connect True outputs of corresponding If nodes to these update nodes

12. **Add No Operation Node ("No Operation, do nothing")**

    - Connect all outputs from Profit and Loss update nodes to this no-op node to gracefully end the workflow

13. **Add Sticky Note Node**

    - Paste full documentation, use cases, notes, and links as provided to assist users

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This n8n template simulates Forex news trading results using data from MyFxBook and Google Sheets. It is inspired by a previous workflow for Forex news alerts: https://n8n.io/workflows/8340-automated-forex-news-alert-system-with-forex-factory-and-telegram/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Original workflow inspiration link                                                                                  |
| Supported currency pairs include EURUSD, GBPUSD, AUDUSD, NZDUSD, USDJPY, USDCHF, USDCAD, and XAUUSD (Gold).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Currency pairs covered                                                                                              |
| Limitations include potential broker price discrepancies, spread and slippage variability, weekend price gaps, and dependency on trade closing time.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Trading limitations                                                                                                 |
| Requires Google Drive API enabled in Google Cloud Console and valid Google Sheets OAuth2 credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Google API setup requirement                                                                                        |
| Example Google Sheets file (template): https://docs.google.com/spreadsheets/d/1OhrbUQEc_lGegk5pRWWKz5nrnMbTZGT0lxK9aJqqId4/edit?usp=drive_link                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Example Google Sheets template                                                                                      |
| For help and community support, join the [Discord](https://discord.gg/n8n) or visit the [n8n Forum](https://community.n8n.io/)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Community resources                                                                                                |

---

This completes the detailed analysis, documentation, and reproduction instructions for the "Track and Analyze Forex News Trading Results with MyFxBook and Google Sheets" n8n workflow.