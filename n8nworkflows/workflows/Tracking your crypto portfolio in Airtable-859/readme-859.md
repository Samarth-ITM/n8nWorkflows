Tracking your crypto portfolio in Airtable

https://n8nworkflows.xyz/workflows/tracking-your-crypto-portfolio-in-airtable-859


# Tracking your crypto portfolio in Airtable

### 1. Workflow Overview

This workflow automates the hourly tracking and updating of a cryptocurrency investment portfolio stored in Airtable. It fetches current market prices for portfolio coins from CoinGecko, updates individual coin prices in Airtable, calculates the total portfolio value, and appends this summary to a separate Airtable table for historical tracking.

Logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow execution every hour.
- **1.2 Portfolio Retrieval:** Retrieves the list of cryptocurrency symbols from the Airtable "Portfolio" table.
- **1.3 Market Data Fetching:** For each coin symbol, queries CoinGecko to get the current USD price.
- **1.4 Price Update:** Updates the portfolio entries in Airtable with the latest coin prices.
- **1.5 Portfolio Valuation:** Retrieves updated portfolio values from Airtable, sums them, and appends the total portfolio value to an Airtable table "Portfolio Value."

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** This block triggers the entire workflow execution once every hour, ensuring portfolio values are refreshed regularly.
- **Nodes Involved:** 
  - Run Top of Hour

- **Node Details:**

  - **Run Top of Hour**
    - Type: Cron Trigger
    - Role: Initiates workflow on an hourly schedule.
    - Configuration: Trigger set to "everyHour" mode.
    - Inputs: None (trigger node).
    - Outputs: Connects to "Get Portfolio" node.
    - Edge Cases: Misconfiguration of cron schedule can cause workflow not to trigger; cron node downtime may disrupt updates.

#### 1.2 Portfolio Retrieval

- **Overview:** Retrieves the list of cryptocurrency symbols currently tracked in Airtable from the "Portfolio" table.
- **Nodes Involved:**
  - Get Portfolio

- **Node Details:**

  - **Get Portfolio**
    - Type: Airtable (List operation)
    - Role: Fetches portfolio coin symbols from Airtable.
    - Configuration:
      - Table: "Portfolio"
      - Operation: "list"
      - Fields retrieved: ["Symbol"]
      - Airtable App ID: "appT7eX4iZcZVRIdq" (to be replaced with user’s app ID)
      - Uses stored Airtable API credentials.
    - Inputs: Triggered by "Run Top of Hour"
    - Outputs: Connects to "CoinGecko"
    - Edge Cases: Authentication failure if credentials invalid; API rate limiting; missing or empty "Portfolio" table; missing "Symbol" fields.

#### 1.3 Market Data Fetching

- **Overview:** For each coin retrieved from Airtable, fetches current market data from CoinGecko.
- **Nodes Involved:**
  - CoinGecko
  - Set

- **Node Details:**

  - **CoinGecko**
    - Type: CoinGecko API node
    - Role: Retrieves market data for each coin symbol.
    - Configuration:
      - Operation: "get"
      - Coin Id: Expression `={{$json["fields"]["Symbol"]}}` — uses the Symbol field from Airtable record.
      - Options: Request market_data (true), localization (false).
    - Inputs: Receives portfolio entries from "Get Portfolio"
    - Outputs: Connects to "Set"
    - Edge Cases: Invalid coin ID causing API error; CoinGecko API outages or rate limits; expression failure if "Symbol" field missing.

  - **Set**
    - Type: Set node (data transformation)
    - Role: Extracts and formats the current price and record ID for updating Airtable.
    - Configuration:
      - Sets two string fields:
        - "Present Price" = current USD price from CoinGecko response (`{{$json["market_data"]["current_price"]["usd"]}}`)
        - "Id" = Airtable record ID from previous "Get Portfolio" node (`{{$node["Get Portfolio"].json["id"]}}`)
      - Keeps only these two fields.
    - Inputs: From "CoinGecko"
    - Outputs: Connects to "Update Values"
    - Edge Cases: Null or missing price in API response; expression errors if nodes are disconnected or data missing.
  
#### 1.4 Price Update

- **Overview:** Updates the "Portfolio" Airtable table, setting each coin’s current price.
- **Nodes Involved:**
  - Update Values

- **Node Details:**

  - **Update Values**
    - Type: Airtable (Update operation)
    - Role: Updates the "Present Price" field of portfolio entries in Airtable.
    - Configuration:
      - Table: "Portfolio"
      - Operation: "update"
      - Record ID: Expression `={{$node["SplitInBatches"].json["id"]}}` (Note: The workflow JSON does not include a "SplitInBatches" node, but the expression references it. This appears to be an inconsistency or leftover from prior version. The correct expression should likely be `={{$json["Id"]}}` from the "Set" node output.)
      - Fields updated: ["Present Price"]
      - Update All Fields: false
      - Airtable App ID: same as above
      - Credentials: Airtable API
    - Inputs: From "Set"
    - Outputs: Connects to "Get Portfolio Values"
    - Edge Cases: Record ID missing or invalid; API errors; rate limiting; partial updates.

#### 1.5 Portfolio Valuation

- **Overview:** Aggregates the current portfolio values and appends the total to a dedicated Airtable table for tracking portfolio value over time.
- **Nodes Involved:**
  - Get Portfolio Values
  - Determine Total Value
  - Append Portfolio Value

- **Node Details:**

  - **Get Portfolio Values**
    - Type: Airtable (List operation)
    - Role: Retrieves the "Present Value" field from the "Portfolio" table for all coins.
    - Configuration:
      - Table: "Portfolio"
      - Operation: "list"
      - Fields: ["Present Value"]
      - Airtable App ID and credentials as before.
    - Inputs: From "Update Values"
    - Outputs: Connects to "Determine Total Value"
    - Edge Cases: Missing or null present values; empty table; API errors.

  - **Determine Total Value**
    - Type: Function
    - Role: Sums the "Present Value" fields retrieved from Airtable to compute total portfolio value.
    - Configuration:
      - Custom JavaScript code:
        ```javascript
        var totalValues = 0;
        items.forEach(sumValues);

        function sumValues(value, index, array) {
          totalValues = totalValues + value.json.fields['Present Value'];
        }

        items = [{"json": {}}];
        items[0].json['Portfolio Value (US$)'] = totalValues;

        return items;
        ```
    - Inputs: From "Get Portfolio Values"
    - Outputs: Connects to "Append Portfolio Value"
    - Edge Cases: Non-numeric or missing "Present Value" fields; empty inputs causing total to be zero; JavaScript errors.

  - **Append Portfolio Value**
    - Type: Airtable (Append operation)
    - Role: Inserts a new record into "Portfolio Value" table with the total portfolio value.
    - Configuration:
      - Table: "Portfolio Value"
      - Operation: "append"
      - Fields: ["Portfolio Value (US$)"]
      - Airtable App ID and credentials.
    - Inputs: From "Determine Total Value"
    - Outputs: None (end node)
    - Edge Cases: API errors; credential issues; invalid field names.

---

### 3. Summary Table

| Node Name            | Node Type             | Functional Role                         | Input Node(s)         | Output Node(s)           | Sticky Note                         |
|----------------------|-----------------------|---------------------------------------|-----------------------|--------------------------|-----------------------------------|
| Run Top of Hour       | Cron Trigger          | Triggers workflow hourly               | None                  | Get Portfolio            |                                   |
| Get Portfolio        | Airtable (List)       | Retrieves coin symbols from Airtable   | Run Top of Hour       | CoinGecko                |                                   |
| CoinGecko            | CoinGecko API         | Fetches market data for each coin      | Get Portfolio         | Set                      |                                   |
| Set                  | Set                   | Extracts current price and record ID   | CoinGecko             | Update Values            |                                   |
| Update Values        | Airtable (Update)     | Updates coin prices in Airtable        | Set                   | Get Portfolio Values     | Expression references "SplitInBatches" node missing from workflow; likely error. |
| Get Portfolio Values | Airtable (List)       | Retrieves present values for portfolio  | Update Values         | Determine Total Value    |                                   |
| Determine Total Value | Function              | Sums portfolio values                   | Get Portfolio Values  | Append Portfolio Value   |                                   |
| Append Portfolio Value| Airtable (Append)     | Appends total portfolio value record   | Determine Total Value | None                    |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n named "Update Crypto Values".**

2. **Add a Cron node named "Run Top of Hour":**
   - Set trigger mode to "everyHour".
   - No credentials needed.
   - This node will start the workflow hourly.

3. **Add an Airtable node named "Get Portfolio":**
   - Set operation to "list".
   - Set Table to "Portfolio".
   - Under Additional Options, specify Fields to retrieve: ["Symbol"].
   - Set Application ID to your Airtable base ID (e.g., "appT7eX4iZcZVRIdq").
   - Connect credentials for Airtable API.
   - Connect input from "Run Top of Hour".

4. **Add a CoinGecko node named "CoinGecko":**
   - Set operation to "get".
   - Set Coin Id parameter to expression: `{{$json["fields"]["Symbol"]}}`.
   - Under Options, enable "market_data" (true) and disable "localization" (false).
   - Connect input from "Get Portfolio".

5. **Add a Set node named "Set":**
   - Configure to set two string fields:
     - "Present Price": Expression `{{$json["market_data"]["current_price"]["usd"]}}`
     - "Id": Expression `{{$node["Get Portfolio"].json["id"]}}`
   - Enable "Keep Only Set" to remove other fields.
   - Connect input from "CoinGecko".

6. **Add an Airtable node named "Update Values":**
   - Set operation to "update".
   - Set Table to "Portfolio".
   - For Record ID, use expression: `{{$json["Id"]}}` (Note: adjust from original workflow expression referencing missing node).
   - Under Fields, set to update: ["Present Price"].
   - Set Application ID and Airtable API credentials as before.
   - Connect input from "Set".

7. **Add an Airtable node named "Get Portfolio Values":**
   - Set operation to "list".
   - Set Table to "Portfolio".
   - Under Additional Options, specify Fields: ["Present Value"].
   - Application ID and credentials as above.
   - Connect input from "Update Values".

8. **Add a Function node named "Determine Total Value":**
   - Paste the following code:
     ```javascript
     var totalValues = 0;

     items.forEach(sumValues);

     function sumValues(value) {
       totalValues += parseFloat(value.json.fields['Present Value']) || 0;
     }

     items = [{"json": {}}];

     items[0].json['Portfolio Value (US$)'] = totalValues;

     return items;
     ```
   - Connect input from "Get Portfolio Values".

9. **Add an Airtable node named "Append Portfolio Value":**
   - Set operation to "append".
   - Set Table to "Portfolio Value".
   - Under Fields, set: ["Portfolio Value (US$)"].
   - Use same Application ID and credentials.
   - Connect input from "Determine Total Value".

10. **Save and activate the workflow.**

**Notes:**
- Replace all Airtable Application IDs and credentials with your own.
- Ensure that the Airtable tables "Portfolio" and "Portfolio Value" exist with the correct fields.
- Fix the expression in "Update Values" for Record ID as the original workflow references a missing node "SplitInBatches".
- The workflow assumes all portfolio entries have valid "Symbol" and "Present Value" fields.
- Consider adding error handling or batching if portfolio size is large.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                                |
|-------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| The workflow updates your crypto portfolio value every hour by querying CoinGecko and syncing data to Airtable.  | Workflow purpose summary                                                                                                       |
| You can view or copy the sample Airtable base here: https://airtable.com/shrVzSe6GMNYsZ2Zx                         | Airtable base example and template                                                                                            |
| Image preview of expected Airtable portfolio and value tables is included in original documentation (not in JSON) | Visual aid for understanding data structure                                                                                   |
| Ensure Airtable API credentials have read and write access for the relevant bases and tables.                      | Credential management best practice                                                                                           |
| CoinGecko API limits requests; large portfolios may require batching or rate limiting strategies.                  | Potential integration limitation                                                                                              |
| The workflow requires n8n version supporting CoinGecko node (v1 or later) and Airtable node with API v1 support.  | Version compatibility                                                                                                         |