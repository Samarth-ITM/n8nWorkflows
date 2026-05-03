Send Cryptocurrency Price Threshold Alerts from CoinGecko to Discord

https://n8nworkflows.xyz/workflows/send-cryptocurrency-price-threshold-alerts-from-coingecko-to-discord-4759


# Send Cryptocurrency Price Threshold Alerts from CoinGecko to Discord

---

### 1. Workflow Overview

This workflow is designed to monitor the price of a selected cryptocurrency (defaulted to Bitcoin) using the CoinGecko API and send alerts to a Discord channel when the price crosses predefined low or high thresholds. It supports both manual execution and automated periodic checks via a schedule trigger.

The workflow is logically divided into the following blocks:

- **1.1 Triggering Mechanism:** Supports manual execution and scheduled interval-based execution to initiate the workflow.
- **1.2 Price Retrieval and Threshold Setup:** Fetches the current cryptocurrency price from CoinGecko and sets user-defined low and high price thresholds.
- **1.3 Price Movement Evaluation:** Determines if the current price crosses the defined thresholds and identifies the direction of the movement (above high or below low).
- **1.4 Alert Dispatch:** Sends price alerts to a specified Discord channel based on the movement direction (high or low alerts).

---

### 2. Block-by-Block Analysis

#### 1.1 Triggering Mechanism

- **Overview:** This block initiates the workflow either manually by user command or automatically on a recurring schedule (every minute by default).
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - *Type & Role:* Manual Trigger node; starts the workflow on user command.  
     - *Configuration:* No parameters; triggers workflow execution immediately when pressed.  
     - *Inputs/Outputs:* No inputs; output connects to "CoinGecko" node.  
     - *Edge Cases:* No critical failure modes; user must manually trigger.  
     - *Version:* 1

  2. **Schedule Trigger**  
     - *Type & Role:* Scheduled trigger; runs workflow at a fixed interval.  
     - *Configuration:* Interval set to every 1 minute (field: minutes).  
     - *Inputs/Outputs:* No inputs; output connects to "CoinGecko" node.  
     - *Edge Cases:* Network or system downtime could delay execution; misconfiguration of interval possible.  
     - *Version:* 1.2

---

#### 1.2 Price Retrieval and Threshold Setup

- **Overview:** This block fetches the current price of the specified cryptocurrency from CoinGecko and assigns user-defined low and high price thresholds for comparison.
- **Nodes Involved:**  
  - CoinGecko  
  - Set Low and High

- **Node Details:**

  1. **CoinGecko**  
     - *Type & Role:* CoinGecko node; retrieves market data for a specified coin.  
     - *Configuration:*  
       - Coin ID set to "bitcoin" by default.  
       - Operation: market data.  
       - Base currency: USD.  
     - *Input/Output:* Inputs from triggers; outputs current price data to "Set Low and High".  
     - *Key Variables:* Accesses `current_price` from API response JSON.  
     - *Version:* 1  
     - *Edge Cases:* API request failures (e.g., rate limits, network errors), invalid coin IDs, or API downtime could cause errors.

  2. **Set Low and High**  
     - *Type & Role:* Set node; assigns numeric threshold values for price alerts.  
     - *Configuration:*  
       - Assigns fixed numeric values:  
         - `high` = 100,500  
         - `low` = 100,000  
       - These are user-customizable price limits.  
     - *Input/Output:* Receives current price data from CoinGecko; outputs with thresholds for evaluation node.  
     - *Version:* 3.4  
     - *Edge Cases:* User must update thresholds to meaningful values; otherwise, alerts may never trigger.

---

#### 1.3 Price Movement Evaluation

- **Overview:** This logic block checks if the current price has crossed either threshold and determines if the price crossed the high or the low boundary.
- **Nodes Involved:**  
  - Check movement (If node)  
  - Check Direction (If node)

- **Node Details:**

  1. **Check movement**  
     - *Type & Role:* If node; evaluates if current price is greater than high threshold or less than low threshold.  
     - *Configuration:*  
       - Condition uses logical OR between two numeric comparisons:  
         - `current_price > high` OR `current_price < low`  
       - Uses expressions:  
         - `={{ $('CoinGecko').item.json.current_price }}` for current price  
         - `={{ $json.high }}` and `={{ $json.low }}` for thresholds  
     - *Input/Output:* Input from "Set Low and High"; output branches to "Check Direction" if true; no output if false (no alert).  
     - *Version:* 2.2  
     - *Edge Cases:* Expression failures possible if JSON structure changes or data is missing; type mismatches if thresholds are not numeric.

  2. **Check Direction**  
     - *Type & Role:* If node; checks if price crossed above the high threshold.  
     - *Configuration:*  
       - Condition: `current_price > high` (single numeric comparison).  
       - If true, routes to "Message High" node; if false, routes to "Message Low".  
     - *Input/Output:* Input from "Check movement" true branch; outputs to alert nodes.  
     - *Version:* 2.2  
     - *Edge Cases:* Same as "Check movement"; also ensures correct alert routing.

---

#### 1.4 Alert Dispatch

- **Overview:** Sends Discord messages notifying the user when the cryptocurrency price is either above the high threshold or below the low threshold.
- **Nodes Involved:**  
  - Message High  
  - Message Low

- **Node Details:**

  1. **Message High**  
     - *Type & Role:* Discord node; sends a message when price is above the high threshold.  
     - *Configuration:*  
       - Message content template:  
         `💹 {{ $json.name }} is now {{ $json.current_price }}`  
       - Uses Discord Bot API credentials (OAuth2 bot token).  
       - Guild ID and Channel ID must be set by the user (placeholders present).  
     - *Input/Output:* Input from "Check Direction" true branch; no outputs.  
     - *Version:* 2  
     - *Edge Cases:* Authentication errors if credentials are invalid; missing or incorrect guild/channel IDs; Discord API downtime or rate limiting.

  2. **Message Low**  
     - *Type & Role:* Discord node; sends a message when price is below the low threshold.  
     - *Configuration:*  
       - Message content template:  
         `❗〽️ {{ $json.name }} is now {{ $json.current_price }}`  
       - Uses the same Discord Bot API credentials and guild/channel setup as "Message High".  
     - *Input/Output:* Input from "Check Direction" false branch; no outputs.  
     - *Version:* 2  
     - *Edge Cases:* Same as "Message High".

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                                  | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                             |
|---------------------------|---------------------|-------------------------------------------------|--------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Starts workflow manually                         | None                           | CoinGecko                   | Set the trigger time – Run manually or on a schedule.                                                 |
| Schedule Trigger          | Schedule Trigger    | Starts workflow on scheduled intervals           | None                           | CoinGecko                   | Set the trigger time – Run manually or on a schedule.                                                 |
| CoinGecko                 | CoinGecko API       | Fetches current price of selected cryptocurrency | When clicking ‘Execute workflow’, Schedule Trigger | Set Low and High            | Select a coin and define price limits – Fetch current price from CoinGecko – Set your custom low and high price thresholds. |
| Set Low and High          | Set                 | Assigns user-defined price thresholds            | CoinGecko                     | Check movement              | Select a coin and define price limits – Fetch current price from CoinGecko – Set your custom low and high price thresholds. |
| Check movement            | If                  | Checks if current price crossed thresholds       | Set Low and High              | Check Direction             | Check if price crosses defined thresholds – Compare the current price with your limits – Determine if movement is above or below threshold. |
| Check Direction           | If                  | Determines price movement direction               | Check movement                | Message High, Message Low   | Check if price crosses defined thresholds – Compare the current price with your limits – Determine if movement is above or below threshold. |
| Message High              | Discord             | Sends Discord alert for price above high limit   | Check Direction              | None                       | Send alert on Discord – Notify when price is too high or too low – Set your connection and channel.    |
| Message Low               | Discord             | Sends Discord alert for price below low limit    | Check Direction              | None                       | Send alert on Discord – Notify when price is too high or too low – Set your connection and channel.    |
| Sticky Note               | Sticky Note         | Documentation note                                | None                         | None                       | Set the trigger time – Run manually or on a schedule.                                                 |
| Sticky Note1              | Sticky Note         | Documentation note                                | None                         | None                       | Select a coin and define price limits – Fetch current price from CoinGecko – Set your custom low and high price thresholds. |
| Sticky Note2              | Sticky Note         | Documentation note                                | None                         | None                       | Check if price crosses defined thresholds – Compare the current price with your limits – Determine if movement is above or below threshold. |
| Sticky Note3              | Sticky Note         | Documentation note                                | None                         | None                       | Send alert on Discord – Notify when price is too high or too low – Set your connection and channel.    |
| Sticky Note4              | Sticky Note         | Empty/placeholder note                            | None                         | None                       |                                                                                                       |
| Sticky Note5              | Sticky Note         | Empty/placeholder note                            | None                         | None                       |                                                                                                       |
| Sticky Note6              | Sticky Note         | Workflow overview and instructions                | None                         | None                       | This workflow monitors crypto price and sends alerts to Discord on threshold crossings; instructions included. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’" without any parameters.
   - Add a **Schedule Trigger** node named "Schedule Trigger" with the interval set to every 1 minute.

2. **Create CoinGecko Node:**
   - Add a **CoinGecko** node named "CoinGecko".
   - Set the operation to "market".
   - Set "ids" parameter to `bitcoin` (or your target coin).
   - Set "baseCurrency" to `usd`.
   - Connect outputs of both trigger nodes ("When clicking ‘Execute workflow’" and "Schedule Trigger") to the input of "CoinGecko".

3. **Create Threshold Setter Node:**
   - Add a **Set** node named "Set Low and High".
   - Add two number fields:  
     - `high` with value `100500` (modifiable threshold).  
     - `low` with value `100000` (modifiable threshold).
   - Connect "CoinGecko" output to "Set Low and High" input.

4. **Add Price Movement Check (If) Node:**
   - Add an **If** node named "Check movement".
   - Configure conditions as an OR of two numeric comparisons:  
     - Left value: Expression referencing CoinGecko current price: `={{ $('CoinGecko').item.json.current_price }}`  
     - Operation: Greater than (`>`)  
     - Right value: Expression referencing `high`: `={{ $json.high }}`  
     OR  
     - Left value: Same expression for current price  
     - Operation: Less than (`<`)  
     - Right value: Expression referencing `low`: `={{ $json.low }}`
   - Connect "Set Low and High" output to "Check movement" input.

5. **Add Direction Check (If) Node:**
   - Add an **If** node named "Check Direction".
   - Condition:  
     - Left value: Expression for current price `={{ $('CoinGecko').item.json.current_price }}`  
     - Operation: Greater than (`>`)  
     - Right value: Expression for `high` threshold `={{ $json.high }}`.
   - Connect "Check movement" true output to "Check Direction" input.

6. **Add Discord Alert Nodes:**
   - Add a **Discord** node named "Message High".
     - Set resource to "message".
     - Set the message content to: `💹 {{ $json.name }} is now {{ $json.current_price }}`.
     - Configure `guildId` and `channelId` with your Discord server and channel IDs.
     - Set credentials with a valid Discord Bot API credential (OAuth2 bot token).
     - Connect "Check Direction" true output to "Message High".
   - Add a **Discord** node named "Message Low".
     - Similarly, set resource to "message".
     - Set message content to: `❗〽️ {{ $json.name }} is now {{ $json.current_price }}`.
     - Configure `guildId` and `channelId` as above.
     - Use the same Discord Bot API credential.
     - Connect "Check Direction" false output to "Message Low".

7. **Test Workflow:**
   - Run the workflow manually via the "When clicking ‘Execute workflow’" node to verify correctness.
   - Adjust thresholds in "Set Low and High" as required.
   - Ensure Discord credentials are valid and bot has permissions to post messages in the configured channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow monitors the price of a selected cryptocurrency using the CoinGecko API and sends alerts to Discord when thresholds are crossed.              | Workflow purpose and overview                    |
| Edit the "Set Low and High" node to define your target coin and price limits.                                                                                | User customization instructions                  |
| Configure your Discord Webhook URLs or Bot API credentials in the “Message High” and “Message Low” nodes.                                                   | Discord integration setup                         |
| Optionally adjust the schedule trigger to run at your preferred interval.                                                                                   | Scheduling customization                          |
| Enjoy automated crypto alerts 🚀                                                                                                                           | Motivational closing message                      |

---

**Disclaimer:** The text provided is exclusively generated from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.

---