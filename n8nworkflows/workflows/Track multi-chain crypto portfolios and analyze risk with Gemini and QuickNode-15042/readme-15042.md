Track multi-chain crypto portfolios and analyze risk with Gemini and QuickNode

https://n8nworkflows.xyz/workflows/track-multi-chain-crypto-portfolios-and-analyze-risk-with-gemini-and-quicknode-15042


# Track multi-chain crypto portfolios and analyze risk with Gemini and QuickNode

# Workflow Analysis: AI Crypto Portfolio Analyzer

## 1. Workflow Overview
The **AI Crypto Portfolio Analyzer** is a sophisticated automation designed to track multi-chain cryptocurrency portfolios (specifically Ethereum and Polygon), analyze their financial health, and provide AI-driven risk assessments. The workflow aggregates real-time blockchain data, calculates asset allocations, and uses Google Gemini to generate professional investment insights delivered via Slack.

### Logical Blocks:
- **1.1 Trigger and Input Layer:** Initializes the wallet address and the list of coins to be tracked.
- **1.2 Blockchain Data Acquisition:** Fetches balances and gas prices across multiple chains using QuickNode RPCs.
- **1.3 Processing Layer:** Extracts metrics, fetches real-time market prices via API, and calculates total portfolio value and percentages.
- **1.4 AI Analysis:** Uses a Large Language Model (LLM) to analyze the data and generate a structured risk report.
- **1.5 Output Layer:** Formats the AI insights and financial metrics into a human-readable report and sends it to a Slack channel.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger and Input Layer
**Overview:** Sets the starting point of the workflow and defines the target wallet and assets.
- **Nodes Involved:** `When clicking ‘Execute workflow'`, `Set Wallet & Coins`.
- **Node Details:**
    - **When clicking ‘Execute workflow’**: Manual trigger to start the process.
    - **Set Wallet & Coins**: A `Set` node that defines two key variables: `wallet_address` (string) and `coins` (array: `["ETH", "MATIC"]`). These variables are used throughout the workflow to ensure consistency.

### 2.2 Blockchain Data Acquisition
**Overview:** Concurrent fetching of on-chain data from multiple networks.
- **Nodes Involved:** `Ethereum blockchain`, `Polygon blockchain`, `ETH Gas Price`, `Polygon Gas Price`, `Merge Blockchain Data`.
- **Node Details:**
    - **Ethereum blockchain / Polygon blockchain**: QuickNode RPC nodes. They take the `wallet_address` from the input layer to retrieve the current balance on their respective chains.
    - **ETH Gas Price / Polygon Gas Price**: QuickNode RPC nodes configured for the `getGasPrice` operation.
    - **Merge Blockchain Data**: A `Merge` node that aggregates the four separate blockchain data streams into a single array of items for downstream processing.
    - **Potential Failures:** API timeout from QuickNode, invalid wallet address format, or credential expiration.

### 2.3 Processing Layer
**Overview:** Transforms raw blockchain data into financial metrics.
- **Nodes Involved:** `Extract Chain Metrics`, `Fetch Crypto Prices (USD)`, `Merge Price & Portfolio Data`, `Calculate Portfolio Metrics`.
- **Node Details:**
    - **Extract Chain Metrics**: A `Code` node (JavaScript) that iterates through the merged blockchain data to create a clean object containing `eth_balance`, `polygon_balance`, `eth_gas`, `polygon_gas`, and the `coins` list.
    - **Fetch Crypto Prices (USD)**: An `HTTP Request` node calling the CryptoCompare API. It uses a dynamic expression to map the `coins` array into a URL query string (e.g., `fsyms=ETH,MATIC`).
    - **Merge Price & Portfolio Data**: A `Merge` node (Combine by Position) that joins the calculated balances with the live market prices.
    - **Calculate Portfolio Metrics**: A `Code` node that performs the final math: `Balance * Price = Value`. It calculates the total portfolio sum and the percentage allocation for each asset.

### 2.4 AI Analysis
**Overview:** Interprets financial data to provide professional qualitative insights.
- **Nodes Involved:** `Generate AI Portfolio Insights`, `Google Gemini Chat Model`, `Structured Output Parser`.
- **Node Details:**
    - **Generate AI Portfolio Insights**: An AI Agent node. It receives the calculated metrics (balances, prices, gas, total value) and a detailed prompt instructing it to act as a professional crypto analyst.
    - **Google Gemini Chat Model**: The LLM engine providing the reasoning capabilities.
    - **Structured Output Parser**: Ensures the AI returns a strict JSON schema containing: `insight`, `risk`, `suggestion`, `score`, `gas_analysis`, and `allocation_summary`. This prevents the workflow from breaking due to conversational AI "chatter."
    - **Edge Cases:** LLM "hallucinations" regarding risk levels or failure to adhere to the JSON format (mitigated by the parser).

### 2.5 Output Layer
**Overview:** Final delivery of the intelligence report.
- **Nodes Involved:** `Merge Portfolio & AI Data`, `Format Slack Report`, `Send Slack Alert`.
- **Node Details:**
    - **Merge Portfolio & AI Data**: Combines the quantitative metrics (from the Processing Layer) with the qualitative insights (from the AI Layer).
    - **Format Slack Report**: A `Code` node that uses a JavaScript template literal to build a formatted Slack message with emojis, bold text, and clear sections.
    - **Send Slack Alert**: A Slack node that sends the final string to a specific user or channel via OAuth2.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | Manual Trigger | Start workflow | - | Set Wallet & Coins | 🟢 Trigger and Input Layer |
| Set Wallet & Coins | Set | Define wallet/assets | Trigger | Blockchain Nodes | 🟢 Trigger and Input Layer |
| Ethereum blockchain | QuickNode RPC | Get ETH balance | Set Wallet & Coins | Merge Blockchain Data | 🟢 Fetch blockchain + gas data |
| Polygon blockchain | QuickNode RPC | Get MATIC balance | Set Wallet & Coins | Merge Blockchain Data | 🟢 Fetch blockchain + gas data |
| ETH Gas Price | QuickNode RPC | Get ETH gas price | Set Wallet & Coins | Merge Blockchain Data | 🟢 Fetch blockchain + gas data |
| Polygon Gas Price | QuickNode RPC | Get MATIC gas price | Set Wallet & Coins | Merge Blockchain Data | 🟢 Fetch blockchain + gas data |
| Merge Blockchain Data | Merge | Aggregate chain data | Blockchain Nodes | Extract Chain Metrics | 🟢 Fetch blockchain + gas data |
| Extract Chain Metrics | Code | Normalize data | Merge Blockchain Data | Fetch Prices / Merge Price | 🟢 Processing Layer |
| Fetch Crypto Prices (USD) | HTTP Request | Get market prices | Extract Chain Metrics | Merge Price & Port. | 🟢 Processing Layer |
| Merge Price & Portfolio Data | Merge | Combine Price + Bal | Extract/Fetch Prices | Calculate Metrics | 🟢 Processing Layer |
| Calculate Portfolio Metrics | Code | Math & Allocation | Merge Price & Port. | AI Insights / Merge Port. | 🟢 Processing Layer |
| Google Gemini Chat Model | AI Model | Reasoning Engine | - | Generate AI Insights | 🟢 AI Analysis |
| Structured Output Parser | AI Parser | JSON Enforcement | - | Generate AI Insights | 🟢 AI Analysis |
| Generate AI Portfolio Insights | AI Agent | Analyze portfolio | Calculate Metrics | Merge Port. & AI Data | 🟢 AI Analysis |
| Merge Portfolio & AI Data | Merge | Join Quantitative + AI | Calculate/AI Insights | Format Slack Report | - |
| Format Slack Report | Code | Template message | Merge Port. & AI Data | Send Slack Alert | 🟢 Output Layer |
| Send Slack Alert | Slack | Deliver notification | Format Slack Report | - | 🟢 Output Layer |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Input Setup
1. Create a **Manual Trigger** node.
2. Create a **Set** node named "Set Wallet & Coins". Add two fields:
   - `wallet_address` (String): Enter an EVM address.
   - `coins` (Array): Enter `["ETH", "MATIC"]`.

### Step 2: Blockchain Data Integration
1. Add four **QuickNode RPC** nodes.
   - Node 1: Ethereum Mainnet $\rightarrow$ Address = `{{ $json.wallet_address }}`.
   - Node 2: Polygon Mainnet $\rightarrow$ Address = `{{ $json.wallet_address }}`.
   - Node 3: Ethereum Mainnet $\rightarrow$ Operation = `getGasPrice`.
   - Node 4: Polygon Mainnet $\rightarrow$ Operation = `getGasPrice`.
2. Connect all four to a **Merge** node configured to accept multiple inputs.

### Step 3: Data Processing
1. Add a **Code** node ("Extract Chain Metrics") to parse the merged array into a single JSON object with balance and gas variables.
2. Add an **HTTP Request** node. URL: `https://min-api.cryptocompare.com/data/pricemulti?fsyms={{ $json.coins.join(',') }}&tsyms=USD`.
3. Use a **Merge** node to combine the output of "Extract Chain Metrics" and "Fetch Crypto Prices".
4. Add a **Code** node ("Calculate Portfolio Metrics") to calculate:
   - `total_value = (eth_balance * eth_price) + (matic_balance * matic_price)`.
   - Calculate percentages for each asset.

### Step 4: AI Analysis Setup
1. Create an **AI Agent** node.
2. Attach a **Google Gemini Chat Model** (requires Google Palm API credentials).
3. Attach a **Structured Output Parser**. Define the JSON schema with properties: `insight`, `risk`, `suggestion`, `score`, `gas_analysis`, and `allocation_summary`.
4. In the Agent's prompt, pass the variables from the "Calculate Portfolio Metrics" node (e.g., `{{$json.total}}`).

### Step 5: Output Delivery
1. Use a **Merge** node to combine the "Calculate Portfolio Metrics" output and the "AI Agent" output.
2. Add a **Code** node to format the final string using Slack markdown (e.g., `*Total Value:* ${total}`).
3. Add a **Slack** node. Connect via OAuth2 and set the text to the output of the formatting node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Use a Cron node instead of a Manual Trigger for daily reports. | Scheduling Optimization |
| Ensure QuickNode API keys are specifically for the networks being tracked. | Credential Setup |
| The portfolio health score is generated by AI based on allocation and gas. | AI Logic |
| CryptoCompare API is used for real-time USD pricing. | External API |