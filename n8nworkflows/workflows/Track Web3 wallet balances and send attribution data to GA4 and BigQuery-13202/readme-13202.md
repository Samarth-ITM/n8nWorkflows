Track Web3 wallet balances and send attribution data to GA4 and BigQuery

https://n8nworkflows.xyz/workflows/track-web3-wallet-balances-and-send-attribution-data-to-ga4-and-bigquery-13202


# Track Web3 wallet balances and send attribution data to GA4 and BigQuery

### 1. Workflow Overview

This workflow is designed to bridge Web3 user activity with traditional marketing analytics and real-time communication tools. It automates the process of tracking cryptocurrency wallet balances when a user connects their wallet to a website (via Google Tag Manager).

The logic is divided into four functional stages:
1.  **Ingestion & Enrichment:** Receives the wallet address and session metadata, then queries the Zerion API to determine the wallet's USD value.
2.  **Data Normalization:** Centralizes and formats all variables (balances, IDs, and session data) for downstream use.
3.  **Analytics Syncing:** Simultaneously updates Google Analytics 4 (via Measurement Protocol) and Google BigQuery to maintain a record of wallet-to-session attribution.
4.  **Threshold Alerting:** Evaluates the wallet’s value and triggers a detailed "Whale Alert" in Discord if the balance exceeds a specific threshold ($50 in this configuration).

---

### 2. Block-by-Block Analysis

#### 2.1 Input & API Enrichment
This block handles the initial reception of data and fetches the necessary financial metrics from the blockchain.

*   **Nodes Involved:**
    *   `Webhook - Receive data from GTM`
    *   `HTTP Request - Query wallet balance in Zerion`
*   **Node Details:**
    *   **Webhook - Receive data from GTM:**
        *   **Type:** Webhook (v2.1)
        *   **Configuration:** Listens for `POST` requests at the path `wallet_ingest`.
        *   **Input/Output:** Receives `wallet_address`, `ga_client_id`, `ga_session_id`, and `hashed_wallet_id` from Google Tag Manager.
    *   **HTTP Request - Query wallet balance in Zerion:**
        *   **Type:** HTTP Request (v4.3)
        *   **Configuration:** Performs a GET request to Zerion’s API (`/v1/wallets/{{address}}/portfolio`).
        *   **Authentication:** Requires Generic HTTP Basic Auth or Header Auth.
        *   **Edge Cases:** Failure may occur if the wallet address is invalid or if the Zerion API rate limit is reached.

#### 2.2 Data Preparation
This block acts as a data mapper to ensure subsequent nodes have clean access to both the trigger data and the API response.

*   **Nodes Involved:** `Edit Fields`
*   **Node Details:**
    *   **Edit Fields:**
        *   **Type:** Set (v3.4)
        *   **Configuration:** Creates five key variables: `wallet_usd_balance` (from Zerion), `hashed_wallet_id`, `ga_client_id`, `ga_session_id`, and `wallet_id`.
        *   **Logic:** Uses expressions to reference the Webhook's original body and the Zerion response.

#### 2.3 Analytics Integration
This block pushes the enriched data into the marketing and data warehouse stacks.

*   **Nodes Involved:**
    *   **HTTP Request - Send Measurement Protocol Hit Wallet Balance**
    *   **BigQuery - Send wallet address to a table**
*   **Node Details:**
    *   **GA4 Measurement Protocol Hit:**
        *   **Type:** HTTP Request (v4.3)
        *   **Configuration:** Sends a POST request to GA4's Measurement Protocol endpoint using a `measurement_id` and `api_secret`.
        *   **Payload:** Sends a `connect_wallet` event containing the `wallet_usd_balance` linked to the existing `ga_session_id`.
    *   **BigQuery - Send wallet address to a table:**
        *   **Type:** Google BigQuery (v2.1)
        *   **Configuration:** Inserts a row into the `wallets.wallet_mapping` table.
        *   **Technical Role:** Maps the physical wallet address to the hashed ID and GA session IDs for future SQL-based attribution.

#### 2.4 Alerting Logic
The final block filters for high-value users and notifies the team.

*   **Nodes Involved:**
    *   **Switch - Check if balance>50 to know whale**
    *   **Send a message to admin channel - whale wallet balance details**
*   **Node Details:**
    *   **Switch (Threshold Check):**
        *   **Type:** Switch (v3.4)
        *   **Logic:** Checks if `wallet_usd_balance` is Greater Than or Equal to `50`.
    *   **Discord Whale Alert:**
        *   **Type:** Discord (v2)
        *   **Configuration:** Sends a formatted message to a specific channel.
        *   **Key Expression:** Includes a JavaScript snippet that filters the `positions_distribution_by_chain` to only show chains where the user holds more than $0.10. It also generates a direct link to the Zerion Portfolio view.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - Receive data from GTM | Webhook | Entry Point | None | Zerion Query | Workflow overview and setup instructions. |
| HTTP Request - Query wallet balance in Zerion | HTTP Request | Data Enrichment | Webhook | Edit Fields | Workflow overview and setup instructions. |
| Edit Fields | Set | Data Normalization | Zerion Query | GA4, BigQuery, Switch | Workflow overview and setup instructions. |
| HTTP Request - Send Measurement Protocol Hit Wallet Balance | HTTP Request | Analytics Sync | Edit Fields | None | Workflow overview and setup instructions. |
| BigQuery - Send wallet address to a table | BigQuery | Data Warehousing | Edit Fields | None | Workflow overview and setup instructions. |
| Switch - Check if balance>50 to know whale | Switch | Condition Logic | Edit Fields | Discord Alert | Workflow overview and setup instructions. |
| Send a message to admin channel - whale wallet balance details | Discord | Notification | Switch | None | Workflow overview and setup instructions. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Webhook** node. Set the method to `POST` and the path to `wallet_ingest`.
2.  **External Enrichment:**
    *   Add an **HTTP Request** node. Set the URL to `https://api.zerion.io/v1/wallets/{{$json.body.wallet_address}}/portfolio?currency=usd`.
    *   Configure Basic Auth credentials using your Zerion API Key.
3.  **Data Structuring:**
    *   Add an **Edit Fields (Set)** node.
    *   Map `wallet_usd_balance` to the Zerion response: `{{$json.data.attributes.total.positions}}`.
    *   Map the remaining IDs (`ga_client_id`, `ga_session_id`, `wallet_id`) from the Webhook node's output.
4.  **GA4 Integration:**
    *   Add an **HTTP Request** node (POST).
    *   URL: `https://www.google-analytics.com/mp/collect?measurement_id=YOUR_ID&api_secret=YOUR_SECRET`.
    *   Body Type: JSON. Construct an event named `connect_wallet` passing the balance and session ID.
5.  **BigQuery Integration:**
    *   Add a **Google BigQuery** node. Set the operation to `Insert`.
    *   Select your Project, Dataset (`wallets`), and Table (`wallet_mapping`).
    *   Map the fields: `wallet_address`, `hashed_wallet_id`, `ga_client_id`, and `ga_session_id`.
6.  **Filter Logic:**
    *   Add a **Switch** node. Set the data type to Number.
    *   Condition: `{{ $json.wallet_usd_balance }}` >= `50`.
7.  **Alerting:**
    *   Add a **Discord** node connected to the Switch's "true" output.
    *   Use the "Send Message" resource. Format the content using the chain breakdown expression to provide detail on which blockchains the whale holds assets.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Video Walkthrough** | [YouTube Tutorial](https://youtu.be/2_wuTRzRpkg) |
| **Data Mapping Strategy** | Using `hashed_wallet_id` as a join key between GA4 and BigQuery for privacy-compliant tracking. |
| **Zerion API Dependency** | Requires an active Zerion API key for portfolio balance lookups. |
| **Measurement Protocol** | Used to bypass browser-side tracking limitations and ensure session continuity. |