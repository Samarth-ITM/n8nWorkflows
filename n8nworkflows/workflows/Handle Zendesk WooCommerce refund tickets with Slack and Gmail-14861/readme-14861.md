Handle Zendesk WooCommerce refund tickets with Slack and Gmail

https://n8nworkflows.xyz/workflows/handle-zendesk-woocommerce-refund-tickets-with-slack-and-gmail-14861


# Handle Zendesk WooCommerce refund tickets with Slack and Gmail

# Workflow Analysis: Zendesk → WooCommerce Refund Request Handling

## 1. Workflow Overview
This workflow automates the processing of refund requests initiated via Zendesk tickets for a WooCommerce store. It validates if the ticket is related to WooCommerce, verifies the order status and refund history in WooCommerce, and then routes the request based on the refund reason. High-priority "Damaged Item" cases trigger a Slack alert, while other cases trigger automated customer communication via Gmail.

### Logical Blocks
- **1.1 Zendesk Ticket Validation:** Triggering and filtering of incoming tickets to ensure they are WooCommerce-related and extracting key data.
- **1.2 WooCommerce Order Check:** Fetching order details and verifying if the order is eligible for a refund (status check and duplicate refund check).
- **1.3 Refund Reason Evaluation:** Merging ticket and order data to determine the nature of the refund request.
- **1.4 Damaged Item Alert:** Immediate internal notification via Slack for damaged goods.
- **1.5 Refund Processing & Email Handling:** Routing different refund types (Wrong Item, Partial, Full) to specific customer email templates.

---

## 2. Block-by-Block Analysis

### 2.1 Zendesk Ticket Validation
**Overview:** Monitors Zendesk for new tickets and filters for those tagged with 'woocommerce'. It extracts the Order ID and categorizes the refund reason from the ticket description.

- **Nodes Involved:** `Zendesk – New Refund Ticket Trigger`, `If`, `Normalize Zendesk Ticket Data`.
- **Node Details:**
    - **Zendesk – New Refund Ticket Trigger:** Trigger node using OAuth2. Listens for new tickets.
    - **If:** Technical filter. Checks if the expression `{{ $json.ticket.tags.includes('woocommerce') }}` is true.
    - **Normalize Zendesk Ticket Data:** Code node (JavaScript). 
        - *Logic:* Uses Regex to extract Order IDs (e.g., alphanumeric patterns like AB-123) and keyword matching (`damaged`, `wrong`, `partial`, `full`) to assign a `refund_type`.
        - *Output:* A clean JSON object containing `ticket_id`, `customer_email`, `order_id`, and `refund_type`.

### 2.2 WooCommerce Order Check
**Overview:** Retrieves the specific order from WooCommerce and validates whether the order status permits a refund and if a refund has already been processed.

- **Nodes Involved:** `WooCommerce – Fetch Order by ID`, `Normalize WooCommerce Order Data`, `Check Order Status (Completed or Processing)`, `Check If Order Already Refunded`.
- **Node Details:**
    - **WooCommerce – Fetch Order by ID:** Uses the `order_id` from the previous block to GET order details via API.
    - **Normalize WooCommerce Order Data:** Code node. Extracts line items, totals, and calculates `is_refunded` based on the length of the refunds array.
    - **Check Order Status:** IF node. Validates that `order_status` is either `completed` OR `processing`.
    - **Check If Order Already Refunded:** IF node. Validates that `is_refunded` is false (proceeds only if no refund has occurred).

### 2.3 Refund Reason Evaluation
**Overview:** Combines the validated order data with the original ticket data to create a comprehensive dataset for routing.

- **Nodes Involved:** `Merge Ticket and Order Data`, `Is Refund Reason = Damaged Item?`.
- **Node Details:**
    - **Merge Ticket and Order Data:** Merge node (Combine by Position). Joins the normalized Zendesk data and the normalized WooCommerce data.
    - **Is Refund Reason = Damaged Item?:** IF node. Checks if `refund_type` equals `damaged_item`.

### 2.4 Damaged Item Alert
**Overview:** Sends a high-visibility notification to the internal team when a customer reports a damaged product.

- **Nodes Involved:** `Slack – Notify Damaged Item Refund`.
- **Node Details:**
    - **Slack – Notify Damaged Item Refund:** Slack node. Sends a formatted message including Order ID, Product Name, Amount, and Ticket ID to a specified channel.

### 2.5 Refund Processing & Email Handling
**Overview:** Routes non-damaged refund requests to specific communication flows to gather proof or provide instructions.

- **Nodes Involved:** `Route Refund Type (Wrong / Partial / Full)`, `Prepare Email` (3 nodes), `Email Customer` (3 nodes).
- **Node Details:**
    - **Route Refund Type:** Switch node. Routes based on `refund_type`: `wrong_item`, `partial_refund`, or `full_refund`.
    - **Prepare Email Nodes (Set):** Three separate Set nodes that define the `email_subject` and `email_message` based on the scenario:
        - *Wrong Item:* Requests photos of the product and label.
        - *Partial:* Informs the customer the request is under review.
        - *Full:* Informs the customer that a return is required before the refund.
    - **Email Customer Nodes (Gmail):** Gmail nodes that send the prepared content to the `customer_email`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Zendesk – New Refund Ticket Trigger | Zendesk Trigger | Entry Point | - | If | Zendesk Ticket Validation |
| If | If | Filter for WooCommerce tags | Zendesk Trigger | Normalize Zendesk Ticket Data | Zendesk Ticket Validation |
| Normalize Zendesk Ticket Data | Code | Data extraction & cleaning | If | WooCommerce Fetch / Merge | Zendesk Ticket Validation |
| WooCommerce – Fetch Order by ID | WooCommerce | API Data Retrieval | Normalize Zendesk | Normalize WooCommerce Data | WooCommerce Order Check |
| Normalize WooCommerce Order Data | Code | Data normalization | WooCommerce Fetch | Check Order Status | WooCommerce Order Check |
| Check Order Status | If | Eligibility validation | Normalize WooCommerce | Check If Refunded | WooCommerce Order Check |
| Check If Order Already Refunded | If | Duplicate prevention | Check Order Status | Merge Ticket and Order Data | WooCommerce Order Check |
| Merge Ticket and Order Data | Merge | Data aggregation | Normalize Zendesk / Check Refunded | Is Refund Reason = Damaged? | Refund Reason Evaluation |
| Is Refund Reason = Damaged Item? | If | Logic Branching | Merge Ticket and Order Data | Slack Notify / Route Refund | Refund Reason Evaluation |
| Slack – Notify Damaged Item Refund | Slack | Internal Alerting | Is Refund Reason = Damaged? | - | Damaged Item Alert |
| Route Refund Type | Switch | Case Routing | Is Refund Reason = Damaged? | Prepare Email nodes | Refund Processing & Email Handling |
| Prepare Email – Wrong Item... | Set | Template Definition | Route Refund Type | Email Customer (Wrong) | Refund Processing & Email Handling |
| Email Customer – Request Proof... | Gmail | Customer Communication | Prepare Email (Wrong) | - | Refund Processing & Email Handling |
| Prepare Email – Partial Refund... | Set | Template Definition | Route Refund Type | Email Customer (Partial) | Refund Processing & Email Handling |
| Email Customer – Partial... | Gmail | Customer Communication | Prepare Email (Partial) | - | Refund Processing & Email Handling |
| Prepare Email – Full Refund... | Set | Template Definition | Route Refund Type | Email Customer (Full) | Refund Processing & Email Handling |
| Email Customer – Return Required...| Gmail | Customer Communication | Prepare Email (Full) | - | Refund Processing & Email Handling |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Input & Validation
1. Create a **Zendesk Trigger** node. Configure OAuth2 credentials and set it to trigger on new tickets.
2. Add an **If** node. Use the condition: `{{ $json.ticket.tags.includes('woocommerce') }}` is True.
3. Add a **Code** node named "Normalize Zendesk Ticket Data". Paste the JavaScript logic to extract the Order ID (Regex) and identify the `refund_type` (keywords: damaged, wrong, partial, full).

### Step 2: Order Verification
4. Create a **WooCommerce** node. Set Operation to `Get` and Resource to `Order`. Use the expression `{{ $json.order_id }}` for the Order ID.
5. Add a **Code** node to normalize WooCommerce data, specifically mapping line items and determining if the order has already been refunded (`order.refunds.length > 0`).
6. Add an **If** node to check if `order_status` equals `completed` or `processing`.
7. Add another **If** node to ensure `is_refunded` is False.

### Step 3: Logic Routing
8. Add a **Merge** node. Set mode to `Combine` and `Combine by Position`. Connect the "Normalize Zendesk Ticket Data" node and the "Check If Order Already Refunded" node.
9. Add an **If** node to check if `refund_type` equals `damaged_item`.

### Step 4: Actions & Communication
10. **For Damaged Items:** Connect the True branch of the previous If node to a **Slack** node. Configure a channel and a text template using expressions for Order ID and Customer Name.
11. **For Other Items:** Connect the False branch to a **Switch** node. Set rules for `wrong_item`, `partial_refund`, and `full_refund`.
12. **Email Flows:** For each Switch output:
    - Create a **Set** node to define `email_subject`, `email_message`, and `email` (customer email).
    - Connect the Set node to a **Gmail** node. Configure OAuth2 credentials and map the subject, message, and recipient from the Set node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| The workflow requires the `woocommerce` tag in Zendesk to trigger. | Zendesk Configuration |
| Regex used for Order ID extraction is designed for alphanumeric formats (e.g., "US-1234"). | Code Node Logic |
| WooCommerce API credentials must have "Read" permissions for Orders. | API Permissions |
| Gmail OAuth2 is required for automated responses. | Credential Setup |