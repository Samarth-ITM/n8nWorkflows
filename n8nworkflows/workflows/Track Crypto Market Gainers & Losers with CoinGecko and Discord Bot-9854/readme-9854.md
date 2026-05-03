Track Crypto Market Gainers & Losers with CoinGecko and Discord Bot

https://n8nworkflows.xyz/workflows/track-crypto-market-gainers---losers-with-coingecko-and-discord-bot-9854


# Track Crypto Market Gainers & Losers with CoinGecko and Discord Bot

### 1. Workflow Overview

This n8n workflow is designed to track cryptocurrency market gainers and losers by fetching real-time data from multiple pages of the CoinGecko API and then sending updates as formatted messages to a Discord channel via a bot webhook. It targets cryptocurrency enthusiasts or community managers who want automated, periodic updates about the top-performing and worst-performing cryptocurrencies over the last 24 hours.

The workflow is logically structured into these main blocks:

- **1.1 Trigger Block:** Initiates the workflow manually or on a schedule.
- **1.2 Data Fetching Block:** Retrieves cryptocurrency market data from 7 paginated API endpoints.
- **1.3 Data Merging Block:** Combines the fetched data pages into a single dataset.
- **1.4 Data Processing Block:** Extracts and filters the top 20 gainers and top 20 losers.
- **1.5 Limiting and Formatting Block:** Limits the processed data to the top 20 and formats messages for Discord.
- **1.6 Notification Block:** Sends the formatted gainers and losers updates to Discord via webhook.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger Block

- **Overview:** Starts the workflow either manually via a button or automatically on a schedule.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Schedule Trigger

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow for testing or immediate execution.  
    - Configuration: Default manual trigger, no parameters.  
    - Input: None  
    - Output: Triggers data fetching nodes.  
    - Failure: None expected, but workflow won’t proceed if not manually triggered.  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow at configured intervals (default schedule not specified; assumed configured in node parameters).  
    - Configuration: Default scheduling without parameters shown; likely set to periodic intervals (e.g., hourly).  
    - Input: None  
    - Output: Triggers data fetching nodes.  
    - Failure: Potential misconfiguration can cause no triggering.  

- Both triggers connect in parallel to the data fetching block, allowing either manual or automatic execution.

---

#### 2.2 Data Fetching Block

- **Overview:** Fetches cryptocurrency market data from seven separate API pages of CoinGecko to gather a broad market overview.
- **Nodes Involved:**  
  - Fetch Page 1  
  - Fetch Page 2  
  - Fetch Page 3  
  - Fetch Page 4  
  - Fetch Page 5  
  - Fetch Page 6  
  - Fetch Page 7

- **Node Details:**

  Each "Fetch Page" node:  
  - Type: HTTP Request  
  - Role: Executes an HTTP GET request to CoinGecko API endpoints for different pages of cryptocurrency market data.  
  - Configuration: Each node targets a different page number in the API URL query parameters (e.g., page=1, page=2, etc.). Specific URL and query details are abstracted but assumed consistent with CoinGecko’s API for market listings.  
  - Input: Trigger from manual or schedule node.  
  - Output: JSON array of coin market data entries for that page.  
  - Failure Modes: Network errors, API rate limits, or invalid credentials (if API key used). HTTP errors or empty responses possible.  
  - Expressions: None indicated.  

- All seven nodes run in parallel and output data to the merge node.

---

#### 2.3 Data Merging Block

- **Overview:** Combines the seven separate data arrays into a single unified dataset for further processing.
- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - Type: Merge  
  - Role: Merges multiple inputs (outputs from the 7 fetch nodes) into a single aggregated dataset.  
  - Configuration: Likely set to "Merge by Append" mode, combining all input arrays sequentially.  
  - Input: 7 inputs, one from each Fetch Page node.  
  - Output: Single merged data array of combined cryptocurrency data from all pages.  
  - Failure Modes: Mismatched data structure from any fetch node could cause errors or malformed output.  
  - Expressions: None.  

- Output connects to the Top 20 Gainers and Top 20 Losers nodes.

---

#### 2.4 Data Processing Block

- **Overview:** Processes the merged dataset to identify and isolate the top 20 cryptocurrencies with the highest gains and the top 20 with the largest losses over 24 hours.
- **Nodes Involved:**  
  - Top 20 Gainers  
  - Top 20 Losers

- **Node Details:**

  - Both nodes are of type Code (JavaScript).  
  - Role: Custom code filters and sorts the merged data to select the top gainers and losers, respectively.  
  - Key Logic (interpreted):  
    - Parse the merged dataset.  
    - Sort by 24-hour percentage change descending for gainers, ascending for losers.  
    - Select top 20 entries for each category.  
  - Input: Merged data from the Merge node.  
  - Output: Filtered array of the top 20 gainers or losers.  
  - Failure Modes:  
    - Empty or malformed input data causing sorting or filtering errors.  
    - Unexpected data types or missing fields in the dataset.  
  - Expressions: None external; all logic within code node.  

- Outputs connect respectively to Limit and Limit1 nodes.

---

#### 2.5 Limiting and Formatting Block

- **Overview:** Applies limits to ensure only the top 20 entries are passed forward, then formats the data into textual messages suitable for Discord.
- **Nodes Involved:**  
  - Limit  
  - Limit1  
  - Gainers 24h  
  - Losers 24h

- **Node Details:**

  - **Limit** and **Limit1**  
    - Type: Limit nodes  
    - Role: Enforce a maximum number of items passed downstream (likely set to 20, consistent with top 20).  
    - Input: Top 20 Gainers and Top 20 Losers data respectively.  
    - Output: Limited dataset (max 20 entries).  
    - Failure Modes: None significant; if input less than limit, passes all.  

  - **Gainers 24h** and **Losers 24h**  
    - Type: Code nodes  
    - Role: Format the limited dataset into a Discord message structure, likely including coin name, price change percentage, and other relevant stats.  
    - Input: Limited arrays from Limit nodes.  
    - Output: JSON or string representing the message payload for Discord.  
    - Failure Modes: Improper formatting or missing data fields could cause message errors or empty messages.  

- Outputs connect to respective Discord send nodes.

---

#### 2.6 Notification Block

- **Overview:** Sends the formatted messages about top gainers and losers to a Discord channel using a configured Discord webhook.
- **Nodes Involved:**  
  - Send Gainers Update  
  - Send Losers Update

- **Node Details:**

  - Type: Discord  
  - Role: Posts messages to Discord via webhook.  
  - Configuration: Uses a Discord webhook ID configured in the node. Message content comes from previous formatting nodes.  
  - Input: Formatted messages from Gainers 24h and Losers 24h nodes.  
  - Output: Confirmation of successful message delivery or error details.  
  - Failure Modes:  
    - Invalid or expired webhook URL.  
    - Discord API rate limits.  
    - Network connectivity issues.  
  - Version-specific: Requires n8n Discord node version supporting webhook messaging.  

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                       | Input Node(s)                          | Output Node(s)                | Sticky Note                                                                                               |
|--------------------------|--------------------|------------------------------------|--------------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger    | Manual start trigger                 | None                                 | Fetch Page 1-7                |                                                                                                          |
| Schedule Trigger         | Schedule Trigger   | Scheduled start trigger              | None                                 | Fetch Page 1-7                |                                                                                                          |
| Fetch Page 1             | HTTP Request       | Fetch page 1 of market data          | Manual Trigger, Schedule Trigger     | Merge                        |                                                                                                          |
| Fetch Page 2             | HTTP Request       | Fetch page 2 of market data          | Manual Trigger, Schedule Trigger     | Merge                        |                                                                                                          |
| Fetch Page 3             | HTTP Request       | Fetch page 3 of market data          | Manual Trigger, Schedule Trigger     | Merge                        |                                                                                                          |
| Fetch Page 4             | HTTP Request       | Fetch page 4 of market data          | Manual Trigger, Schedule Trigger     | Merge                        |                                                                                                          |
| Fetch Page 5             | HTTP Request       | Fetch page 5 of market data          | Manual Trigger, Schedule Trigger     | Merge                        |                                                                                                          |
| Fetch Page 6             | HTTP Request       | Fetch page 6 of market data          | Manual Trigger, Schedule Trigger     | Merge                        |                                                                                                          |
| Fetch Page 7             | HTTP Request       | Fetch page 7 of market data          | Manual Trigger, Schedule Trigger     | Merge                        |                                                                                                          |
| Merge                    | Merge              | Combine all pages into one dataset   | Fetch Page 1-7                       | Top 20 Gainers, Top 20 Losers |                                                                                                          |
| Top 20 Gainers           | Code               | Filter and sort top 20 gainers       | Merge                               | Limit                       |                                                                                                          |
| Top 20 Losers            | Code               | Filter and sort top 20 losers        | Merge                               | Limit1                      |                                                                                                          |
| Limit                    | Limit              | Limit to top 20 gainers               | Top 20 Gainers                      | Gainers 24h                 |                                                                                                          |
| Limit1                   | Limit              | Limit to top 20 losers                | Top 20 Losers                      | Losers 24h                  |                                                                                                          |
| Gainers 24h              | Code               | Format gainers data for Discord      | Limit                              | Send Gainers Update          |                                                                                                          |
| Losers 24h               | Code               | Format losers data for Discord       | Limit1                             | Send Losers Update           |                                                                                                          |
| Send Gainers Update      | Discord            | Send gainers update message           | Gainers 24h                        | None                        |                                                                                                          |
| Send Losers Update       | Discord            | Send losers update message            | Losers 24h                        | None                        |                                                                                                          |
| Overview Note            | Sticky Note        | General overview or comment           | None                               | None                        |                                                                                                          |
| Merge Note               | Sticky Note        | Comment related to merging            | None                               | None                        |                                                                                                          |
| Limit Note               | Sticky Note        | Comment related to limiting nodes    | None                               | None                        |                                                                                                          |
| Send Gainers Note        | Sticky Note        | Comment related to sending gainers   | None                               | None                        |                                                                                                          |
| Send Losers Note         | Sticky Note        | Comment related to sending losers    | None                               | None                        |                                                                                                          |
| Sticky Note4             | Sticky Note        | Additional comment                    | None                               | None                        |                                                                                                          |
| Sticky Note5             | Sticky Note        | Additional comment                    | None                               | None                        |                                                                                                          |
| Sticky Note6             | Sticky Note        | Additional comment                    | None                               | None                        |                                                                                                          |
| Sticky Note7             | Sticky Note        | Additional comment                    | None                               | None                        |                                                                                                          |
| Sticky Note              | Sticky Note        | Additional comment                    | None                               | None                        |                                                                                                          |
| Sticky Note1             | Sticky Note        | Additional comment                    | None                               | None                        |                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’". No configuration needed.  
   - Add a **Schedule Trigger** node named "Schedule Trigger". Configure its schedule to desired intervals (e.g., every hour).

2. **Add Data Fetching HTTP Request Nodes**  
   - Add seven **HTTP Request** nodes named "Fetch Page 1" through "Fetch Page 7".  
   - For each node, configure:  
     - HTTP Method: GET  
     - URL: CoinGecko API endpoint for market data, e.g., `https://api.coingecko.com/api/v3/coins/markets`  
     - Query Parameters:  
       - `vs_currency=usd`  
       - `order=market_cap_desc`  
       - `per_page=250` (or API max per page)  
       - `page=1` through `page=7` respectively  
       - `price_change_percentage=24h`  
     - Ensure no authentication needed as CoinGecko’s public API is usually open, but be mindful of rate limits.

3. **Connect Trigger Nodes to All Fetch Nodes**  
   - Connect both "When clicking ‘Execute workflow’" and "Schedule Trigger" nodes to all seven Fetch Page nodes to allow either trigger to start data fetching.

4. **Add Merge Node**  
   - Add a **Merge** node named "Merge".  
   - Set mode to "Append" to combine all incoming arrays.  
   - Connect each Fetch Page node output to a separate input of the Merge node.

5. **Add Code Nodes for Sorting Top 20**  
   - Add a **Code** node named "Top 20 Gainers".  
   - Configure code to:  
     - Parse merged data input.  
     - Sort descending by 24-hour percentage price change.  
     - Return top 20 entries.  
   - Add a second **Code** node named "Top 20 Losers".  
   - Configure code to:  
     - Parse merged data input.  
     - Sort ascending by 24-hour percentage price change.  
     - Return top 20 entries.

6. **Connect Merge Node to Both Top 20 Nodes**  
   - Connect Merge node to "Top 20 Gainers" and "Top 20 Losers".

7. **Add Limit Nodes**  
   - Add two **Limit** nodes named "Limit" and "Limit1".  
   - Configure each to limit output to 20 items.  
   - Connect "Top 20 Gainers" to "Limit".  
   - Connect "Top 20 Losers" to "Limit1".

8. **Add Formatting Code Nodes**  
   - Add a **Code** node named "Gainers 24h" connected to "Limit".  
   - Configure code to format the top gainers data into a readable Discord message string or JSON payload (including coin name, symbol, price, change percentage).  
   - Add a **Code** node named "Losers 24h" connected to "Limit1".  
   - Configure code similarly for losers.

9. **Add Discord Nodes for Sending Messages**  
   - Add a **Discord** node named "Send Gainers Update".  
   - Set node type to "Send Webhook Message".  
   - Configure with your Discord webhook URL or webhook ID.  
   - Connect "Gainers 24h" to this node.  
   - Add another **Discord** node named "Send Losers Update" similarly.  
   - Connect "Losers 24h" to this node.

10. **Test Workflow**  
    - Use the manual trigger node to execute and verify data is fetched, processed, and sent to Discord correctly.  
    - Debug any formatting or API errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                      | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| CoinGecko API is public but subject to rate limits. For heavy use, consider API key or caching strategies.                                        | https://www.coingecko.com/en/api                     |
| Discord webhook URLs should be kept secure; avoid exposing them publicly to prevent misuse.                                                       | https://discord.com/developers/docs/resources/webhook|
| Formatting the Discord message payload as embeds or markdown improves message clarity and visual appeal.                                         | https://discord.com/developers/docs/resources/channel#message-formatting |
| Testing the workflow manually before scheduling helps catch data format or network errors early.                                                  |                                                     |
| Ensure n8n Discord node credentials or webhook configurations are up to date to avoid authentication failures.                                   |                                                     |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.