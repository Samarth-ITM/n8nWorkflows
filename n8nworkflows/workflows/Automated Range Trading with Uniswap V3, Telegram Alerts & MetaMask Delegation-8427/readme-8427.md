Automated Range Trading with Uniswap V3, Telegram Alerts & MetaMask Delegation

https://n8nworkflows.xyz/workflows/automated-range-trading-with-uniswap-v3--telegram-alerts---metamask-delegation-8427


# Automated Range Trading with Uniswap V3, Telegram Alerts & MetaMask Delegation

### 1. Workflow Overview

This workflow automates range trading on the Uniswap V3 decentralized exchange, integrating price monitoring via TWAP (Time Weighted Average Price), executing trades within predefined price ranges, sending Telegram notifications for confirmations and status updates, and managing wallet delegation through MetaMask using the 1Shot API. The workflow targets users who want to automate DCA-style (Dollar-Cost Averaging) ETH/USDC trades based on price thresholds while maintaining control over wallet delegation and receiving trade alerts via Telegram.

**Logical Blocks:**

- **1.1 Configuration & Initialization:** Setting up trading parameters and wallet delegation details.
- **1.2 TWAP Calculation:** Fetching on-chain pool observations and computing the TWAP price.
- **1.3 Trade Decision Logic:** Determining whether to buy, sell, or hold based on TWAP vs configured thresholds.
- **1.4 Trade Confirmation via Telegram:** Sending confirmation requests to the user and handling responses.
- **1.5 Quote Retrieval:** Obtaining trade quotes from the Uniswap Quoter contract for buy or sell.
- **1.6 Approval & Execution:** Granting token approvals to the Uniswap router, executing trades, and handling success or failure notifications.
- **1.7 Wallet & Contract Method Setup:** Managing wallet creation and ensuring required contract methods are available in the 1Shot API environment.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration & Initialization

- **Overview:** This block defines all trading parameters (amount, price ranges, tokens, fees), sets the delegated wallet address, and Telegram chat ID for notifications. It initializes data flow with these constants.

- **Nodes Involved:**  
  - `Swap Configs`  
  - `Schedule Trigger`  
  - `Sticky Note1` (documentation)  
  - `Sticky Note2` (documentation)  

- **Node Details:**

  - **Swap Configs**  
    - Type: Code node  
    - Role: Defines all swap parameters such as amount to trade, price boundaries, token addresses, decimals, fees, slippage, and Telegram chat ID.  
    - Key parameters:  
      - `amountDCA`: 200,000 (units depend on token decimals, here USDC with 6 decimals)  
      - `upperPrice` & `lowerPrice`: 4600 and 4500 USD (ETH price bounds)  
      - `delegator`: Ethereum wallet address delegated to execute trades  
      - Token addresses and decimals for WETH and USDC  
      - Uniswap SwapRouterV2 address  
      - Fee tier (500 corresponds to 0.05%)  
      - Slippage tolerance (2.5%)  
      - TWAP window length in seconds (120)  
    - Outputs all these parameters as JSON for downstream nodes.  
    - Input: Triggered by `Schedule Trigger`.  
    - Output: Connects to `Fetch Pool TWA Observations`.  
    - Edge cases: Misconfigured parameters could cause contract call failures or incorrect trade logic; decimals mismatch may cause price calculation errors.  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow to execute periodically (every minute).  
    - Input: None (starting point)  
    - Output: To `Swap Configs`.  

  - **Sticky Notes (1 & 2)**  
    - Type: Documentation nodes  
    - Function: Provide user instructions on configuration and 1Shot API setup.  
    - No connections, purely for reference.  

#### 1.2 TWAP Calculation

- **Overview:** Fetches Uniswap pool observations to compute the TWAP and related metrics based on on-chain data.

- **Nodes Involved:**  
  - `Fetch Pool TWA Observations`  
  - `Calculate TWAP`  

- **Node Details:**

  - **Fetch Pool TWA Observations**  
    - Type: 1Shot API one-shot node  
    - Role: Reads the cumulative tick and liquidity data from the Uniswap pool contract using the `observe` method. The method is called with parameters `[0, secondsAgo]` to get cumulative values at now and secondsAgo ago.  
    - Parameters: Dynamically set `secondsAgos` from config node.  
    - Credential: 1Shot OAuth2 API with appropriate permissions.  
    - Output: Passes raw observation arrays to `Calculate TWAP`.  
    - Edge cases: API or contract call failure, invalid pool address, or network issues.  

  - **Calculate TWAP**  
    - Type: Code node  
    - Role:  
      - Implements Uniswap V3 TWAP calculation logic via JavaScript BigInt math.  
      - Converts cumulative tick differences to average tick.  
      - Converts average tick to sqrt price using Uniswap's formula.  
      - Calculates TWAP price with decimal adjustment between token0 and token1.  
      - Calculates Time-Weighted Average Liquidity (TWAL).  
      - Adds these computed values to the workflow data.  
    - Inputs: Raw observation data and swap config decimals.  
    - Outputs: JSON with `twap`, `sqrtTWAPriceX96`, and `twal`.  
    - Edge cases: Tick out of bounds, BigInt operation errors, malformed input data.  

#### 1.3 Trade Decision Logic

- **Overview:** Determines whether the current TWAP price indicates the need to buy, sell, or hold.

- **Nodes Involved:**  
  - `Get Last Trade Type`  
  - `Parse Last Trade`  
  - `Buy, Sell or Hold`  
  - `Don't Repeat Buys`  
  - `Don't Repeat Sells`  

- **Node Details:**

  - **Get Last Trade Type**  
    - Type: 1Shot API one-shot node  
    - Role: Retrieves the most recent completed transaction from the 1Shot wallet with memo `rangeTrade` to check last trade direction.  
    - Parameters: Filters on `memo`, `status`, `chainId`, pagination, and time window.  
    - Output: Passes last trade data to parser.  
    - Edge cases: No trades found, API errors.

  - **Parse Last Trade**  
    - Type: Code node  
    - Role: Parses JSON memo from last trade to extract trade metadata like buy/sell and amounts.  
    - Output: Adds parsed memo object to JSON.  
    - Edge cases: Invalid JSON in memo, missing memo field.

  - **Buy, Sell or Hold**  
    - Type: Switch node  
    - Role: Compares current TWAP price to upper and lower thresholds:  
      - If `TWAP > upperPrice`: route to "Sell ETH" branch  
      - If `TWAP < lowerPrice`: route to "Buy ETH" branch  
      - Otherwise, no action (hold).  
    - Inputs: TWAP from `Calculate TWAP` and thresholds from `Swap Configs`.  
    - Output: Directs flow toward buy or sell confirmation nodes.  
    - Edge cases: Non-numeric TWAP or thresholds.

  - **Don't Repeat Buys**  
    - Type: If node  
    - Role: Prevents repeated buy trades if last trade was also a buy or no memo exists.  
    - Conditions: Passes only if last trade was not a buy.  
    - Output: Routes to "Get Buy Quote" if new buy allowed.  
    - Edge cases: Missing memo or unexpected memo fields.

  - **Don't Repeat Sells**  
    - Type: If node  
    - Role: Prevents repeated sell trades if last trade was a sell.  
    - Conditions: Passes only if last trade was not a sell.  
    - Output: Routes to "Get Sell Quote" if new sell allowed.  
    - Edge cases: Same as above.

#### 1.4 Trade Confirmation via Telegram

- **Overview:** Requests user confirmation for buy or sell trades via Telegram, waits for response, and routes accordingly.

- **Nodes Involved:**  
  - `Get Buy Qoute`  
  - `Get Sell Qoute`  
  - `Confirm Buy`  
  - `Confirm Sell`  
  - `Buy or Cancel?`  
  - `Sell or Cancel?`  
  - `Purchase Cancelled`  
  - `Sell Cancelled`  

- **Node Details:**

  - **Get Buy Qoute & Get Sell Qoute**  
    - Type: 1Shot API one-shot simulation nodes  
    - Role: Call QuoterV2 contract's `quoteExactInputSingle` method to get estimated output for buy or sell amounts, considering amountIn and pool fee.  
    - Input params: tokenIn, tokenOut, amountIn, fee, and price limit (0).  
    - Output: Decoded quote result for minimum amountOut.  
    - Edge cases: Contract call failure or invalid parameters.

  - **Confirm Buy / Confirm Sell**  
    - Type: Telegram node (sendAndWait)  
    - Role: Sends confirmation message to Telegram user with details of the proposed trade amount and price, and presents dropdown with Yes/No options.  
    - Waits for user response within a time limit (minutes).  
    - Output: User response triggers subsequent logic.  
    - Edge cases: User timeout, no response, Telegram API issues.

  - **Buy or Cancel? / Sell or Cancel?**  
    - Type: If nodes  
    - Role: Branch based on user confirmation response:  
      - If "yes": proceed with approvals and trade execution.  
      - Else: route to cancellation notification nodes.  
    - Edge cases: Unexpected user inputs.

  - **Purchase Cancelled / Sell Cancelled**  
    - Type: Telegram nodes  
    - Role: Notify user that trade was cancelled.  
    - Edge cases: Telegram delivery failure.

#### 1.5 Approval & Execution

- **Overview:** Grants token approval to the Uniswap SwapRouter contract, executes the trade via 1Shot delegated wallet, and handles success or failure notifications.

- **Nodes Involved:**  
  - `Give Approval to Router (Buy)`  
  - `Give Approval to Router (Sell)`  
  - `Buy ETH`  
  - `Sell ETH`  
  - `Success Details (Buy)`  
  - `Success Details (Sell)`  
  - `Buy Failure Notification`  
  - `Sell Failure Notification`  

- **Node Details:**

  - **Give Approval to Router (Buy / Sell)**  
    - Type: 1Shot API synchronous execution nodes  
    - Role: Executes `approve` calls on USDC (for buy) or WETH (for sell) tokens, permitting the Uniswap SwapRouter to spend specified amounts.  
    - Uses delegated wallet address from config.  
    - Includes memo for logging purpose.  
    - Edge cases: Approval denial, network failure.

  - **Buy ETH / Sell ETH**  
    - Type: 1Shot API synchronous execution nodes  
    - Role: Executes the `exactInputSingle` swap on Uniswap SwapRouterV2 via delegated wallet.  
    - Parameters include tokens, fee, recipient, amountIn, minimum amountOut (with slippage adjustment), and price limits.  
    - Includes memo with trade details (amounts, TWAP).  
    - Edge cases: Insufficient balance, slippage exceeded, transaction failure.

  - **Success Details (Buy / Sell)**  
    - Type: Telegram notification nodes  
    - Role: Sends detailed success messages to Telegram with amounts swapped, TWAP value, transaction hash, and remaining token balance.  
    - Edge cases: Telegram API failure.

  - **Buy Failure Notification / Sell Failure Notification**  
    - Type: Telegram notification nodes  
    - Role: Notify user if trade execution failed.  
    - Edge cases: Telegram API failure.

#### 1.6 Wallet & Contract Method Setup

- **Overview:** Manages the setup of the 1Shot wallet environment, ensures required contracts and methods are present, and creates wallet if missing.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (manual trigger)  
  - `List wallets`  
  - `If` (wallet exists check)  
  - `Create wallet`  
  - `Get Wallet ID`  
  - `Assure WETH Methods`  
  - `Assure USDC Methods`  
  - `Assure QuoterV2 Methods`  
  - `Assure SwapRouter02 Methods`  
  - `Assure ETH/USDC Pool Methods`  
  - `Sticky Note4` (documentation)  
  - `Sticky Note5` (YouTube tutorial link)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual trigger  
    - Role: Entry point for setup phase.  

  - **List wallets**  
    - Type: 1Shot API node  
    - Role: Retrieves existing wallets on chain 42161 (Arbitrum).  

  - **If**  
    - Type: If node  
    - Role: Checks if a wallet ID exists; branches to create new wallet or proceed.  

  - **Create wallet**  
    - Type: 1Shot API create wallet node  
    - Role: Creates a new wallet named with current date and description "Range Trading Wallet".  

  - **Get Wallet ID**  
    - Type: Code node  
    - Role: Extracts wallet ID from wallet creation or listing result.  

  - **Assure Methods Nodes**  
    - Type: 1Shot API nodes  
    - Role: For each relevant contract (WETH, USDC, QuoterV2, SwapRouterV2, ETH/USDC Pool), ensures all required smart contract methods are imported and ready for use in 1Shot.  
    - Parameters specify chain ID, contract address, and prompt ID referencing method sets.  
    - Edge cases: API or network errors, missing contract ABI.  

  - **Sticky Note4 & Sticky Note5**  
    - Documentation notes guiding user through 1Shot API wallet setup and linking to a YouTube tutorial.  

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                                  | Input Node(s)                           | Output Node(s)                          | Sticky Note                                           |
|------------------------------|---------------------------|-------------------------------------------------|---------------------------------------|---------------------------------------|-------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger          | Periodic workflow start                          | -                                     | Swap Configs                          |                                                       |
| Swap Configs                | Code                      | Define trading parameters and addresses         | Schedule Trigger                      | Fetch Pool TWA Observations           | See Sticky Note1 for swap config instructions         |
| Fetch Pool TWA Observations | 1Shot One-shot            | Fetch pool observation data for TWAP calculation| Swap Configs                         | Calculate TWAP                        | See Sticky Note2 for 1Shot API connection details     |
| Calculate TWAP              | Code                      | Calculate TWAP price and liquidity               | Fetch Pool TWA Observations           | Get Last Trade Type                   |                                                       |
| Get Last Trade Type         | 1Shot One-shot            | Retrieve last completed trade data               | Calculate TWAP                       | Parse Last Trade                      |                                                       |
| Parse Last Trade            | Code                      | Parse memo JSON from last trade                   | Get Last Trade Type                  | Buy, Sell or Hold                    |                                                       |
| Buy, Sell or Hold           | Switch                    | Decide trade action based on TWAP and thresholds | Parse Last Trade                     | Don't Repeat Sells, Don't Repeat Buys|                                                       |
| Don't Repeat Buys           | If                        | Prevent duplicate buy trades                       | Buy, Sell or Hold                   | Get Buy Qoute                       |                                                       |
| Don't Repeat Sells          | If                        | Prevent duplicate sell trades                      | Buy, Sell or Hold                   | Get Sell Qoute                     |                                                       |
| Get Buy Qoute              | 1Shot One-shot            | Get estimated buy quote from Quoter contract      | Don't Repeat Buys                   | Confirm Buy                        | See Sticky Note2 for 1Shot API connection details     |
| Get Sell Qoute             | 1Shot One-shot            | Get estimated sell quote from Quoter contract     | Don't Repeat Sells                  | Confirm Sell                      | See Sticky Note2 for 1Shot API connection details     |
| Confirm Buy                | Telegram                  | Prompt user to confirm buy trade                   | Get Buy Qoute                      | Buy or Cancel?                    |                                                       |
| Confirm Sell               | Telegram                  | Prompt user to confirm sell trade                  | Get Sell Qoute                     | Sell or Cancel?                   |                                                       |
| Buy or Cancel?             | If                        | Branch based on user confirmation for buy         | Confirm Buy                       | Give Approval to Router (Buy), Purchase Cancelled |                                                       |
| Sell or Cancel?            | If                        | Branch based on user confirmation for sell        | Confirm Sell                      | Give Approval to Router (Sell), Sell Cancelled |                                                       |
| Give Approval to Router (Buy) | 1Shot One-shot Synch     | Approve USDC token spend for buy                   | Buy or Cancel?                    | Buy ETH                         |                                                       |
| Give Approval to Router (Sell)| 1Shot One-shot Synch     | Approve WETH token spend for sell                  | Sell or Cancel?                   | Sell ETH                        |                                                       |
| Buy ETH                    | 1Shot One-shot Synch      | Execute buy swap transaction                        | Give Approval to Router (Buy)      | Check Remaining Funds, Buy Failure Notification |                                                       |
| Sell ETH                   | 1Shot One-shot Synch      | Execute sell swap transaction                       | Give Approval to Router (Sell)     | Check New Funds, Sell Failure Notification |                                                       |
| Check Remaining Funds      | 1Shot One-shot            | Check balance after buy                             | Buy ETH                          | Success Details (Buy)             |                                                       |
| Check New Funds            | 1Shot One-shot            | Check balance after sell                            | Sell ETH                         | Success Details (Sell)            |                                                       |
| Success Details (Buy)      | Telegram                  | Notify user of successful buy                       | Check Remaining Funds             | -                               |                                                       |
| Success Details (Sell)     | Telegram                  | Notify user of successful sell                      | Check New Funds                  | -                               |                                                       |
| Buy Failure Notification   | Telegram                  | Notify user of failed buy                            | Buy ETH (onError)                | -                               |                                                       |
| Sell Failure Notification  | Telegram                  | Notify user of failed sell                           | Sell ETH (onError)               | -                               |                                                       |
| Purchase Cancelled         | Telegram                  | Notify user buy was cancelled                        | Buy or Cancel? (no)              | -                               |                                                       |
| Sell Cancelled            | Telegram                  | Notify user sell was cancelled                       | Sell or Cancel? (no)             | -                               |                                                       |
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual start for wallet/method setup                | -                               | List wallets                     |                                                       |
| List wallets              | 1Shot One-shot            | Retrieve existing wallets                            | When clicking ‘Execute workflow’| If                             |                                                       |
| If                        | If                        | Check if wallet exists                               | List wallets                    | Create wallet, Get Wallet ID     |                                                       |
| Create wallet             | 1Shot One-shot            | Create new wallet if none exists                     | If (no wallet)                 | Get Wallet ID                   |                                                       |
| Get Wallet ID             | Code                      | Extract wallet ID from created or existing wallet   | Create wallet, If (yes wallet)  | Assure WETH Methods, Assure SwapRouter02 Methods, Assure USDC Methods, Assure ETH/USDC Pool Methods, Assure QuoterV2 Methods |                                                       |
| Assure WETH Methods       | 1Shot One-shot            | Ensure WETH contract methods imported                | Get Wallet ID                  | -                               |                                                       |
| Assure USDC Methods       | 1Shot One-shot            | Ensure USDC contract methods imported                | Get Wallet ID                  | -                               |                                                       |
| Assure QuoterV2 Methods   | 1Shot One-shot            | Ensure QuoterV2 contract methods imported            | Get Wallet ID                  | -                               |                                                       |
| Assure SwapRouter02 Methods| 1Shot One-shot           | Ensure SwapRouter contract methods imported          | Get Wallet ID                  | -                               |                                                       |
| Assure ETH/USDC Pool Methods| 1Shot One-shot          | Ensure pool contract methods imported                 | Get Wallet ID                  | -                               |                                                       |
| Sticky Note1              | Sticky Note               | Instructions on swap configs                          | -                             | -                             | See detailed setup instructions for swap configs      |
| Sticky Note2              | Sticky Note               | Instructions on 1Shot API connection                  | -                             | -                             | Details on API connection and methods setup            |
| Sticky Note4              | Sticky Note               | Instructions on wallet and methods setup              | -                             | -                             | Setup wallet and contracts for range trading            |
| Sticky Note5              | Sticky Note               | YouTube tutorial link                                 | -                             | -                             | [YouTube Tutorial](https://youtu.be/Hppd04sM4xE)       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Trigger interval: Every 1 minute  
   - Connect to Swap Configs.

2. **Create a Code Node "Swap Configs"**  
   - Define constants:  
     - `amountDCA`: 200000 (USDC units)  
     - `upperPrice`: 4600  
     - `lowerPrice`: 4500  
     - `delegator`: Your delegated wallet address (e.g., MetaMask address)  
     - `telegramChatId`: Your Telegram chat ID for notifications  
     - `router`: Uniswap V3 SwapRouterV2 address (e.g., 0x68b3...)  
     - `token0`: WETH address  
     - `token1`: USDC address  
     - `token0Decimals`: 18  
     - `token1Decimals`: 6  
     - `fee`: 500 (pool fee)  
     - `slippage`: 0.025 (2.5%)  
     - `secondsAgo`: 120 (TWAP window)  
   - Output all parameters as workflow JSON.

3. **Create a 1Shot API Node "Fetch Pool TWA Observations"**  
   - Operation: Read  
   - Contract method: Uniswap pool’s `observe` method  
   - Params: `secondsAgos` = [0, secondsAgo] (from `Swap Configs`)  
   - Credential: 1Shot OAuth2 API  
   - Connect output to "Calculate TWAP".

4. **Create a Code Node "Calculate TWAP"**  
   - Implement Uniswap V3 TWAP formula with BigInt math  
   - Input: Observations and decimals from previous nodes  
   - Calculate average tick, sqrt price, TWAP price, and TWAL  
   - Output these values in JSON for downstream nodes  
   - Connect to "Get Last Trade Type".

5. **Create a 1Shot API Node "Get Last Trade Type"**  
   - Operation: List transactions  
   - Filters: memo = "rangeTrade", status = Completed, chain ID, limit 1  
   - Output to "Parse Last Trade".

6. **Create a Code Node "Parse Last Trade"**  
   - Parse JSON string in memo field of last trade  
   - Output parsed memo object  
   - Connect to "Buy, Sell or Hold".

7. **Create a Switch Node "Buy, Sell or Hold"**  
   - Rule 1: If TWAP > upperPrice → Output "Sell ETH"  
   - Rule 2: If TWAP < lowerPrice → Output "Buy ETH"  
   - Else: no action  
   - Connect "Sell ETH" to "Don't Repeat Sells", "Buy ETH" to "Don't Repeat Buys".

8. **Create If Nodes "Don't Repeat Buys" and "Don't Repeat Sells"**  
   - "Don't Repeat Buys": Pass only if last trade was not a buy or memo absent  
   - "Don't Repeat Sells": Pass only if last trade was not a sell  
   - Connect "Don't Repeat Buys" to "Get Buy Quote"  
   - Connect "Don't Repeat Sells" to "Get Sell Quote"

9. **Create 1Shot API Nodes "Get Buy Quote" and "Get Sell Quote"**  
   - Operation: Simulate call to QuoterV2 `quoteExactInputSingle`  
   - Input parameters: tokenIn, tokenOut, amountIn, fee, sqrtPriceLimitX96=0  
   - Connect "Get Buy Quote" to "Confirm Buy"  
   - Connect "Get Sell Quote" to "Confirm Sell"

10. **Create Telegram Nodes "Confirm Buy" and "Confirm Sell"**  
    - Send message to Telegram with trade details (amount, price)  
    - Use sendAndWait with dropdown Yes/No for confirmation  
    - Connect "Confirm Buy" to "Buy or Cancel?"  
    - Connect "Confirm Sell" to "Sell or Cancel?"

11. **Create If Nodes "Buy or Cancel?" and "Sell or Cancel?"**  
    - If user confirms ("yes"), connect to approval nodes  
    - Else, connect to cancellation notification nodes

12. **Create Telegram Nodes "Purchase Cancelled" and "Sell Cancelled"**  
    - Notify user on cancellation

13. **Create 1Shot API Nodes "Give Approval to Router (Buy)" and "Give Approval to Router (Sell)"**  
    - For buy: approve USDC token for SwapRouter spending  
    - For sell: approve WETH token for SwapRouter spending  
    - Use delegated wallet from config  
    - On success, connect to "Buy ETH" or "Sell ETH" respectively  
    - On failure, connect to failure notification nodes

14. **Create 1Shot API Nodes "Buy ETH" and "Sell ETH"**  
    - Execute `exactInputSingle` swap on SwapRouterV2  
    - Include slippage-adjusted minimum amountOut  
    - Include memo with trade details  
    - On success, connect to "Check Remaining Funds" (buy) or "Check New Funds" (sell)  
    - On failure, connect to failure notification nodes

15. **Create 1Shot API Nodes "Check Remaining Funds" and "Check New Funds"**  
    - Call `balanceOf` on USDC or WETH for delegator wallet  
    - Connect to respective success notification nodes

16. **Create Telegram Nodes "Success Details (Buy)" and "Success Details (Sell)"**  
    - Notify user with trade success, amounts, TWAP, tx hash, and balances

17. **Create Manual Trigger "When clicking ‘Execute workflow’"**  
    - Connect to "List wallets"

18. **Create 1Shot API Node "List wallets"**  
    - Retrieve wallets on chain (Arbitrum)  
    - Connect to "If" node

19. **Create If Node "If"**  
    - Check if there is an existing wallet ID  
    - If yes: connect to "Get Wallet ID"  
    - If no: connect to "Create wallet"

20. **Create 1Shot API Node "Create wallet"**  
    - Create new wallet named with current date and description "Range Trading Wallet"  
    - Connect to "Get Wallet ID"

21. **Create Code Node "Get Wallet ID"**  
    - Extract wallet ID from wallet creation or listing result  
    - Connect to contract method assurance nodes

22. **Create 1Shot API Nodes for "Assure WETH Methods", "Assure USDC Methods", "Assure QuoterV2 Methods", "Assure SwapRouter02 Methods", "Assure ETH/USDC Pool Methods"**  
    - For each, set chainId, walletId, promptId (method set), and contract address  
    - These import necessary contract methods for trading operations

23. **Add Sticky Notes as documentation**  
    - Include detailed instructions on config, 1Shot API setup, and YouTube tutorial link

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Swap configs require setting amount to spend, price limits, delegator wallet, Telegram chat ID, and token/contract addresses. Adjust these carefully to match your trading strategy and network parameters.                                     | See Sticky Note1                                                                                         |
| Connect to 1Shot API by creating an API key and secret. Important contract methods include pool observations, approval, quoting, swaps, and balance checks. Use the 1Shot API to delegate wallet actions securely.                               | See Sticky Note2                                                                                         |
| To set up the 1Shot wallet and import needed contract methods, run the manual trigger and follow the wallet creation and method assurance nodes. This ensures the wallet can execute delegated trades properly.                                  | See Sticky Note4                                                                                         |
| A detailed YouTube tutorial explaining the entire setup and usage is available: https://youtu.be/Hppd04sM4xE                                                                                                                                   | See Sticky Note5                                                                                         |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.