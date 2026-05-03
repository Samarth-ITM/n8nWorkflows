Generate Ethereum & Solana Blockchain Audit Reports with PDF Export to Drive & Notion

https://n8nworkflows.xyz/workflows/generate-ethereum---solana-blockchain-audit-reports-with-pdf-export-to-drive---notion-8547


# Generate Ethereum & Solana Blockchain Audit Reports with PDF Export to Drive & Notion

### 1. Workflow Overview

This workflow automates the generation of multi-chain blockchain transaction audit reports for Ethereum and Solana. It is designed for enterprise use cases requiring robust transaction monitoring, risk scoring, compliance checks, and audit trail creation. The workflow integrates blockchain event reception, transaction data fetching and processing, PDF report generation, storage, and notifications.

Logical blocks:

- **1.1 Input Reception:** Captures incoming blockchain transaction events via a webhook.
- **1.2 Blockchain Transaction Monitoring:** Fetches detailed transaction data from Ethereum (via Alchemy API) and Solana (via Solana API).
- **1.3 Transaction Data Processing:** Processes and analyzes transaction data, calculates risk scores and compliance status using custom code.
- **1.4 Audit Report Generation:** Generates PDF audit reports using APITemplate.io based on processed data.
- **1.5 Storage and Indexing:** Uploads generated reports to Google Drive and creates corresponding entries in Notion for record-keeping.
- **1.6 Notifications:** Sends email notifications to the finance team with audit report details.
- **1.7 Smart Contract Audit Trail:** Verifies contract ABI and audit trail data via Etherscan or Solscan APIs.
- **1.8 Webhook Response:** Confirms successful processing to the original event sender.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Entry point that receives blockchain transaction events from external services via HTTP POST webhook.
- **Nodes Involved:** Blockchain Event Webhook
- **Node Details:**

  - **Blockchain Event Webhook**
    - Type: Webhook node (HTTP entry point)
    - Configuration: POST method, path set to `blockchain-transaction`, immediate response mode after processing
    - Inputs: External HTTP POST requests
    - Outputs: Triggers both Ethereum and Solana transaction monitoring nodes
    - Edge Cases: Webhook failures, malformed JSON, unauthorized requests (authentication not configured here)
    - Sticky Note: Explains this is the system entry point, can be called by external services

#### 1.2 Blockchain Transaction Monitoring

- **Overview:** Fetches detailed transaction data from blockchain APIs for Ethereum and Solana.
- **Nodes Involved:** Ethereum Transaction Monitor, Solana Transaction Monitor
- **Node Details:**

  - **Ethereum Transaction Monitor**
    - Type: HTTP Request node
    - Configuration: POST to Alchemy API endpoint using API key credential; sends JSON-RPC requests (e.g., eth_getLogs)
    - Headers: Content-Type application/json
    - Authentication: HTTP Header Auth with API key
    - Inputs: Triggered by Webhook
    - Outputs: Sends fetched Ethereum transaction data to processing
    - Edge Cases: API key invalid/expired, rate limits, network timeouts
    - Sticky Note: Uses Alchemy API key, fetches data via eth_getLogs

  - **Solana Transaction Monitor**
    - Type: HTTP Request node
    - Configuration: POST to `https://api.mainnet-beta.solana.com` with JSON body to fetch transaction info
    - Headers: Content-Type application/json
    - Inputs: Triggered by Webhook
    - Outputs: Sends fetched Solana transaction data to processing
    - Edge Cases: API downtime, malformed requests, rate limiting
    - Sticky Note: Direct call to Solana mainnet API

#### 1.3 Transaction Data Processing

- **Overview:** Processes raw blockchain data into structured audit data, calculates risk score and compliance status.
- **Nodes Involved:** Transaction Data Processor
- **Node Details:**

  - **Transaction Data Processor**
    - Type: Code node (JavaScript)
    - Configuration:
      - Parses raw transaction JSON
      - Extracts key fields: transaction hash, block number/slot, sender, recipient, value, gas/compute units, timestamps, contract addresses, event data
      - Implements risk scoring logic based on transaction value, missing addresses, failure status, and contract interactions
      - Performs placeholder compliance checks (KYC, sanctions, jurisdiction, reporting requirement)
      - Returns processed data array for downstream nodes
    - Inputs: Data from Ethereum and Solana monitors
    - Outputs: Processed transaction data for PDF generation, Notion entry, audit trail
    - Edge Cases: Unexpected data formats, missing fields, calculation errors
    - Sticky Note: Contains AI-like logic for risk and compliance checks

#### 1.4 Audit Report Generation

- **Overview:** Generates a PDF audit report from processed transaction data using external API (APITemplate.io).
- **Nodes Involved:** Audit Report PDF Generator
- **Node Details:**

  - **Audit Report PDF Generator**
    - Type: HTTP Request node
    - Configuration: POST to APITemplate.io with prepared request body containing transaction audit details; uses HTTP header authentication
    - Inputs: Processed data from Transaction Data Processor
    - Outputs: PDF file or URL for upload
    - Edge Cases: API failures, invalid template ID, authentication errors
    - Sticky Note: Uses APITemplate.io, requires template ID configuration

#### 1.5 Storage and Indexing

- **Overview:** Uploads the generated PDF report to Google Drive and indexes audit data in Notion database.
- **Nodes Involved:** Upload to Google Drive, Create Notion Audit Entry
- **Node Details:**

  - **Upload to Google Drive**
    - Type: Google Drive node
    - Configuration: Uploads PDF named with transaction hash and date to root folder or specified Drive folder
    - Credentials: OAuth2 with Google Drive account
    - Inputs: PDF from Audit Report PDF Generator
    - Outputs: Trigger notification node
    - Edge Cases: Authentication expiration, quota exceeded, permission errors
    - Sticky Note: Naming convention: Blockchain_Audit_Report_{hash}_{date}.pdf

  - **Create Notion Audit Entry**
    - Type: Notion node
    - Configuration: Creates page in Notion database with fields such as transaction hash, blockchain, risk score, date, compliance status, report link
    - Credentials: Notion API token with database access
    - Inputs: Processed transaction data
    - Outputs: Triggers Finance Team Notification
    - Edge Cases: API rate limits, schema mismatch, authentication failures
    - Sticky Note: Stores structured audit info in Notion

#### 1.6 Notifications

- **Overview:** Sends email notifications to finance and compliance teams with audit report details.
- **Nodes Involved:** Finance Team Notification
- **Node Details:**

  - **Finance Team Notification**
    - Type: Email Send node
    - Configuration:
      - Sends email from `blockchain-audit@company.com` to `finance-team@company.com`
      - CC: compliance@company.com
      - BCC: audit-logs@company.com
      - Subject includes blockchain and transaction hash
      - SMTP credentials configured
    - Inputs: Triggered from Notion entry creation
    - Outputs: Final webhook response
    - Edge Cases: SMTP server errors, email delivery failures
    - Sticky Note: Includes CC and BCC, full format

#### 1.7 Smart Contract Audit Trail

- **Overview:** Verifies smart contract ABI for the transaction using blockchain-specific explorer APIs.
- **Nodes Involved:** Smart Contract Audit Trail
- **Node Details:**

  - **Smart Contract Audit Trail**
    - Type: HTTP Request node
    - Configuration:
      - Chooses API endpoint dynamically based on blockchain (Etherscan for Ethereum, Solscan for Solana)
      - Sends GET query for contract ABI with contract address and API key
      - Headers: Content-Type application/json
      - Authentication: API key for Etherscan
    - Inputs: Processed transaction data
    - Outputs: Final webhook response if triggered independently
    - Edge Cases: Invalid contract address, API key issues, response delays
    - Sticky Note: Used for blockchain-native audit trail verification

#### 1.8 Webhook Response

- **Overview:** Sends HTTP 200 OK response back to the original webhook caller confirming successful workflow completion.
- **Nodes Involved:** Webhook Response
- **Node Details:**

  - **Webhook Response**
    - Type: Respond to Webhook node
    - Configuration: Returns HTTP status 200
    - Inputs: Triggered by completion of upload or notification nodes
    - Outputs: None (terminal node)
    - Edge Cases: Response timing errors if upstream nodes fail
    - Sticky Note: Confirms all processes succeeded

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                                  | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                  |
|-------------------------------|-----------------------|-------------------------------------------------|-------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Blockchain Event Webhook       | Webhook               | Entry point for blockchain transaction events  | External HTTP POST             | Ethereum Transaction Monitor, Solana Transaction Monitor | Entry point system, can be called by external service. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Ethereum Transaction Monitor   | HTTP Request          | Fetch Ethereum transaction details from Alchemy| Blockchain Event Webhook       | Transaction Data Processor        | Uses Alchemy API key, fetches data via eth_getLogs. [Guide](https://docs.n8n.io/workflows/sticky-notes/)   |
| Solana Transaction Monitor     | HTTP Request          | Fetch Solana transaction details                 | Blockchain Event Webhook       | Transaction Data Processor        | Direct call to Solana mainnet API. [Guide](https://docs.n8n.io/workflows/sticky-notes/)                  |
| Transaction Data Processor     | Code                  | Process and analyze transaction data, risk, compliance | Ethereum Transaction Monitor, Solana Transaction Monitor | Audit Report PDF Generator, Create Notion Audit Entry, Smart Contract Audit Trail | AI logic for risk scoring and compliance checks. [Guide](https://docs.n8n.io/workflows/sticky-notes/)         |
| Audit Report PDF Generator     | HTTP Request          | Generate PDF audit report via APITemplate.io    | Transaction Data Processor     | Upload to Google Drive            | Uses APITemplate.io, requires template ID. [Guide](https://docs.n8n.io/workflows/sticky-notes/)               |
| Upload to Google Drive         | Google Drive          | Store generated PDF audit report                 | Audit Report PDF Generator     | Webhook Response                 | Naming: Blockchain_Audit_Report_{hash}_{date}.pdf [Guide](https://docs.n8n.io/workflows/sticky-notes/)         |
| Create Notion Audit Entry      | Notion                | Index audit data in Notion database              | Transaction Data Processor     | Finance Team Notification         | Stores structured audit info: Hash, Blockchain, Risk, Compliance. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Finance Team Notification      | Email Send            | Notify finance and compliance teams              | Create Notion Audit Entry      | Webhook Response                 | CC compliance, BCC audit-logs, detailed format. [Guide](https://docs.n8n.io/workflows/sticky-notes/)            |
| Smart Contract Audit Trail     | HTTP Request          | Verify contract ABI via Etherscan/Solscan        | Transaction Data Processor     | Webhook Response                 | Verify contract ABI via blockchain explorers. [Guide](https://docs.n8n.io/workflows/sticky-notes/)              |
| Webhook Response              | Respond to Webhook    | Confirm successful processing                     | Upload to Google Drive, Finance Team Notification, Smart Contract Audit Trail | None                            | Confirms all processes succeeded. [Guide](https://docs.n8n.io/workflows/sticky-notes/)                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**
   - Type: Webhook
   - Name: Blockchain Event Webhook
   - HTTP Method: POST
   - Path: blockchain-transaction
   - Response Mode: Response Node
   - No authentication configured here (can be added if needed)

2. **Create Ethereum Transaction Monitor HTTP Request Node**
   - Type: HTTP Request
   - Name: Ethereum Transaction Monitor
   - Method: POST
   - URL: `https://api.alchemy.com/v2/{{ $credentials.alchemy.apiKey }}/`
   - Headers: Content-Type: application/json
   - Authentication: HTTP Header Auth using Alchemy API key credential
   - Body: JSON-RPC call for `eth_getLogs` or other appropriate method
   - Connect input from Blockchain Event Webhook

3. **Create Solana Transaction Monitor HTTP Request Node**
   - Type: HTTP Request
   - Name: Solana Transaction Monitor
   - Method: POST
   - URL: `https://api.mainnet-beta.solana.com`
   - Headers: Content-Type: application/json
   - Body: JSON request for Solana transaction info
   - Connect input from Blockchain Event Webhook

4. **Create Transaction Data Processor Code Node**
   - Type: Code (JavaScript)
   - Name: Transaction Data Processor
   - Paste the provided JavaScript code that:
     - Maps incoming transaction data (Ethereum and Solana)
     - Extracts key fields (hash, block/slot, from, to, value, gas, etc.)
     - Calculates risk score based on value, missing fields, status, contract interactions
     - Performs compliance checks (KYC, sanctions, jurisdiction, reporting threshold)
   - Inputs: Connect from both Ethereum and Solana Transaction Monitor nodes

5. **Create Audit Report PDF Generator HTTP Request Node**
   - Type: HTTP Request
   - Name: Audit Report PDF Generator
   - Method: POST
   - URL: `https://api.apitemplate.io/v2/create`
   - Headers: Content-Type: application/json
   - Authentication: HTTP Header Auth using APITemplate.io API key credential
   - Body: Construct with transaction data passed from Transaction Data Processor
   - Connect input from Transaction Data Processor

6. **Create Upload to Google Drive Node**
   - Type: Google Drive
   - Name: Upload to Google Drive
   - Operation: Upload File
   - File Name: `Blockchain_Audit_Report_{{ $json.transactionHash }}_{{ new Date().toISOString().split('T')[0] }}.pdf`
   - Folder: Root or specific folder ID
   - Credentials: Google Drive OAuth2 credentials
   - Connect input from Audit Report PDF Generator

7. **Create Create Notion Audit Entry Node**
   - Type: Notion
   - Name: Create Notion Audit Entry
   - Operation: Create Page in Database
   - Database ID: Use Notion audit database credential (`auditDatabaseId`)
   - Fields: Map transaction hash, blockchain, risk score, date, compliance check results, and report link
   - Credentials: Notion API credentials
   - Connect input from Transaction Data Processor

8. **Create Finance Team Notification Email Node**
   - Type: Email Send
   - Name: Finance Team Notification
   - From: blockchain-audit@company.com
   - To: finance-team@company.com
   - CC: compliance@company.com
   - BCC: audit-logs@company.com
   - Subject: `🔗 New Blockchain Transaction Audit Report - {{ $json.blockchain }} {{ $json.transactionHash }}`
   - Credentials: SMTP credentials configured for sending email
   - Connect input from Create Notion Audit Entry

9. **Create Smart Contract Audit Trail HTTP Request Node**
   - Type: HTTP Request
   - Name: Smart Contract Audit Trail
   - Method: POST
   - URL: Use expression to switch between:
     - Ethereum: `https://api.etherscan.io/api`
     - Solana: `https://api.solscan.io/transaction`
   - Query parameters: module=contract, action=getabi, address=contractAddress, apikey=etherscan API key
   - Headers: Content-Type: application/json
   - Credentials: Etherscan API key credential
   - Connect input from Transaction Data Processor

10. **Create Webhook Response Node**
    - Type: Respond to Webhook
    - Name: Webhook Response
    - Status Code: 200
    - Connect inputs from:
      - Upload to Google Drive
      - Finance Team Notification
      - Smart Contract Audit Trail (if independent)
    
11. **Connect Nodes according to Workflow Logic**
    - Blockchain Event Webhook → Ethereum Transaction Monitor and Solana Transaction Monitor
    - Both Transaction Monitor nodes → Transaction Data Processor
    - Transaction Data Processor → Audit Report PDF Generator, Create Notion Audit Entry, Smart Contract Audit Trail
    - Audit Report PDF Generator → Upload to Google Drive
    - Upload to Google Drive → Webhook Response
    - Create Notion Audit Entry → Finance Team Notification
    - Finance Team Notification → Webhook Response

12. **Credentials Setup**
    - Alchemy API key (HTTP Header Auth)
    - Google Drive OAuth2 account
    - Notion API token with database access and auditDatabaseId
    - SMTP email account credentials
    - Etherscan API key for contract audit
    - APITemplate.io API key for PDF generation

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Guide for sticky notes editing and usage                                                           | https://docs.n8n.io/workflows/sticky-notes/                   |
| APITemplate.io required for PDF report generation, needs valid template ID configured               | https://apitemplate.io/                                      |
| Alchemy API usage for Ethereum transaction monitoring                                               | https://docs.alchemy.com/alchemy/                            |
| Solana Mainnet RPC API endpoint                                                                     | https://docs.solana.com/developing/clients/jsonrpc-api       |
| Etherscan API documentation for contract ABI retrieval                                             | https://docs.etherscan.io/api-endpoints/contracts             |
| Solscan API for Solana contract info                                                               | https://public-api.solscan.io/                                |
| Best practice: secure API keys and credentials in n8n credentials manager                           | https://docs.n8n.io/integrations/credentials/                 |
| Compliance placeholders in code require integration with KYC and sanctions APIs for production use | Custom integration recommended                                |

---

This structured documentation fully describes the workflow, enabling users and automation agents to understand, reproduce, and maintain the multi-chain blockchain audit report automation system.