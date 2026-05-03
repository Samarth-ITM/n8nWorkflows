Blockchain Monitor with ScrapeGraphAI Risk Detection, Instant Alerts and Slack

https://n8nworkflows.xyz/workflows/blockchain-monitor-with-scrapegraphai-risk-detection--instant-alerts-and-slack-6929


# Blockchain Monitor with ScrapeGraphAI Risk Detection, Instant Alerts and Slack

### 1. Workflow Overview

This workflow is designed as a **Blockchain Monitor with AI-enhanced risk detection**, providing real-time alerts on significant blockchain activity through Slack. It targets blockchain analysts, security teams, and operations monitoring multiple blockchains such as Ethereum, Bitcoin, Binance Smart Chain (BSC), and Polygon. The workflow focuses on extracting transactional data from blockchain explorers, assessing risk based on transaction volume and failure rates, and notifying stakeholders instantly.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Receives real-time blockchain block data via a webhook.
- **1.2 Data Normalization:** Standardizes and enriches raw input with explorer URLs and metadata.
- **1.3 AI Data Extraction:** Uses ScrapeGraphAI to scrape detailed transaction data from blockchain explorer pages.
- **1.4 Risk Analysis:** Computes risk scores based on the extracted transaction data.
- **1.5 Risk Filtering:** Determines if the risk level warrants alerting.
- **1.6 Alerting:** Sends formatted alerts to a Slack channel for team awareness.
- **1.7 Configuration and Documentation:** Notes and instructions embedded as sticky notes for clarity and maintainability.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block receives blockchain data from external sources through a webhook endpoint, acting as the workflow’s trigger point.

**Nodes Involved:**  
- 🔗 Blockchain Webhook  
- 📡 Input Info (sticky note)  
- 📋 Overview (sticky note)

**Node Details:**

- **🔗 Blockchain Webhook**  
  - *Type:* Webhook node (Trigger)  
  - *Role:* Listens for incoming HTTP POST requests carrying blockchain block data.  
  - *Configuration:*  
    - Webhook path set to a unique identifier (`eed656b3-6a7f-4460-92e0-802bca2522d0`).  
    - No authentication or additional options specified.  
  - *Input/Output:* No input; outputs raw JSON payload from webhook.  
  - *Edge Cases:*  
    - Missing or malformed data in payload can affect downstream nodes.  
    - Potential webhook downtime or unauthorized calls if not protected externally.  
  - *Sticky Notes:*  
    - 📡 Input Info explains expected data and sources such as Moralis, Alchemy, or exchange webhooks.

---

#### 2.2 Data Normalization

**Overview:**  
Transforms raw webhook data into a standardized format, ensuring consistency across multiple blockchains and enriching data with explorer URLs.

**Nodes Involved:**  
- 🔄 Normalize Data (Code node)  
- 🔧 Process Info (sticky note)

**Node Details:**

- **🔄 Normalize Data**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Extracts block number, blockchain type, timestamp, and generates a session ID. Maps blockchains to their respective explorer URLs.  
  - *Configuration Highlights:*  
    - Supports flexible input keys (`blockNumber`, `block_number`, `height`).  
    - Defaults blockchain to Ethereum if unspecified.  
    - Timestamp defaults to current date if missing.  
    - Explorer URLs predefined for Ethereum, Bitcoin, BSC, Polygon.  
  - *Input:* JSON from webhook node.  
  - *Output:* Normalized JSON with fields: `block_number`, `blockchain`, `timestamp`, `session_id`, `explorer_url`.  
  - *Edge Cases:*  
    - Unknown blockchain defaults to Ethereum explorer URL, which may cause inaccurate URLs.  
    - Missing block number or timestamp handled by fallback but may reduce data fidelity.  
  - *Sticky Notes:*  
    - 🔧 Process Info describes normalization goals and supported chains.

---

#### 2.3 AI Data Extraction

**Overview:**  
Utilizes ScrapeGraphAI to scrape and extract detailed transaction information from the blockchain explorer page URL generated in normalization.

**Nodes Involved:**  
- 🤖 ScrapeGraphAI  
- 🕷️ Scraping Info (sticky note)

**Node Details:**

- **🤖 ScrapeGraphAI**  
  - *Type:* ScrapeGraphAI node (specialized AI scraping)  
  - *Role:* Extracts structured transaction data from the provided explorer URL.  
  - *Configuration Highlights:*  
    - User prompt explicitly instructs to extract transaction arrays with fields such as `hash`, `from`, `to`, `value_usd`, `fee_usd`, `status`, and `is_contract`.  
    - Also extracts block summary data: total transactions, volume, high-value count, failed count.  
    - Focuses on confirmed data accuracy.  
    - Uses the explorer URL from normalized data as input URL.  
  - *Input:* Normalized JSON, specifically the `explorer_url`.  
  - *Output:* JSON containing transaction array and summary statistics.  
  - *Edge Cases:*  
    - If ScrapeGraphAI fails or returns incomplete data, downstream risk analysis may be inaccurate or fail.  
    - Network or API key issues with ScrapeGraphAI could cause node failure.  
  - *Sticky Notes:*  
    - 🕷️ Scraping Info details the AI extraction process and data types.

---

#### 2.4 Risk Analysis

**Overview:**  
Processes extracted transaction data to compute risk scores and classify the risk level of the block based on transaction volume, high-value transactions, and failure rates.

**Nodes Involved:**  
- ⚡ Risk Analyzer (Code node)  
- 📊 Analysis Info (sticky note)

**Node Details:**

- **⚡ Risk Analyzer**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Aggregates and analyzes transactions for risk factors, assigns risk scores and levels, and generates alerts based on thresholds.  
  - *Configuration Highlights:*  
    - Initializes analysis object with block metadata.  
    - Iterates over transactions to sum volume, identify high-value transactions (> $10,000), and count failed transactions.  
    - Uses summary data from ScrapeGraphAI if available to supplement counts.  
    - Applies weighted risk scoring: +15 per high-value transaction, +20 if total volume > $1,000,000, +10 if > $100,000, +15 if failure rate > 10%.  
    - Caps risk score at 100.  
    - Determines risk level: low (<25), medium (25-49), high (≥50).  
    - Generates alerts for high-risk conditions (e.g., multiple high-value txs, high failure rates).  
  - *Input:* ScrapeGraphAI output and normalized block data (via node reference).  
  - *Output:* JSON risk analysis summary with scores, levels, and alert messages.  
  - *Edge Cases:*  
    - Missing or malformed transaction data can cause runtime errors or inaccurate scoring.  
    - Unexpected data types require robust parsing.  
  - *Sticky Notes:*  
    - 📊 Analysis Info explains risk factors and scoring thresholds.

---

#### 2.5 Risk Filtering

**Overview:**  
Filters the analyzed risk data to pass only blocks with significant risk levels (medium or high) to the alerting system.

**Nodes Involved:**  
- 🚨 Risk Filter (IF node)  
- 🎯 Filter Info (sticky note)

**Node Details:**

- **🚨 Risk Filter**  
  - *Type:* IF node (conditional logic)  
  - *Role:* Checks if the risk level is not "low" to decide if an alert should be triggered.  
  - *Configuration Highlights:*  
    - Condition: `risk_level` ≠ "low" (case sensitive, strict type check).  
    - If true, routes to Slack alert node; otherwise, stops here.  
  - *Input:* Risk analysis JSON.  
  - *Output:* Passes data only if medium or high risk.  
  - *Edge Cases:*  
    - Risk level missing or malformed will cause condition to fail or pass incorrectly.  
  - *Sticky Notes:*  
    - 🎯 Filter Info outlines the filtering criteria and alert routing logic.

---

#### 2.6 Alerting

**Overview:**  
Sends formatted alerts to a configured Slack channel, providing real-time notifications of risky blockchain blocks.

**Nodes Involved:**  
- 📱 Slack Alert (Slack node)  
- 🔔 Alert Info (sticky note)  
- ⚙️ Configuration (sticky note)

**Node Details:**

- **📱 Slack Alert**  
  - *Type:* Slack node (message creation)  
  - *Role:* Posts messages to Slack using configured credentials and webhook/token.  
  - *Configuration Highlights:*  
    - Operation set to "create" message.  
    - Uses Slack webhook or bot token credentials (configured externally).  
    - Message formatting expects JSON from previous node to include block info, risk score, transaction stats, and alerts.  
  - *Input:* Filtered risk data from Risk Filter node.  
  - *Output:* Confirmation of Slack message sent.  
  - *Edge Cases:*  
    - Slack API errors (auth failures, rate limits, channel not found).  
    - Message formatting errors if input data is incomplete or malformed.  
  - *Sticky Notes:*  
    - 🔔 Alert Info describes alert content and formatting.  
    - ⚙️ Configuration lists required setup steps and customizable parameters.

---

#### 2.7 Configuration and Documentation

**Overview:**  
Sticky notes throughout the workflow provide essential documentation, setup instructions, and overview information for maintainers and users.

**Nodes Involved:**  
- 📋 Overview  
- 📡 Input Info  
- 🔧 Process Info  
- 🕷️ Scraping Info  
- 📊 Analysis Info  
- 🎯 Filter Info  
- 🔔 Alert Info  
- ⚙️ Configuration

**Node Details:**

- *Type:* Sticky Note nodes  
- *Role:* Provide contextual information, project overview, configuration requirements, and detailed explanations of each step.  
- *Use:* Aid understanding, maintenance, and onboarding.  
- *Sticky Note Examples:*  
  - Overview explains purpose and supported blockchains.  
  - Configuration lists API keys and tokens needed, plus adjustable thresholds.  
  - Filter Info and Alert Info clarify alerting logic and Slack integration.

---

### 3. Summary Table

| Node Name             | Node Type             | Functional Role          | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                 |
|-----------------------|-----------------------|-------------------------|------------------------|-----------------------|---------------------------------------------------------------------------------------------|
| 🔗 Blockchain Webhook  | Webhook               | Input Reception         | —                      | 🔄 Normalize Data      | 📡 Input Info: Receives blockchain data from nodes/APIs/exchanges.                         |
| 🔄 Normalize Data      | Code                  | Data Normalization      | 🔗 Blockchain Webhook   | 🤖 ScrapeGraphAI       | 🔧 Process Info: Normalizes and enriches blockchain data for multi-chain support.          |
| 🤖 ScrapeGraphAI       | ScrapeGraphAI         | AI Data Extraction      | 🔄 Normalize Data       | ⚡ Risk Analyzer       | 🕷️ Scraping Info: Extracts structured transaction data from explorer pages.               |
| ⚡ Risk Analyzer       | Code                  | Risk Analysis           | 🤖 ScrapeGraphAI, 🔄 Normalize Data (reference) | 🚨 Risk Filter        | 📊 Analysis Info: Calculates risk scores and levels based on transaction data.             |
| 🚨 Risk Filter         | IF                    | Risk Filtering          | ⚡ Risk Analyzer        | 📱 Slack Alert         | 🎯 Filter Info: Filters medium/high risk blocks for alerting.                              |
| 📱 Slack Alert         | Slack                 | Alerting                | 🚨 Risk Filter          | —                     | 🔔 Alert Info, ⚙️ Configuration: Sends formatted alerts to Slack; requires API keys.       |
| 📋 Overview           | Sticky Note           | Documentation           | —                      | —                     | Overview of the entire workflow and features.                                              |
| 📡 Input Info          | Sticky Note           | Documentation           | —                      | —                     | Details input sources and expected data format.                                            |
| 🔧 Process Info        | Sticky Note           | Documentation           | —                      | —                     | Explains normalization and supported chains.                                              |
| 🕷️ Scraping Info      | Sticky Note           | Documentation           | —                      | —                     | Describes AI extraction process and output data.                                          |
| 📊 Analysis Info       | Sticky Note           | Documentation           | —                      | —                     | Details risk scoring methodology and thresholds.                                          |
| 🎯 Filter Info         | Sticky Note           | Documentation           | —                      | —                     | Explains filter criteria and alert routing logic.                                         |
| 🔔 Alert Info          | Sticky Note           | Documentation           | —                      | —                     | Details Slack alert contents and formatting.                                              |
| ⚙️ Configuration       | Sticky Note           | Documentation           | —                      | —                     | Lists setup steps, credentials needed, and customizable parameters.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: 🔗 Blockchain Webhook  
   - Configure path: `eed656b3-6a7f-4460-92e0-802bca2522d0` (unique identifier)  
   - No authentication required by default  
   - Position: starting point of workflow  

2. **Create Code Node for Normalization**  
   - Type: Code (JavaScript)  
   - Name: 🔄 Normalize Data  
   - Connect input from: 🔗 Blockchain Webhook  
   - Paste the provided JavaScript code to:  
     - Extract `block_number`, `blockchain`, `timestamp`, `session_id`  
     - Map explorer URLs for Ethereum, Bitcoin, BSC, Polygon  
     - Default blockchain to 'ethereum' if missing  
   - Position: downstream of webhook  

3. **Create ScrapeGraphAI Node**  
   - Type: ScrapeGraphAI  
   - Name: 🤖 ScrapeGraphAI  
   - Connect input from: 🔄 Normalize Data  
   - Set parameter `websiteUrl` to expression: `{{$json["explorer_url"]}}`  
   - Paste the user prompt instructing to extract transaction data and summary as described  
   - Ensure ScrapeGraphAI credentials/API key configured in n8n  

4. **Create Risk Analyzer Code Node**  
   - Type: Code (JavaScript)  
   - Name: ⚡ Risk Analyzer  
   - Connect input from: 🤖 ScrapeGraphAI  
   - Reference normalized data node: use `$('Normalize Data').first().json` in code  
   - Paste provided JavaScript code that:  
     - Aggregates transactions, calculates volume, counts failures, identifies high-value txs  
     - Computes risk score and level  
     - Generates alert messages  
   - Position accordingly  

5. **Create IF Node for Risk Filtering**  
   - Type: IF  
   - Name: 🚨 Risk Filter  
   - Connect input from: ⚡ Risk Analyzer  
   - Configure condition:  
     - Check if `risk_level` (string, case sensitive) is NOT equal to `low`  
   - Output 'true' branch: connect to Slack Alert node  
   - Output 'false' branch: no downstream connection  

6. **Create Slack Node for Alerts**  
   - Type: Slack  
   - Name: 📱 Slack Alert  
   - Connect input from: 🚨 Risk Filter (true branch)  
   - Set operation to "Create" message  
   - Configure Slack credentials: webhook URL or bot token with chat permissions  
   - Compose message using data from risk analysis: block info, risk score, alerts, and transaction stats  
   - Position at workflow end  

7. **Add Sticky Notes for Documentation**  
   - Add sticky notes at appropriate positions with content for:  
     - Overview  
     - Input Info  
     - Process Info  
     - Scraping Info  
     - Analysis Info  
     - Filter Info  
     - Alert Info  
     - Configuration (include API keys, webhook URLs, and customizable thresholds)  

8. **Set Execution and Version Settings**  
   - Ensure nodes are connected in order:  
     🔗 Blockchain Webhook → 🔄 Normalize Data → 🤖 ScrapeGraphAI → ⚡ Risk Analyzer → 🚨 Risk Filter → 📱 Slack Alert  
   - Verify credentials for ScrapeGraphAI and Slack are properly set  
   - Optional: Configure webhook external access for real-time blockchain data input  
   - Save and activate workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                      | Context or Link                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Workflow supports multi-chain monitoring with real-time blockchain data ingestion and AI-powered transaction scraping.                          | Overview sticky note                                               |
| Requires ScrapeGraphAI API key for AI extraction; Slack webhook or bot token for alerts.                                                          | Configuration sticky note                                          |
| Risk scoring is customizable by adjusting thresholds in the Risk Analyzer code node.                                                             | Configuration sticky note                                          |
| Slack alerts include emojis and concise summaries for quick situational awareness.                                                                | Alert Info sticky note                                             |
| ScrapeGraphAI prompt designed to prioritize confirmed transaction data and exclude unconfirmed or irrelevant information.                        | Scraping Info sticky note                                          |
| Input expected from blockchain nodes or third-party API webhooks like Moralis or Alchemy; ensure webhook URL is accessible externally.            | Input Info sticky note                                             |
| Risk levels: low (<25), medium (25-49), high (≥50), with alerts triggered only for medium and high risk to reduce false positives.                 | Analysis Info sticky note                                          |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a no-code integration and automation platform. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly available.