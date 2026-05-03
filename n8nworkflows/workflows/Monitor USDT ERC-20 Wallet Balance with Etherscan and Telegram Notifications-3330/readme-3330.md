Monitor USDT ERC-20 Wallet Balance with Etherscan and Telegram Notifications

https://n8nworkflows.xyz/workflows/monitor-usdt-erc-20-wallet-balance-with-etherscan-and-telegram-notifications-3330


# Monitor USDT ERC-20 Wallet Balance with Etherscan and Telegram Notifications

### 1. Workflow Overview

This workflow monitors the USDT ERC-20 token balance of a specified Ethereum wallet address by querying Etherscan’s public API every 5 minutes. It detects any balance changes and sends notifications via Telegram accordingly. The workflow is designed for users who want automated, periodic tracking of their ERC-20 token holdings without requiring complex authentication or manual checks.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow execution every 5 minutes.
- **1.2 User Data Setup:** Defines wallet address, Etherscan API key, and USDT contract address.
- **1.3 Etherscan API Query:** Fetches the current USDT token balance for the wallet.
- **1.4 Balance Change Detection:** Compares the current balance with the previously stored balance.
- **1.5 Conditional Notification:** Sends Telegram messages if the balance changed or remained the same.
- **1.6 Telegram Messaging:** Sends formatted messages to a Telegram chat via a bot.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution automatically every 5 minutes to ensure up-to-date monitoring.

- **Nodes Involved:**  
  - Check Balance Every 5 Minutes

- **Node Details:**

  - **Check Balance Every 5 Minutes**  
    - Type: Cron Trigger  
    - Configuration: Set to trigger every 5 minutes (unit: minutes, value: 5)  
    - Input: None (trigger node)  
    - Output: Triggers the next node (`userData`)  
    - Edge Cases: Cron misconfiguration could cause missed or excessive triggers; ensure n8n instance time is correct.

#### 2.2 User Data Setup

- **Overview:**  
  This block sets essential parameters such as the wallet address, Etherscan API key, and the USDT ERC-20 contract address, which are used downstream.

- **Nodes Involved:**  
  - userData

- **Node Details:**

  - **userData**  
    - Type: Set Node  
    - Configuration:  
      - "Your Wallet Address": User must input their Ethereum wallet address.  
      - "Your Etherscan Api Key": User’s Etherscan API key (optional but recommended for higher rate limits).  
      - "USDT ERC-20 Token Address": Defaulted to USDT contract address `0xdAC17F958D2ee523a2206206994597C13D831ec7`, but can be changed to monitor other tokens.  
    - Input: Trigger from Cron node  
    - Output: Passes these parameters as JSON to the next node  
    - Edge Cases: Missing or incorrect wallet address or API key will cause API request failures.

#### 2.3 Etherscan API Query

- **Overview:**  
  Queries Etherscan’s API to retrieve the current USDT token balance of the specified wallet address.

- **Nodes Involved:**  
  - Fetch USDT Balance from Etherscan

- **Node Details:**

  - **Fetch USDT Balance from Etherscan**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.etherscan.io/api`  
      - Query Parameters:  
        - module=account  
        - action=tokenbalance  
        - address= (from `userData` node’s wallet address)  
        - tag=latest  
        - apikey= (from `userData` node’s Etherscan API key)  
        - contractaddress= (from `userData` node’s USDT token address)  
    - Input: Receives wallet and API key data from `userData`  
    - Output: JSON response containing the token balance in the `result` field  
    - Edge Cases:  
      - API rate limits or downtime may cause request failures or empty responses.  
      - Incorrect wallet or contract address will return zero or error.  
      - No authentication required but API key recommended for higher limits.

#### 2.4 Balance Change Detection

- **Overview:**  
  Compares the newly fetched balance with the previously stored balance to detect any changes.

- **Nodes Involved:**  
  - balanceChecker

- **Node Details:**

  - **balanceChecker**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Uses workflow static data to store `previousBalance` persistently across executions.  
      - Extracts `currentBalance` from the Etherscan API response.  
      - Compares `previousBalance` and `currentBalance`.  
      - Sets `balanceChanged` boolean flag accordingly.  
      - Updates static data with the new balance for next comparison.  
      - Passes `balanceChanged`, `previousBalance`, `currentBalance`, and `walletAddress` downstream.  
    - Input: Receives Etherscan API response JSON  
    - Output: JSON object with balance change status and values  
    - Edge Cases:  
      - On first run, no previous balance exists, so it initializes it.  
      - If static data is cleared or workflow reset, balance comparison resets.  
      - Parsing errors if API response format changes.

#### 2.5 Conditional Notification

- **Overview:**  
  Routes workflow execution based on whether the balance has changed or not.

- **Nodes Involved:**  
  - Balance Changed?

- **Node Details:**

  - **Balance Changed?**  
    - Type: If Node  
    - Configuration:  
      - Condition: Checks if `balanceChanged` is true  
    - Input: Receives JSON from `balanceChecker`  
    - Output:  
      - True branch: proceeds to `Balance Changed.` node  
      - False branch: proceeds to `Balance Not Changed.` node  
    - Edge Cases: Expression evaluation errors if `balanceChanged` is missing or malformed.

#### 2.6 Telegram Messaging

- **Overview:**  
  Sends formatted Telegram messages notifying the user about balance changes or stability.

- **Nodes Involved:**  
  - Balance Changed.  
  - Balance Not Changed.

- **Node Details:**

  - **Balance Changed.**  
    - Type: Telegram  
    - Configuration:  
      - Text: Markdown formatted message showing:  
        - Alert emoji and title "USDT Balance Change!"  
        - Wallet address  
        - Previous balance (converted from raw value by dividing by 1e6)  
        - New balance (converted similarly)  
      - Chat ID: User’s Telegram chat ID (must be configured)  
      - Parse Mode: Markdown  
      - Credentials: Telegram API credentials (bot token)  
    - Input: True branch from `Balance Changed?` node  
    - Output: None (terminal node)  
    - Edge Cases:  
      - Invalid Telegram credentials or chat ID cause message failure.  
      - Telegram API rate limits or downtime.  
      - Markdown formatting errors if variables are missing.

  - **Balance Not Changed.**  
    - Type: Telegram  
    - Configuration:  
      - Text: "Balance Unchanged. USDT balance remained stable."  
      - Chat ID: User’s Telegram chat ID  
      - Parse Mode: Markdown  
    - Input: False branch from `Balance Changed?` node  
    - Output: None (terminal node)  
    - Edge Cases: Same as above for Telegram node.

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                      | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                          |
|--------------------------------|--------------------|------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Check Balance Every 5 Minutes   | Cron Trigger       | Initiates workflow every 5 minutes | None                          | userData                      |                                                                                                    |
| userData                       | Set                | Defines wallet, API key, token addr| Check Balance Every 5 Minutes | Fetch USDT Balance from Etherscan |                                                                                                    |
| Fetch USDT Balance from Etherscan | HTTP Request      | Queries Etherscan for token balance| userData                      | balanceChecker                |                                                                                                    |
| balanceChecker                 | Code (JavaScript)  | Detects balance changes             | Fetch USDT Balance from Etherscan | Balance Changed?              |                                                                                                    |
| Balance Changed?               | If                 | Routes based on balance change      | balanceChecker                | Balance Changed., Balance Not Changed. |                                                                                                    |
| Balance Changed.               | Telegram            | Sends Telegram alert on balance change | Balance Changed? (true branch) | None                         |                                                                                                    |
| Balance Not Changed.           | Telegram            | Sends Telegram message if no change | Balance Changed? (false branch) | None                         |                                                                                                    |
| Sticky Note                   | Sticky Note         | Describes workflow purpose          | None                          | None                          | ## USDT ERC-20 Wallet Balance Tracker\n**This workflow** Is a basic concept of integrating your ERC-20 wallet with n8n nodes. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Name: `Check Balance Every 5 Minutes`  
   - Type: Cron Trigger  
   - Set to trigger every 5 minutes (unit: minutes, value: 5).

2. **Create Set Node for User Data**  
   - Name: `userData`  
   - Type: Set  
   - Add three string fields:  
     - `Your Wallet Address`: Paste your Ethereum wallet address here.  
     - `Your Etherscan Api Key`: Paste your Etherscan API key (optional but recommended).  
     - `USDT ERC-20 Token Address`: Default to `0xdAC17F958D2ee523a2206206994597C13D831ec7` or your token contract address.  
   - Connect output of `Check Balance Every 5 Minutes` to this node.

3. **Create HTTP Request Node to Query Etherscan**  
   - Name: `Fetch USDT Balance from Etherscan`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.etherscan.io/api`  
   - Query Parameters:  
     - `module`: `account`  
     - `action`: `tokenbalance`  
     - `address`: Expression: `{{$json["Your Wallet Address"]}}`  
     - `tag`: `latest`  
     - `apikey`: Expression: `{{$json["Your Etherscan Api Key"]}}`  
     - `contractaddress`: Expression: `{{$json["USDT ERC-20 Token Address"]}}`  
   - Connect output of `userData` to this node.

4. **Create Code Node to Detect Balance Change**  
   - Name: `balanceChecker`  
   - Type: Code (JavaScript)  
   - Paste the following code (adapted for n8n v2):  
     ```javascript
     const staticData = $getWorkflowStaticData('global');
     const currentBalance = items[0].json.result;
     const walletAddress = $('userData').first().json['Your Wallet Address'];
     let previousBalance = staticData.previousBalance;
     if (!previousBalance) {
       staticData.previousBalance = currentBalance;
       previousBalance = currentBalance;
     }
     const balanceChanged = previousBalance !== currentBalance;
     staticData.previousBalance = currentBalance;
     return [{json: {balanceChanged, previousBalance, currentBalance, walletAddress}}];
     ```
   - Connect output of `Fetch USDT Balance from Etherscan` to this node.

5. **Create If Node to Check Balance Change**  
   - Name: `Balance Changed?`  
   - Type: If  
   - Condition: Boolean  
     - Value 1: Expression `{{$json.balanceChanged}}`  
     - Value 2: `true`  
   - Connect output of `balanceChecker` to this node.

6. **Create Telegram Node for Balance Changed Notification**  
   - Name: `Balance Changed.`  
   - Type: Telegram  
   - Configure Telegram credentials with your bot token (see Telegram BotFather instructions).  
   - Chat ID: Your Telegram chat ID.  
   - Text (Markdown):  
     ```
     🚨 *USDT Balance Change!*

     Wallet Address: {{ $json.walletAddress }}

     🔴 Previous Balance: {{parseFloat($json.previousBalance)/1e6}} USDT

     🟢 New Balance: {{parseFloat($json.currentBalance)/1e6}} USDT
     ```  
   - Parse mode: Markdown  
   - Connect True output of `Balance Changed?` node to this node.

7. **Create Telegram Node for No Balance Change Notification**  
   - Name: `Balance Not Changed.`  
   - Type: Telegram  
   - Use same Telegram credentials and chat ID as above.  
   - Text: `Balance Unchanged. USDT balance remained stable.`  
   - Parse mode: Markdown  
   - Connect False output of `Balance Changed?` node to this node.

8. **Add Sticky Note (Optional)**  
   - Add a sticky note describing the workflow purpose for clarity.

9. **Activate and Test**  
   - Save and activate the workflow.  
   - Run manually to verify correct operation.  
   - Adjust Telegram chat ID and bot token as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| To create a Telegram bot, search for "BotFather" in Telegram and use the `/newbot` command.         | Telegram BotFather                                                                                   |
| To get your Telegram Chat ID, search for "Get My ChatID" bot and start a conversation.              | Telegram Get My ChatID Bot                                                                           |
| USDT ERC-20 Token Contract Address: `0xdAC17F958D2ee523a2206206994597C13D831ec7`                     | Default token address for USDT on Ethereum                                                         |
| Etherscan API documentation: https://docs.etherscan.io/api-endpoints/accounts                        | For advanced API usage and rate limit details                                                      |
| Markdown formatting is used in Telegram messages for better readability                             | Telegram Bot API Markdown Guide                                                                     |
| Workflow uses n8n static data to persist previous balance between executions                         | n8n Static Data documentation                                                                        |

---

This documentation provides a complete understanding of the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting.