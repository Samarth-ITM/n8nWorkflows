Sync Shopify and WooCommerce sales into a Google Sheets accounting ledger

https://n8nworkflows.xyz/workflows/sync-shopify-and-woocommerce-sales-into-a-google-sheets-accounting-ledger-13829


# Sync Shopify and WooCommerce sales into a Google Sheets accounting ledger

### 1. Workflow Overview

This workflow automates the synchronization of retail transactions from two major e-commerce platforms—**Shopify** and **WooCommerce**—into a centralized **Google Sheets** accounting ledger. It acts as a middleware that listens for new orders, normalizes disparate data structures into a single schema, distinguishes between sales and potential refunds, and formats the data into accounting-ready entries.

**Logical Blocks:**
*   **1.1 Data Acquisition & Initial Normalization:** Captures real-time webhooks from Shopify and WooCommerce and maps platform-specific keys (e.g., `current_total_price` vs `total`) to a standardized internal format.
*   **1.2 Consolidation & Refinement:** Merges the two data streams and ensures all transaction metadata (customer details, order numbers, and sources) are uniformly prepared.
*   **1.3 Business Logic & Routing:** Evaluates the nature of the transaction to route data through specific formatting logic for sales or refunds.
*   **1.4 Accounting Preparation & Logging:** Converts the data into specialized ledger structures (including line items and currency references) and appends it to a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition & Initial Normalization
This block serves as the entry point for the workflow, ensuring that no matter where the sale originates, the data looks identical for the rest of the process.

*   **Nodes Involved:** `Shopify Order Events`, `Shopify Configuration`, `WooCommerce Order Events`, `WooCommerce Configuration`.
*   **Node Details:**
    *   **Shopify Order Events (Trigger):** Listens for `orders/create`. Requires an Access Token with order read permissions.
    *   **WooCommerce Order Events (Trigger):** Listens for `order.created`. Requires REST API credentials.
    *   **Configuration Nodes (Set):** These nodes map specific JSON paths to a unified schema. For example:
        *   Shopify's `current_total_price` becomes `totalAmount`.
        *   WooCommerce's `billing.email` becomes `customerEmail`.
        *   A hardcoded `sourceSystem` value ('shopify' or 'woocommerce') is added for tracking.
    *   **Potential Failures:** Webhook secret mismatches, authentication expiration, or schema changes in platform API updates.

#### 2.2 Consolidation & Refinement
This block gathers the standardized inputs into a single execution path.

*   **Nodes Involved:** `Merge Platform Data`, `Format Transaction Data`.
*   **Node Details:**
    *   **Merge Platform Data (Merge):** Combines the outputs of both configuration nodes. Since it uses Version 3.2, it passes through data as it arrives from either branch.
    *   **Format Transaction Data (Set):** Acts as a final "cleanup" node. It constructs a `description` field by concatenating the order number and source system and ensures variables like `transactionType` (defaulted to 'sale') are present.
    *   **Key Expression:** `{{ 'Order #' + $json.orderNumber + ' from ' + $json.sourceSystem }}`.

#### 2.3 Business Logic & Routing
This block determines the accounting treatment required for the incoming data.

*   **Nodes Involved:** `Check Transaction Type`, `Format Sale Entry`, `Format Refund Entry`.
*   **Node Details:**
    *   **Check Transaction Type (If):** Filters data based on the `transactionType` property. 
        *   **True branch:** Directed to Sale formatting.
        *   **False branch:** Directed to Refund formatting.
    *   **Format Sale/Refund Entry (Set):** These nodes prepare the JSON structure specifically for spreadsheet columns. 
        *   **Sale:** Creates a stringified JSON array for the `Line` column representing a "SalesItemLineDetail".
        *   **Refund:** Uses `Math.abs($json.amount)` to ensure refund values are represented correctly and prefixes the document number with "REF-".

#### 2.4 Data Persistence
The final stage where data is committed to the financial record.

*   **Nodes Involved:** `Log to Google Sheets`.
*   **Node Details:**
    *   **Type:** Google Sheets (Version 4.7).
    *   **Operation:** `appendOrUpdate`. This is critical for preventing duplicate entries if a webhook is retried.
    *   **Configuration:** Maps the keys created in the previous block (`TxnDate`, `CustomerRef`, `TotalAmt`, etc.) to matching columns in the spreadsheet.
    *   **Failure Modes:** Sheet permissions (403), "Document ID not found" (404), or column header mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Shopify Order Events | Shopify Trigger | Trigger (Order Created) | None | Shopify Configuration | The Shopify and WooCommerce Order Events nodes serve as the workflow's entry points... |
| WooCommerce Order Events | WooCommerce Trigger | Trigger (Order Created) | None | WooCommerce Configuration | The Shopify and WooCommerce Order Events nodes serve as the workflow's entry points... |
| Shopify Configuration | Set | Data Normalization | Shopify Order Events | Merge Platform Data | Subsequent Shopify and WooCommerce Configuration nodes then normalize these native JSON payloads... |
| WooCommerce Configuration | Set | Data Normalization | WooCommerce Order Events | Merge Platform Data | Subsequent Shopify and WooCommerce Configuration nodes then normalize these native JSON payloads... |
| Merge Platform Data | Merge | Stream Consolidation | Shopify/WooCommerce Config | Format Transaction Data | The Merge Platform Data node consolidates incoming streams from different e-commerce sources into a single workflow path. |
| Format Transaction Data | Set | Final Refinement | Merge Platform Data | Check Transaction Type | The subsequent Format Transaction Data node then maps various platform attributes... into a unified schema. |
| Check Transaction Type | If | Logic Router | Format Transaction Data | Format Sale/Refund Entry | The final Check Transaction Type node acts as a router, evaluating the transactionType field... |
| Format Sale Entry | Set | Accounting Formatting | Check Transaction Type | Log to Google Sheets | The Format Sale and Format Refund Entry nodes prepare the normalized data for accounting by structuring it into financial records. |
| Format Refund Entry | Set | Accounting Formatting | Check Transaction Type | Log to Google Sheets | They calculate absolute values for amounts, generate transaction dates, and create detailed line items... |
| Log to Google Sheets | Google Sheets | Data Persistence | Format Sale/Refund Entry | None | The final Log to Google Sheets node records these formatted entries into a centralized spreadsheet. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Shopify Trigger** node. Set the topic to `orders/create`. Use Access Token authentication.
    *   Create a **WooCommerce Trigger** node. Set the event to `order.created`.
2.  **Initial Schema Mapping:**
    *   Add a **Set** node after Shopify. Create fields for `orderId`, `orderNumber`, `totalAmount` (using `current_total_price`), `customerEmail`, and `sourceSystem` (value: `shopify`).
    *   Add a **Set** node after WooCommerce. Create identical fields, mapping WooCommerce's `total` to `totalAmount` and `billing.email` to `customerEmail`. Set `sourceSystem` to `woocommerce`.
3.  **Consolidation:**
    *   Connect both Set nodes to a **Merge** node (Mode: Append).
    *   Connect the Merge node to a new **Set** node named "Format Transaction Data". Use expressions to create a `description` field: `Order #{{$json.orderNumber}} from {{$json.sourceSystem}}`.
4.  **Routing Logic:**
    *   Add an **If** node. Set the condition to: `{{ $json.transactionType }}` equals `sale`.
5.  **Accounting Formatting:**
    *   **Sale Branch:** Add a **Set** node. Create a field `TxnDate` using `{{ $json.date.split('T')[0] }}`. Create a `Line` field containing a stringified JSON array of the item details.
    *   **Refund Branch:** Add a **Set** node. Use `Math.abs()` on the `amount` field to normalize refund values. Prefix the `DocNumber` with `REF-`.
6.  **Google Sheets Integration:**
    *   Add a **Google Sheets** node. Select the `Append or Update` operation.
    *   Provide the **Document ID** and **Sheet Name**.
    *   Under "Mapping Mode", select `Auto Map Input Data` and ensure your spreadsheet headers match the keys created in Step 5 (`TxnDate`, `CustomerRef`, `TotalAmt`, `DocNumber`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Dependency** | Webhook URLs must be manually registered in Shopify/WooCommerce admin panels if not using n8n's automatic registration. |
| **Permissions** | The Google Account must have "Editor" access to the target Spreadsheet. |
| **Audit Trail** | The `PrivateNote` field is designed to hold the original Platform Order ID for cross-referencing during audits. |
| **Data Integrity** | The `appendOrUpdate` operation in Google Sheets requires at least one column to be designated as a "Key" to prevent duplicates. |