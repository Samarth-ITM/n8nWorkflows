Monitor Cryptocurrency Trading Activity with CoinGecko and Send Alerts to Discord

https://n8nworkflows.xyz/workflows/monitor-cryptocurrency-trading-activity-with-coingecko-and-send-alerts-to-discord-10400


# Monitor Cryptocurrency Trading Activity with CoinGecko and Send Alerts to Discord

### 1. Workflow Overview

This workflow monitors cryptocurrency trading activity by fetching market data from multiple CoinGecko API pages, analyzing volume activity to detect significant changes, and sending alerts to a Discord channel when criteria are met. It is designed for users who want automated, periodic alerts about unusual trading volumes in the cryptocurrency market.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Data Fetch:** Periodically trigger the workflow every 2 hours and concurrently fetch multiple pages of cryptocurrency market data from the CoinGecko API.
- **1.2 Data Aggregation and Analysis:** Merge fetched data pages, analyze the combined dataset for significant volume activity changes.
- **1.3 Alert Decision and Formatting:** Evaluate if alerts need to be sent based on analysis results, format the alert message for Discord.
- **1.4 Discord Notification:** Post the formatted alert message to a Discord channel via webhook.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Data Fetch

- **Overview:**  
This block initiates the workflow every two hours and concurrently fetches data from multiple CoinGecko API endpoints (pages 1 through 8, non-sequential order: page, 6, 7, 8, 5) to gather cryptocurrency market data.

- **Nodes Involved:**  
  - Every 2 Hours (Schedule Trigger)  
  - Fetch Page  
  - Fetch Page 5  
  - Fetch Page 6  
  - Fetch Page 7  
  - Fetch Page 8  

- **Node Details:**  
  - **Every 2 Hours**  
    - Type: Schedule Trigger  
    - Configuration: Triggers workflow execution every 2 hours.  
    - Inputs: None  
    - Outputs: Triggers all fetch page HTTP Request nodes in parallel.  
    - Edge cases: Missed triggers if n8n instance is down; no parameters needed.

  - **Fetch Page, Fetch Page 5, Fetch Page 6, Fetch Page 7, Fetch Page 8**  
    - Type: HTTP Request  
    - Technical Role: Fetch cryptocurrency market data from CoinGecko API pages.  
    - Configuration: Each node requests a specific page of market data; URL and query parameters configured to CoinGecko’s `/coins/markets` endpoint with pagination.  
    - Inputs: Trigger from "Every 2 Hours" node.  
    - Outputs: JSON data arrays of coins with market info.  
    - Key considerations:  
      - API rate limits may cause request failures or delays.  
      - Network errors or API downtime.  
      - Pagination order is non-sequential, likely optimized for relevant datasets.  
      - No authentication required for CoinGecko public API.

---

#### 1.2 Data Aggregation and Analysis

- **Overview:**  
Aggregates the multiple fetched pages into one dataset and runs custom JavaScript code to analyze trading volume activity to detect coins with significant volume changes.

- **Nodes Involved:**  
  - Merge1  
  - Analyze Volume Activity1  

- **Node Details:**  
  - **Merge1**  
    - Type: Merge  
    - Role: Combine the outputs from all fetch page nodes into a single data stream.  
    - Configuration: Default merge mode (likely “Merge By Position” or “Append” to concatenate arrays).  
    - Inputs: Outputs from all five fetch page nodes.  
    - Outputs: Single merged dataset forwarded to the analysis node.  
    - Failure considerations: Misalignment of data arrays if pages differ greatly in length or structure.

  - **Analyze Volume Activity1**  
    - Type: Code (JavaScript)  
    - Role: Analyze merged data for volume anomalies or significant activity.  
    - Configuration: Custom script that likely filters or flags coins based on volume thresholds or percentage changes.  
    - Inputs: Merged dataset from Merge1.  
    - Outputs: Dataset annotated with alert flags or filtered result set.  
    - Failure modes: Expression errors, data format inconsistencies, edge cases if input data is empty or malformed.

---

#### 1.3 Alert Decision and Formatting

- **Overview:**  
Checks if the analysis produced any alerts and, if so, formats a message to be sent to Discord.

- **Nodes Involved:**  
  - Has Alerts? (If)  
  - Format Discord Message1 (Code)  

- **Node Details:**  
  - **Has Alerts?**  
    - Type: If  
    - Role: Conditional branching to detect if analysis results contain alerts.  
    - Configuration: Checks if the dataset from Analyze Volume Activity1 contains any items indicating alerts.  
    - Inputs: Output from Analyze Volume Activity1.  
    - Outputs: True branch if alerts present, False branch otherwise.  
    - Edge cases: Handling empty datasets or unexpected data structures.

  - **Format Discord Message1**  
    - Type: Code (JavaScript)  
    - Role: Construct a Discord-compatible message payload based on alert data.  
    - Configuration: Custom code formatting alert details into a JSON structure suitable for Discord webhook.  
    - Inputs: Alert data from Has Alerts? (True branch).  
    - Outputs: JSON message payload for Discord.  
    - Failure modes: Formatting errors, missing data fields, escaping issues.

---

#### 1.4 Discord Notification

- **Overview:**  
Sends the formatted alert message to a Discord channel using an HTTP request node configured with a Discord webhook URL.

- **Nodes Involved:**  
  - Send Message Alert to Discord (HTTP Request)  

- **Node Details:**  
  - **Send Message Alert to Discord**  
    - Type: HTTP Request  
    - Role: Post alert message to Discord via webhook.  
    - Configuration:  
      - Method: POST  
      - URL: Discord webhook URL (credential or parameterized for security)  
      - Body: JSON payload from Format Discord Message1  
      - Headers: Content-Type application/json  
    - Inputs: Formatted message from Format Discord Message1.  
    - Outputs: HTTP response from Discord API.  
    - Edge cases:  
      - Invalid webhook URL or revoked webhook.  
      - Network errors or Discord API downtime.  
      - Rate limiting by Discord.  
      - Improperly formatted message causing rejection.

---

### 3. Summary Table

| Node Name                  | Node Type         | Functional Role                        | Input Node(s)                                   | Output Node(s)                         | Sticky Note                         |
|----------------------------|-------------------|-------------------------------------|------------------------------------------------|--------------------------------------|-----------------------------------|
| Every 2 Hours              | Schedule Trigger  | Periodic trigger every 2 hours       | None                                           | Fetch Page, Fetch Page 6, 7, 8, 5     |                                   |
| Fetch Page                 | HTTP Request      | Fetch CoinGecko page 1 data           | Every 2 Hours                                   | Merge1                               |                                   |
| Fetch Page 5               | HTTP Request      | Fetch CoinGecko page 5 data           | Every 2 Hours                                   | Merge1                               |                                   |
| Fetch Page 6               | HTTP Request      | Fetch CoinGecko page 6 data           | Every 2 Hours                                   | Merge1                               |                                   |
| Fetch Page 7               | HTTP Request      | Fetch CoinGecko page 7 data           | Every 2 Hours                                   | Merge1                               |                                   |
| Fetch Page 8               | HTTP Request      | Fetch CoinGecko page 8 data           | Every 2 Hours                                   | Merge1                               |                                   |
| Merge1                     | Merge             | Aggregate all fetched pages           | Fetch Page, Fetch Page 6, 7, 8, 5               | Analyze Volume Activity1             |                                   |
| Analyze Volume Activity1   | Code              | Analyze aggregated data for volume alerts | Merge1                                      | Has Alerts?                         |                                   |
| Has Alerts?                | If                | Check if alerts exist                  | Analyze Volume Activity1                        | Format Discord Message1 (True branch) |                                   |
| Format Discord Message1    | Code              | Format alert message for Discord      | Has Alerts? (True)                              | Send Message Alert to Discord        |                                   |
| Send Message Alert to Discord | HTTP Request    | Send alert message to Discord channel | Format Discord Message1                         | None                                |                                   |
| Sticky Note(s)             | Sticky Note       | Comments / Notes                      | N/A                                            | N/A                                 | Various (empty content in this workflow) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: "Every 2 Hours"  
   - Type: Schedule Trigger  
   - Set to trigger every 2 hours (e.g., cron expression or interval).

2. **Create HTTP Request Nodes for CoinGecko API Pages**  
   For each page (1, 5, 6, 7, 8):  
   - Name accordingly (e.g., "Fetch Page", "Fetch Page 5", etc.)  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.coingecko.com/api/v3/coins/markets`  
   - Query Parameters (example):  
     - vs_currency=usd  
     - order=volume_desc  
     - per_page=50  
     - page=[number as per node]  
     - sparkline=false  
   - Connect trigger output to each fetch node input.

3. **Create Merge Node**  
   - Name: "Merge1"  
   - Type: Merge  
   - Mode: Append or Merge By Position (concatenate all fetched arrays)  
   - Connect outputs of all Fetch Page nodes to Merge1 inputs.

4. **Create Code Node for Volume Analysis**  
   - Name: "Analyze Volume Activity1"  
   - Type: Code (JavaScript)  
   - Input: Merged data from Merge1.  
   - Logic: Implement volume analysis to detect coins with abnormal volume changes.  
     - Example: Filter coins with volume increase > threshold.  
   - Output: Filtered or annotated list indicating alerts.  
   - Connect Merge1 output to this node input.

5. **Create If Node to Check Alerts**  
   - Name: "Has Alerts?"  
   - Type: If  
   - Condition: Check if input data array length > 0 or contains alert flags.  
   - Connect Analyze Volume Activity1 output to this node input.

6. **Create Code Node to Format Discord Message**  
   - Name: "Format Discord Message1"  
   - Type: Code (JavaScript)  
   - Input: True branch from Has Alerts?  
   - Logic: Format alert data into a JSON payload compatible with Discord webhook message format.  
     - Include coin names, volume changes, links, timestamps.  
   - Connect True output of Has Alerts? to this node.

7. **Create HTTP Request Node to Send Discord Message**  
   - Name: "Send Message Alert to Discord"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Discord webhook URL (stored as credential or environment variable)  
   - Headers: Content-Type: application/json  
   - Body: JSON from Format Discord Message1 node.  
   - Connect output of Format Discord Message1 to this node.

8. **Connect the Workflow**  
   - Output of Every 2 Hours → Inputs of all Fetch Page nodes in parallel.  
   - Outputs of all Fetch Page nodes → Merge1 inputs.  
   - Merge1 output → Analyze Volume Activity1 input.  
   - Analyze Volume Activity1 output → Has Alerts? input.  
   - Has Alerts? True branch → Format Discord Message1 input.  
   - Format Discord Message1 output → Send Message Alert to Discord input.

9. **Credential Setup**  
   - Discord webhook URL should be stored securely in n8n credentials or environment variables.  
   - No authentication needed for CoinGecko API.

10. **Parameter Defaults and Constraints**  
    - Ensure paging parameters in fetch nodes match CoinGecko limits (max 250 per page).  
    - Volume threshold in analysis node should be configurable for tuning sensitivity.  
    - Rate limits of CoinGecko and Discord should be respected to avoid blocking.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                         |
|------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| CoinGecko public API is free and does not require authentication but has rate limits. | https://www.coingecko.com/api/documentations/v3                                                     |
| Discord webhook format requires JSON payload with specific fields (content, embeds). | https://discord.com/developers/docs/resources/webhook#execute-webhook                                 |
| Scheduling frequency (every 2 hours) balances data freshness and API rate limits. | Workflow design choice                                                                                 |

---

**Disclaimer:** The provided text is generated from an n8n automated workflow. It complies with current content policies and contains no illegal or protected content. All data used is public and legal.