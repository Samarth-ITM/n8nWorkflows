Detect WooCommerce order fraud and send alerts to Slack

https://n8nworkflows.xyz/workflows/detect-woocommerce-order-fraud-and-send-alerts-to-slack-14897


# Detect WooCommerce order fraud and send alerts to Slack

# Technical Documentation: WooCommerce Fraud Detection & Slack Alert Workflow

## 1. Workflow Overview
This workflow is designed to automatically monitor WooCommerce orders for potential fraud by analyzing specific risk indicators. It retrieves order data, evaluates it against a set of business rules (fraud signals), calculates a cumulative risk score, and triggers a notification to a Slack channel if the order exceeds a defined risk threshold.

### Logical Blocks
- **1.1 Order Retrieval & Preparation:** Handles the scheduled trigger, data fetching from WooCommerce, and normalization of fields.
- **1.2 Order Eligibility Check:** Filters out completed or cancelled orders, ensuring only "Pending" or "Processing" orders are analyzed.
- **1.3 Fraud Signal Detection:** A parallel processing block that evaluates four specific risk factors: Address Mismatch, High Order Value, Disposable Email usage, and Admin Creation status.
- **1.4 Fraud Scoring & Alerting:** Aggregates all signals, calculates a final numerical score via JavaScript, and sends a detailed alert to Slack if the score is $\ge 1$.

---

## 2. Block-by-Block Analysis

### 2.1 Order Retrieval & Preparation
**Overview:** Initiates the process and prepares a clean dataset for analysis.
- **Nodes Involved:** `Cron: Fetch Orders`, `Fetch Order (WooCommerce)`, `Normalize Order Data`, `Initialize Fraud Flags`.
- **Node Details:**
    - **Cron: Fetch Orders** (Schedule Trigger): Triggers the workflow based on a defined interval.
    - **Fetch Order (WooCommerce)** (WooCommerce): Retrieves a specific order (ID 511 in the template) using the `get` operation.
        - *Potential Failure:* API authentication errors or invalid Order ID.
    - **Normalize Order Data** (Set): Maps complex WooCommerce JSON to a flat structure (e.g., `billing_country`, `order_total`, `customer_ip`).
    - **Initialize Fraud Flags** (Set): Creates a baseline JSON object with all fraud flags set to `false` to prevent "undefined" errors during scoring.

### 2.2 Order Eligibility Check
**Overview:** Validates if the order is in a state that requires fraud checking.
- **Nodes Involved:** `Check Eligible Status (Pending/Processing)`.
- **Node Details:**
    - **Check Eligible Status** (If): Uses an `OR` condition to check if the status is either `pending` or `processing`.
        - *Logic:* If true, the workflow continues to the analysis block; otherwise, it stops.

### 2.3 Fraud Signal Detection
**Overview:** Evaluates the order against four independent risk criteria.
- **Nodes Involved:** `Check Address Mismatch`, `Flag: Address Mismatch`, `Check High Value (>500)`, `Flag: High Value Order`, `Prepare Email`, `Detect Disposable Email`, `Flag: Suspicious Email`, `Check Admin Created Order`, `Mark Admin Order`.
- **Node Details:**
    - **Address Mismatch Check:** (If) Compares `billing_postcode` vs `shipping_postcode` AND `billing_country` vs `shipping_country`. If they differ, `billing_shipping_mismatch` is flagged.
    - **High Value Check:** (If) Converts `order_total` to a number and checks if it is $> 500$. If true, `high_value_order` is set to `true`.
    - **Email Analysis:**
        - `Prepare Email` (Set): Isolates the email string.
        - `Detect Disposable Email` (If): Uses a Regex pattern `(mailinator|tempmail|10minutemail)` to identify burner emails.
        - `Flag: Suspicious Email` (Set): Sets `suspicious_email` to `true` if regex matches.
    - **Admin Validation:** (If) Checks if `created_via` equals `admin`. If so, it marks the order as admin-created (used later to reduce the fraud score).

### 2.4 Fraud Scoring & Alerting
**Overview:** Synthesizes all findings into a final decision.
- **Nodes Involved:** `Merge Fraud Signals`, `Calculate Fraud Score`, `Fraud Threshold Check`, `Send Slack Fraud Alert`.
- **Node Details:**
    - **Merge Fraud Signals** (Merge): Combines the outputs of the four parallel check branches into a single item using "Combine by Position".
    - **Calculate Fraud Score** (Code): Executes JavaScript logic to assign weights:
        - Address Mismatch: $+2$
        - High Value: $+3$
        - Suspicious Email: $+1$
        - Admin Created: $-2$ (reduces risk)
        - *Constraint:* Final score cannot be less than $0$.
    - **Fraud Threshold Check** (If): Checks if `fraud_score` $\ge 1$.
    - **Send Slack Fraud Alert** (Slack): Sends a formatted template to a specific channel including Order ID, Total, Customer Name, and the specific flags that triggered the alert.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Cron: Fetch Orders | Schedule Trigger | Trigger | - | Fetch Order (WooCommerce) | Order Retrieval & Preparation |
| Fetch Order (WooCommerce) | WooCommerce | Data Retrieval | Cron: Fetch Orders | Normalize Order Data | Order Retrieval & Preparation |
| Normalize Order Data | Set | Data Cleaning | Fetch Order (WooCommerce) | Initialize Fraud Flags | Order Retrieval & Preparation |
| Initialize Fraud Flags | Set | Variable Init | Normalize Order Data | Check Eligible Status | Order Retrieval & Preparation |
| Check Eligible Status | If | Filter | Initialize Fraud Flags | Address/Value/Email/Admin Checks | Order Eligibility Check |
| Check Address Mismatch | If | Risk Analysis | Check Eligible Status | Flag: Address Mismatch | Address Mismatch Detection |
| Flag: Address Mismatch | Set | Risk Tagging | Check Address Mismatch | Merge Fraud Signals | Address Mismatch Detection |
| Check High Value (>500) | If | Risk Analysis | Check Eligible Status | Flag: High Value Order | High-Value Order Detection |
| Flag: High Value Order | Set | Risk Tagging | Check High Value (>500) | Merge Fraud Signals | High-Value Order Detection |
| Prepare Email | Set | Data Prep | Check Eligible Status | Detect Disposable Email | Suspicious Email Detection |
| Detect Disposable Email | If | Risk Analysis | Prepare Email | Flag: Suspicious Email | Suspicious Email Detection |
| Flag: Suspicious Email | Set | Risk Tagging | Detect Disposable Email | Merge Fraud Signals | Suspicious Email Detection |
| Check Admin Created Order | If | Trust Analysis | Check Eligible Status | Mark Admin Order | Admin Order Validation |
| Mark Admin Order | Set | Trust Tagging | Check Admin Created Order | Merge Fraud Signals | Admin Order Validation |
| Merge Fraud Signals | Merge | Data Aggregation | All Flag Nodes | Calculate Fraud Score | Fraud Scoring & Alerting |
| Calculate Fraud Score | Code | Logic Engine | Merge Fraud Signals | Fraud Threshold Check | Fraud Scoring & Alerting |
| Fraud Threshold Check | If | Final Decision | Calculate Fraud Score | Send Slack Fraud Alert | Fraud Scoring & Alerting |
| Send Slack Fraud Alert | Slack | Notification | Fraud Threshold Check | - | Fraud Scoring & Alerting |

---

## 4. Reproducing the Workflow from Scratch

1.  **Trigger & Fetch:**
    - Create a **Schedule Trigger** node.
    - Add a **WooCommerce** node: Set Resource to `Order`, Operation to `Get`, and provide a valid Order ID. Configure WooCommerce API credentials.
2.  **Data Normalization:**
    - Add a **Set** node (`Normalize Order Data`): Map `id` $\to$ `order_id`, `total` $\to$ `order_total`, `billing.country` $\to$ `billing_country`, etc.
    - Add a second **Set** node (`Initialize Fraud Flags`): Set Mode to "Raw" and define a JSON object with `billing_shipping_mismatch`, `high_value_order`, `suspicious_email`, and `admin_created` all as `false`.
3.  **Eligibility Filter:**
    - Add an **If** node: Set a condition to check if `status` equals `pending` OR `processing`.
4.  **Parallel Risk Analysis (Create 4 branches from the 'True' output of the If node):**
    - **Branch 1 (Address):** **If** node (Compare billing vs shipping postcode/country) $\to$ **Set** node (Set `billing_shipping_mismatch` to `true`).
    - **Branch 2 (Value):** **If** node (Check if `order_total` $> 500$) $\to$ **Set** node (Set `high_value_order` to `true`).
    - **Branch 3 (Email):** **Set** node (Isolate email) $\to$ **If** node (Regex: `(mailinator|tempmail|10minutemail)`) $\to$ **Set** node (Set `suspicious_email` to `true`).
    - **Branch 4 (Admin):** **If** node (Check if `created_via` equals `admin`) $\to$ **Set** node (Set `created_via` to `admin`).
5.  **Aggregation & Scoring:**
    - Add a **Merge** node: Mode `Combine`, Combine by `Position`, Number of inputs `4`. Connect all four branch ends here.
    - Add a **Code** node: Paste the JavaScript logic to calculate the `fraud_score` based on the weights provided in Section 2.4.
6.  **Alerting:**
    - Add an **If** node: Check if `fraud_score` $\ge 1$.
    - Add a **Slack** node: Connect to a Slack API credential, select the channel, and use expressions to build the alert message using data from the `Fetch Order` and `Calculate Fraud Score` nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow performs automatic monitoring of incoming orders to prevent fraud. | General logic overview |
| Requires WooCommerce API keys and Slack OAuth2 credentials. | Integration Requirements |
| Uses Regex for disposable email detection; list can be expanded for better accuracy. | Technical Note |