Automated Stock Trading with AI: Integrating Alpaca and Google Sheets

https://n8nworkflows.xyz/workflows/automated-stock-trading-with-ai--integrating-alpaca-and-google-sheets-5711


# Automated Stock Trading with AI: Integrating Alpaca and Google Sheets

### 1. Workflow Overview

This workflow automates stock trading by integrating sentiment analysis data with the Alpaca paper trading API and Google Sheets for record-keeping. It is designed to run daily after the US stock market opens, using sentiment scores to decide which stocks to buy or sell. The workflow manages portfolio adjustments by selling underperforming assets and buying stocks with the highest positive sentiment, logging all trades and account balances.

The workflow logic is organized into four main blocks:

- **1.1 Daily Trigger and Account Snapshot:** Scheduled daily trigger to fetch current Alpaca account info and record balance changes.
- **1.2 Sentiment-Based Stock Selection:** Reads daily sentiment scores from Google Sheets, filters top stocks, and fetches current open positions.
- **1.3 Trading Logic and Execution:** Compares current positions with top sentiment stocks to determine sell and buy orders, executes trades with necessary timing.
- **1.4 Logging and Record-Keeping:** Merges trade results and logs all orders and balances into Google Sheets for audit and tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger and Account Snapshot

**Overview:**  
Triggers the workflow daily at 16:45 Asia/Jerusalem time, fetches Alpaca paper trading account details, and records the account balance and daily change into a Google Sheet.

**Nodes Involved:**  
- Schedule Trigger  
- Alpaca-get-account-info  
- write_account_balace_today

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow daily at 16:45 (Asia/Jerusalem timezone)  
  - Config: Trigger time set to 16:45  
  - Inputs: None (start node)  
  - Outputs: Connects to Alpaca-get-account-info  
  - Failure modes: Trigger misconfiguration or timezone mismatch could delay execution  

- **Alpaca-get-account-info**  
  - Type: HTTP Request  
  - Role: Retrieves current Alpaca paper trading account info (equity, balance, etc.)  
  - Config: GET request to `https://paper-api.alpaca.markets/v2/account` with custom HTTP authentication credentials  
  - Inputs: From Schedule Trigger  
  - Outputs: Connects to write_account_balace_today  
  - Failure modes: API auth errors, rate limits, network failures  
  - Credentials: Alpaca API HTTP Custom Auth, plus HTTP Query Auth (for token or similar)  

- **write_account_balace_today**  
  - Type: Google Sheets  
  - Role: Logs current account balance and daily percent change in the "balance" sheet  
  - Config: Append or update mode keyed by date; writes fields: date, balance, change calculated as `(equity - last_equity)/last_equity`  
  - Inputs: Alpaca-get-account-info output JSON  
  - Outputs: Connects to read_sentiments_score_today  
  - Failure modes: Google Sheets auth, sheet access issues, data conversion errors  
  - Credentials: Google Sheets OAuth2  

---

#### 1.2 Sentiment-Based Stock Selection

**Overview:**  
Reads sentiment scores for the current day from Google Sheets, selects the top four stocks by sentiment, then obtains the list of currently open positions from Alpaca.

**Nodes Involved:**  
- read_sentiments_score_today  
- filter_top_sentiment_score  
- Alpaca_get_open_positions

**Node Details:**

- **read_sentiments_score_today**  
  - Type: Google Sheets  
  - Role: Reads sentiment data filtered by today's date from "sentiments" sheet  
  - Config: Uses filter on "date" column for today's date, reads from sheet with gid=0 in a specified Google Sheet document  
  - Inputs: From write_account_balace_today  
  - Outputs: Connects to filter_top_sentiment_score  
  - Failure modes: Sheet access, filter mismatches, empty data  

- **filter_top_sentiment_score**  
  - Type: Code (JavaScript)  
  - Role: Sorts sentiment data descending by sentimentScore, selects top 4 stocks  
  - Config: Custom JS code sorts all input items, slices top 4  
  - Inputs: From read_sentiments_score_today  
  - Outputs: Connects to Alpaca_get_open_positions  
  - Failure modes: JS errors if sentimentScore missing or non-numeric  

- **Alpaca_get_open_positions**  
  - Type: HTTP Request  
  - Role: Retrieves open positions from Alpaca account  
  - Config: GET request to `https://paper-api.alpaca.markets/v2/positions`, authenticated with Alpaca credentials  
  - Inputs: From filter_top_sentiment_score  
  - Outputs: Connects to create_positions_to_close_and_positions_two_open  
  - Failure modes: API errors, authentication failure, network issues  
  - Credentials: Same as Alpaca-get-account-info  

---

#### 1.3 Trading Logic and Execution

**Overview:**  
Compares currently held positions with top sentiment stocks, deciding which positions to close and which new positions to open. Executes sell orders, waits 2 minutes, then executes buy orders.

**Nodes Involved:**  
- create_positions_to_close_and_positions_two_open  
- positions_to_close  
- Alpaca-post-order-sell  
- Wait  
- positions_to_open  
- Alpaca-post-order-buy

**Node Details:**

- **create_positions_to_close_and_positions_two_open**  
  - Type: Code (JavaScript)  
  - Role: Core trading logic computing two lists: positions to close (held but not top sentiment) and positions to open (top sentiment not held)  
  - Config: Compares symbols, calculates total market value of positions to close, divides value equally among new positions  
  - Inputs: From Alpaca_get_open_positions (open positions) and filter_top_sentiment_score (top stocks)  
  - Outputs: Two outputs: positions_not_in_top_sentiment (to close), symbols_not_in_open_positions (to open, with allocated market value)  
  - Failure modes: JS errors if input data malformed, empty arrays, division by zero (handled by conditional)  

- **positions_to_close**  
  - Type: Split Out  
  - Role: Splits the array of positions to close into individual items for sequential processing  
  - Config: Splits on field “positions_not_in_top_sentiment”  
  - Inputs: From create_positions_to_close_and_positions_two_open  
  - Outputs: Connects to Alpaca-post-order-sell  
  - Failure modes: Empty input arrays lead to no output  

- **Alpaca-post-order-sell**  
  - Type: HTTP Request  
  - Role: Sends DELETE requests to Alpaca API to close positions based on asset_id  
  - Config: DELETE request to `https://paper-api.alpaca.markets/v2/positions/{{$json.asset_id}}`  
  - Inputs: From positions_to_close (individual positions)  
  - Outputs: Connects to merge_orders_to_write_to_sheets (output index 0)  
  - Failure modes: API errors, invalid asset_id, network issues  
  - Credentials: Alpaca HTTP Custom Auth  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 2 minutes after sell orders to ensure settlement before buying  
  - Config: 2 minutes delay  
  - Inputs: From positions_to_open  
  - Outputs: Connects to Alpaca-post-order-buy  
  - Failure modes: Unlikely, but workflow interruptions during wait could cause issues  

- **positions_to_open**  
  - Type: Split Out  
  - Role: Splits the list of symbols to open into individual buy orders  
  - Config: Splits on “symbols_not_in_open_positions” field  
  - Inputs: From create_positions_to_close_and_positions_two_open  
  - Outputs: Connects to Wait  
  - Failure modes: Empty input arrays cause no output  

- **Alpaca-post-order-buy**  
  - Type: HTTP Request  
  - Role: Sends POST requests to Alpaca to buy stocks with allocated notional value  
  - Config: POST to `https://paper-api.alpaca.markets/v2/orders` with body parameters: symbol, notional (market value per symbol), side=buy, type=market, time_in_force=day  
  - Inputs: From Wait  
  - Outputs: Connects to merge_orders_to_write_to_sheets (output index 1)  
  - Failure modes: API errors, invalid symbols, insufficient funds (unlikely due to prior sell), network errors  
  - Credentials: Alpaca HTTP Custom Auth  

---

#### 1.4 Logging and Record-Keeping

**Overview:**  
Collects results from buy and sell orders, merges them, and logs all trade details (date, symbol, order type, value) into the "positions" sheet of the Google Sheet for audit and tracking.

**Nodes Involved:**  
- merge_orders_to_write_to_sheets  
- Google Sheets (positions logging)

**Node Details:**

- **merge_orders_to_write_to_sheets**  
  - Type: Merge  
  - Role: Merges parallel outputs from buy and sell order nodes into a single stream  
  - Config: Default merge mode (likely "append")  
  - Inputs: From Alpaca-post-order-sell (main output 0) and Alpaca-post-order-buy (main output 1)  
  - Outputs: Connects to Google Sheets node for logging  
  - Failure modes: Unequal input sizes or timing issues could cause partial merges  

- **Google Sheets (positions logging)**  
  - Type: Google Sheets  
  - Role: Appends trade records to the "positions" sheet with fields: date, symbol, order (buy/sell), value (notional or market value)  
  - Config: Append mode, mapping columns explicitly, keyed by date  
  - Inputs: From merge_orders_to_write_to_sheets  
  - Outputs: None (end node)  
  - Failure modes: Google Sheets API limits, auth failures, data conversion errors  
  - Credentials: Google Sheets OAuth2  

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                           | Input Node(s)                   | Output Node(s)                     | Sticky Note                                                                                               |
|--------------------------------|----------------------|-----------------------------------------|--------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------|
| Schedule Trigger                | Schedule Trigger     | Daily workflow trigger                   | None                           | Alpaca-get-account-info          | ## 1. Daily Trigger and Account Snapshot 📈\nTrigger at 16:45 Asia/Jerusalem time after US market open    |
| Alpaca-get-account-info         | HTTP Request         | Fetch Alpaca account info                | Schedule Trigger               | write_account_balace_today       |                                                                                                          |
| write_account_balace_today      | Google Sheets        | Log daily account balance and changes   | Alpaca-get-account-info        | read_sentiments_score_today      |                                                                                                          |
| read_sentiments_score_today     | Google Sheets        | Read daily sentiment scores from Sheets | write_account_balace_today     | filter_top_sentiment_score       | ## 2. Sentiment-Based Stock Selection 👍👎\nReads sentiment scores for the day from Google Sheets          |
| filter_top_sentiment_score      | Code                 | Filter top 4 stocks by sentiment score  | read_sentiments_score_today    | Alpaca_get_open_positions        |                                                                                                          |
| Alpaca_get_open_positions       | HTTP Request         | Get current open positions on Alpaca    | filter_top_sentiment_score     | create_positions_to_close_and_positions_two_open |                                                                                                          |
| create_positions_to_close_and_positions_two_open | Code                 | Determine positions to sell and buy     | Alpaca_get_open_positions      | positions_to_close, positions_to_open | ## 3. Trading Logic and Execution ⚙️\nCore logic deciding positions to close and open based on sentiment   |
| positions_to_close             | Split Out            | Split sell positions for sequential sell| create_positions_to_close_and_positions_two_open | Alpaca-post-order-sell           |                                                                                                          |
| Alpaca-post-order-sell          | HTTP Request         | Send sell orders to Alpaca               | positions_to_close             | merge_orders_to_write_to_sheets  |                                                                                                          |
| positions_to_open              | Split Out            | Split buy positions for sequential buy  | create_positions_to_close_and_positions_two_open | Wait                            |                                                                                                          |
| Wait                          | Wait                 | Pause 2 minutes before buying            | positions_to_open              | Alpaca-post-order-buy            |                                                                                                          |
| Alpaca-post-order-buy           | HTTP Request         | Send buy orders to Alpaca                | Wait                          | merge_orders_to_write_to_sheets  |                                                                                                          |
| merge_orders_to_write_to_sheets | Merge                | Merge buy and sell order results         | Alpaca-post-order-sell, Alpaca-post-order-buy | Google Sheets                   | ## 4. Logging and Record-Keeping 📝\nMerges orders and logs all trades into Google Sheets                 |
| Google Sheets                  | Google Sheets        | Log trade orders into positions sheet    | merge_orders_to_write_to_sheets | None                           |                                                                                                          |
| Sticky Note1                  | Sticky Note          | Description of daily trigger and account snapshot logic | None | None | ## 1. Daily Trigger and Account Snapshot 📈\nDetailed explanation of trigger and account info logging      |
| Sticky Note2                  | Sticky Note          | Overview of entire Alpaca Trading workflow | None | None | ## Alpaca Trading 📈\nSummary of workflow purpose and connection to sentiment bot                         |
| Sticky Note3                  | Sticky Note          | Sentiment-based stock selection explanation | None | None | ## 2. Sentiment-Based Stock Selection 👍👎\nDetails on reading and filtering sentiment scores               |
| Sticky Note4                  | Sticky Note          | Explanation of trading logic and execution | None | None | ## 3. Trading Logic and Execution ⚙️\nDetails on deciding, selling, waiting, buying steps                  |
| Sticky Note5                  | Sticky Note          | Logging and record-keeping explanation   | None | None | ## 4. Logging and Record-Keeping 📝\nDetails on merging orders and logging to Google Sheets                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set timezone to Asia/Jerusalem  
   - Set trigger time: 16:45 daily  

2. **Create HTTP Request node "Alpaca-get-account-info"**  
   - Method: GET  
   - URL: `https://paper-api.alpaca.markets/v2/account`  
   - Authentication: HTTP Custom Auth with Alpaca API credentials  
   - Link from Schedule Trigger output to this node input  

3. **Create Google Sheets node "write_account_balace_today"**  
   - Operation: Append or Update  
   - Document ID: Your Google Sheet containing "balance" sheet  
   - Sheet name: "balance" (gid: 449152551)  
   - Columns mapped:  
     - date: `={{$today}}`  
     - balance: `={{$json.equity}}`  
     - change: `={{($json.equity - $json.last_equity) / $json.last_equity}}`  
   - Credentials: Google Sheets OAuth2  
   - Connect output of Alpaca-get-account-info to this node  

4. **Create Google Sheets node "read_sentiments_score_today"**  
   - Operation: Read rows with filter  
   - Document ID: Your Google Sheet with sentiment data  
   - Sheet name: "sentiments" (gid=0)  
   - Filter rows where "date" equals today’s date `={{$today}}`  
   - Credentials: Google Sheets OAuth2  
   - Connect output of write_account_balace_today to this node  

5. **Create Code node "filter_top_sentiment_score"**  
   - Paste JavaScript to sort input items by `sentimentScore` descending and slice top 4  
   - Connect output of read_sentiments_score_today to this node  

6. **Create HTTP Request node "Alpaca_get_open_positions"**  
   - Method: GET  
   - URL: `https://paper-api.alpaca.markets/v2/positions`  
   - Authentication: Same Alpaca HTTP Custom Auth  
   - Connect output of filter_top_sentiment_score to this node  

7. **Create Code node "create_positions_to_close_and_positions_two_open"**  
   - Paste JavaScript that:  
     - Compares open positions and top sentiment stocks  
     - Creates two lists: positions to close and positions to open with allocated market value  
   - Connect output of Alpaca_get_open_positions to this node  

8. **Create Split Out node "positions_to_close"**  
   - Split on field `positions_not_in_top_sentiment`  
   - Connect first output of create_positions_to_close_and_positions_two_open to this node  

9. **Create HTTP Request node "Alpaca-post-order-sell"**  
   - Method: DELETE  
   - URL: `https://paper-api.alpaca.markets/v2/positions/{{$json.asset_id}}`  
   - Authentication: Alpaca HTTP Custom Auth  
   - Connect output of positions_to_close to this node  

10. **Create Split Out node "positions_to_open"**  
    - Split on field `symbols_not_in_open_positions`  
    - Connect second output of create_positions_to_close_and_positions_two_open to this node  

11. **Create Wait node "Wait"**  
    - Duration: 2 minutes  
    - Connect output of positions_to_open to this node  

12. **Create HTTP Request node "Alpaca-post-order-buy"**  
    - Method: POST  
    - URL: `https://paper-api.alpaca.markets/v2/orders`  
    - Body Parameters (JSON):  
      - symbol: `={{ $json.symbol }}`  
      - notional: `={{ $json.market_value_per_symbol.toFixed(2) }}`  
      - side: "buy"  
      - type: "market"  
      - time_in_force: "day"  
    - Authentication: Alpaca HTTP Custom Auth  
    - Connect output of Wait to this node  

13. **Create Merge node "merge_orders_to_write_to_sheets"**  
    - Default merge mode (append)  
    - Connect output of Alpaca-post-order-sell (main 0) and Alpaca-post-order-buy (main 1) to this node  

14. **Create Google Sheets node "Google Sheets" (positions logging)**  
    - Operation: Append  
    - Document ID: Same Google Sheet as earlier  
    - Sheet Name: "positions" (gid=486231870)  
    - Columns mapped:  
      - date: `={{$today}}`  
      - symbol: `={{$json.symbol}}`  
      - order: `={{$json.side}}`  
      - value: `={{ $if( $json.side == "buy", $json.notional, $('positions_to_close').item.json.market_value) }}`  
    - Credentials: Google Sheets OAuth2  
    - Connect output of merge_orders_to_write_to_sheets to this node  

15. **Set up all credentials**  
    - Alpaca API HTTP Custom Auth and HTTP Query Auth with valid API key and secret  
    - Google Sheets OAuth2 with access to relevant sheets  

16. **Set workflow timezone**  
    - Configure workflow settings timezone to Asia/Jerusalem  

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                                     |
|------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| This workflow relies on daily sentiment scores generated by a separate "Sentiment Analysis Bot" workflow.              | Sentiment Bot Template: https://n8n.io/workflows/5369-automated-stock-sentiment-analysis-with-google-gemini-and-eodhd-news-api/    |
| Alpaca paper trading account is used for safe testing without real money exposure.                                      | https://alpaca.markets/docs/api-documentation/api-v2/                                                                             |
| Timezone is set to Asia/Jerusalem to align with the target user’s local schedule; adjust if needed.                     |                                                                                                                                    |
| The workflow manages order execution timing cautiously: a 2-minute wait between sell and buy to ensure fund availability.|                                                                                                                                    |
| The Google Sheet document used for data storage must have sheets named exactly "sentiments", "balance", and "positions".|                                                                                                                                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.