Automated Inventory Management with Airtable PO Creation & Supplier Emails

https://n8nworkflows.xyz/workflows/automated-inventory-management-with-airtable-po-creation---supplier-emails-5815


# Automated Inventory Management with Airtable PO Creation & Supplier Emails

### 1. Workflow Overview

This workflow automates inventory management by monitoring product stock levels in Airtable, calculating intelligent reorder quantities, creating purchase orders (POs), and emailing suppliers automatically. It runs daily with two distinct scheduled triggers:

- **Midnight Trigger (00:00)**: Checks products with low stock, calculates reorder quantities, filters out duplicates, and creates new purchase order records in Airtable.
- **Early Morning Trigger (01:00)**: Fetches all pending purchase orders, groups them by supplier, emails detailed purchase orders to each supplier, and updates the PO status to "Sent."

Logical blocks:

- **1.1 Scheduled Triggers**: Two schedule triggers at midnight and 1 AM to start the respective processes.
- **1.2 Low Stock Product Identification & PO Creation**: Nodes that retrieve low-stock products, get supplier details, calculate reorder quantities, check existing POs to avoid duplicates, and create new purchase order records.
- **1.3 Purchase Order Emailing & Status Update**: Nodes that fetch pending POs, group them by supplier, construct and send purchase order emails using Brevo (SendInBlue), and update PO statuses in Airtable.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Triggers

**Overview:**  
Defines the two daily schedule triggers to initiate inventory checking and supplier communication workflows.

**Nodes Involved:**  
- Run Every Mid Night (Schedule Trigger)  
- Schedule Trigger (Schedule Trigger)

**Node Details:**

- **Run Every Mid Night**  
  - Type: Schedule Trigger  
  - Configuration: Cron expression `0 0 * * *` (every day at midnight)  
  - Outputs to: Get Products with low stock  
  - Edge Cases: Cron misconfiguration or n8n service downtime may prevent trigger firing.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Cron expression `0 1 * * *` (every day at 1 AM)  
  - Outputs to: Get Purchase Orders which are pending  
  - Edge Cases: Same as above.

---

#### 1.2 Low Stock Product Identification & PO Creation

**Overview:**  
Identifies products with stock levels below reorder thresholds, calculates reorder quantities dynamically, avoids duplicate orders, and creates new purchase order records in Airtable.

**Nodes Involved:**  
- Get Products with low stock (Airtable)  
- Get supplier Details for those low stock products (Airtable)  
- Calculate Dynamic Re-order Quantity (Code)  
- Search records (Airtable)  
- Remove Duplicate Product Orders (Code)  
- Create purchase records in table (Airtable)

**Node Details:**

- **Get Products with low stock**  
  - Type: Airtable node (search operation)  
  - Config: Queries "Products Table" for records where `{stock_level} <= {reorder_threshold}`  
  - Input: Triggered by midnight schedule  
  - Output: List of low-stock products  
  - Edge Cases: Airtable API quota limits, incorrect filter formula, missing stock_level or reorder_threshold fields.

- **Get supplier Details for those low stock products**  
  - Type: Airtable node (get by record ID)  
  - Config: Uses dynamic record ID from product’s `supplier_id` field to fetch supplier info from "Supplier" table  
  - Input: From "Get Products with low stock"  
  - Output: Supplier details corresponding to each product  
  - Edge Cases: Missing or malformed `supplier_id`, API errors.

- **Calculate Dynamic Re-order Quantity**  
  - Type: Code node (JavaScript)  
  - Logic: Calculates reorder quantity using formula:  
    `reorder_qty = ceil(average_daily_sales × lead_time_days × 1.5 × safetyMargin(1.2))`  
  - Input: Supplier-enriched product data  
  - Output: Product data enriched with `reorder_qty`  
  - Edge Cases: Missing average_daily_sales or lead_time_days defaults to fallback values; code errors.

- **Search records**  
  - Type: Airtable node (search)  
  - Config: Searches "Purchase Orders" table for existing POs with matching `product_id` and status "Pending" or "Sent" to avoid duplicates  
  - Input: From reorder quantity calculation  
  - Output: Existing POs for filtering  
  - Edge Cases: Formula evaluation errors, API limits.

- **Remove Duplicate Product Orders**  
  - Type: Code node  
  - Logic:  
    - Collects all low stock products, reorder quantities, supplier details, and existing POs  
    - Filters out products which already have active POs (Pending or Sent)  
    - Builds a list of new purchase order data with product ID, name, reorder quantity, supplier ID, and supplier email  
  - Input: From Search records node (existing POs)  
  - Output: Filtered list of products to reorder  
  - Edge Cases: Missing or mismatched IDs, empty input sets.

- **Create purchase records in table**  
  - Type: Airtable node (create operation)  
  - Config: Creates new records in "Purchase Orders" table with fields: status="Pending", quantity=reorder_qty, product_id, supplier_id, product_name, supplier_email  
  - Input: Filtered new POs data  
  - Output: Created PO records in Airtable  
  - Edge Cases: Airtable API errors, data validation errors.

---

#### 1.3 Purchase Order Emailing & Status Update

**Overview:**  
Fetches pending purchase orders, groups them by supplier, sends detailed purchase order emails via Brevo (SendInBlue), and updates the status of those POs to "Sent" in Airtable.

**Nodes Involved:**  
- Get Purchase Orders which are pending (Airtable)  
- Group products with suppliers (Code)  
- Send PO to suppliers via Email (SendInBlue)  
- Get the record id's of PO (Code)  
- Update the PO status to sent (Airtable)

**Node Details:**

- **Get Purchase Orders which are pending**  
  - Type: Airtable node (search)  
  - Config: Queries "Purchase Orders" table for records where `{status} = 'Pending'`  
  - Input: Triggered by 1 AM schedule  
  - Output: List of pending POs  
  - Edge Cases: API errors, unexpected record statuses.

- **Group products with suppliers**  
  - Type: Code node  
  - Logic:  
    - Groups purchase orders by supplier_id and supplier_email  
    - Aggregates order details (product name, quantity, product ID) per supplier  
    - Creates HTML and plain text email bodies summarizing the purchase order with a styled table and totals  
    - Prepares email subject lines and records PO IDs for later status updates  
  - Input: Pending POs  
  - Output: Supplier-grouped email data including email content and record IDs  
  - Edge Cases: Missing supplier emails, empty order details.

- **Send PO to suppliers via Email**  
  - Type: SendInBlue (Brevo) node  
  - Config:  
    - Sender email: `abhiram.bvb@gmail.com`  
    - Dynamic subject: from grouped data  
    - HTML email body: from grouped data  
    - Recipient: supplier_email from grouped data  
  - Input: Grouped supplier email data  
  - Output: Sent email confirmation  
  - Credential: Brevo API credentials required  
  - Edge Cases: Email sending failures, invalid email addresses, API quota limits.

- **Get the record id's of PO**  
  - Type: Code node  
  - Logic: Extracts the Airtable record IDs of POs sent to suppliers and prepares update objects with status "Sent"  
  - Input: From Send PO node output  
  - Output: List of PO records to update  
  - Edge Cases: Missing or malformed record IDs.

- **Update the PO status to sent**  
  - Type: Airtable node (upsert)  
  - Config: Updates PO records in "Purchase Orders" table by matching record ID; sets status to "Sent"  
  - Input: PO record updates from prior code node  
  - Output: Updated PO records  
  - Edge Cases: Airtable API errors, race conditions if multiple updates occur simultaneously.

---

### 3. Summary Table

| Node Name                            | Node Type             | Functional Role                                    | Input Node(s)                       | Output Node(s)                               | Sticky Note                                                                                      |
|------------------------------------|-----------------------|---------------------------------------------------|-----------------------------------|----------------------------------------------|------------------------------------------------------------------------------------------------|
| Run Every Mid Night                 | Schedule Trigger      | Initiates low stock check & reorder process daily | -                                 | Get Products with low stock                    | See Sticky Note: Automated Purchase Order Creation – n8n Workflow (block 1.2)                   |
| Get Products with low stock         | Airtable              | Fetches low stock products from Airtable          | Run Every Mid Night                | Get supplier Details for those low stock products | See Sticky Note: Automated Purchase Order Creation – n8n Workflow (block 1.2)                   |
| Get supplier Details for those low stock products | Airtable              | Fetches supplier details for each low-stock product | Get Products with low stock         | Calculate Dynamic Re-order Quantity            | See Sticky Note: Automated Purchase Order Creation – n8n Workflow (block 1.2)                   |
| Calculate Dynamic Re-order Quantity | Code                  | Calculates reorder quantities based on sales & lead time | Get supplier Details for those low stock products | Search records                                 | See Sticky Note: Automated Purchase Order Creation – n8n Workflow (block 1.2)                   |
| Search records                     | Airtable              | Searches for existing POs to avoid duplicates      | Calculate Dynamic Re-order Quantity | Remove Duplicate Product Orders                 | See Sticky Note: Automated Purchase Order Creation – n8n Workflow (block 1.2)                   |
| Remove Duplicate Product Orders    | Code                  | Filters products with existing active POs          | Search records                    | Create purchase records in table               | See Sticky Note: Automated Purchase Order Creation – n8n Workflow (block 1.2)                   |
| Create purchase records in table   | Airtable              | Creates new purchase order records                  | Remove Duplicate Product Orders    | -                                              | See Sticky Note: Automated Purchase Order Creation – n8n Workflow (block 1.2)                   |
| Schedule Trigger                   | Schedule Trigger      | Starts daily PO email & status update process       | -                                 | Get Purchase Orders which are pending           | See Sticky Note1: Daily PO Email & Status Sync (block 1.3)                                      |
| Get Purchase Orders which are pending | Airtable              | Fetches all pending purchase orders                 | Schedule Trigger                  | Group products with suppliers                   | See Sticky Note1: Daily PO Email & Status Sync (block 1.3)                                      |
| Group products with suppliers      | Code                  | Groups POs by supplier, builds email content        | Get Purchase Orders which are pending | Send PO to suppliers via Email, Get the record id's of PO | See Sticky Note1: Daily PO Email & Status Sync (block 1.3)                                      |
| Send PO to suppliers via Email     | SendInBlue (Brevo)    | Sends purchase order emails to suppliers            | Group products with suppliers      | -                                              | See Sticky Note1: Daily PO Email & Status Sync (block 1.3)                                      |
| Get the record id's of PO          | Code                  | Prepares PO records for status update                | Send PO to suppliers via Email     | Update the PO status to sent                    | See Sticky Note1: Daily PO Email & Status Sync (block 1.3)                                      |
| Update the PO status to sent       | Airtable              | Updates PO statuses to "Sent"                        | Get the record id's of PO          | -                                              | See Sticky Note1: Daily PO Email & Status Sync (block 1.3)                                      |
| Sticky Note                       | Sticky Note           | Describes the automated purchase order creation     | -                                 | -                                              | 📦 Automated Purchase Order Creation – n8n Workflow (blocks 1.1 & 1.2)                          |
| Sticky Note1                      | Sticky Note           | Describes daily PO email and status synchronization | -                                 | -                                              | 📦 Daily PO Email & Status Sync (block 1.3)                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Run Every Mid Night" Schedule Trigger**  
   - Node Type: Schedule Trigger  
   - Configuration: Cron expression `0 0 * * *` (daily at midnight)

2. **Add "Get Products with low stock" Airtable Node**  
   - Operation: Search  
   - Base: Select your Airtable base containing product data  
   - Table: Products Table  
   - Filter formula: `{stock_level} <= {reorder_threshold}`  
   - Credentials: Airtable Personal Access Token  
   - Connect from "Run Every Mid Night"

3. **Add "Get supplier Details for those low stock products" Airtable Node**  
   - Operation: Get Record by ID  
   - Base: Same Airtable base  
   - Table: Supplier Table  
   - Record ID: Set expression `={{ $json.supplier_id }}` dynamically from product data  
   - Credentials: Same Airtable token  
   - Connect from "Get Products with low stock"

4. **Add "Calculate Dynamic Re-order Quantity" Code Node**  
   - Language: JavaScript  
   - Code: Use provided calculation logic (average daily sales × lead time × 1.5 × 1.2)  
   - Input: From "Get supplier Details for those low stock products"  
   - Output: Adds `reorder_qty` field on each product

5. **Add "Search records" Airtable Node**  
   - Operation: Search  
   - Base and Table: Purchase Orders in Airtable  
   - Filter formula:  
     ```plaintext
     =AND({product_id} = '{{ $('Get Products with low stock').item.json.product_id }}', OR({status} = 'Pending', {status} = 'Sent'))
     ```  
   - Credentials: Airtable token  
   - Connect from "Calculate Dynamic Re-order Quantity"

6. **Add "Remove Duplicate Product Orders" Code Node**  
   - JavaScript: Use provided code to filter out products with existing POs and enrich with supplier emails  
   - Input: From "Search records"

7. **Add "Create purchase records in table" Airtable Node**  
   - Operation: Create  
   - Base and Table: Purchase Orders table  
   - Map fields:  
     - status: "Pending"  
     - quantity: `={{ $json.reorder_qty }}`  
     - product_id, supplier_id, product_name, supplier_email from JSON data  
   - Credentials: Airtable token  
   - Connect from "Remove Duplicate Product Orders"

8. **Create the "Schedule Trigger" for 1 AM**  
   - Node Type: Schedule Trigger  
   - Configuration: Cron expression `0 1 * * *`

9. **Add "Get Purchase Orders which are pending" Airtable Node**  
   - Operation: Search  
   - Base and Table: Purchase Orders  
   - Filter formula: `{status} = 'Pending'`  
   - Credentials: Airtable token  
   - Connect from 1 AM Schedule Trigger

10. **Add "Group products with suppliers" Code Node**  
    - JavaScript: Group POs by supplier, create HTML and text email content, prepare email subjects, and collect record IDs  
    - Input: From "Get Purchase Orders which are pending"

11. **Add "Send PO to suppliers via Email" SendInBlue Node**  
    - Sender: `abhiram.bvb@gmail.com` (or your verified sender email)  
    - Subject: `={{ $json.email_subject }}`  
    - HTML Content: `={{ $json.email_body_html }}`  
    - Recipients: `={{ $json.supplier_email }}`  
    - Credentials: Brevo (SendInBlue) API key  
    - Connect from "Group products with suppliers"

12. **Add "Get the record id's of PO" Code Node**  
    - JavaScript: Extract record IDs from previous node output and prepare updates with status "Sent"  
    - Input: From "Send PO to suppliers via Email"

13. **Add "Update the PO status to sent" Airtable Node**  
    - Operation: Upsert  
    - Base and Table: Purchase Orders  
    - Match by: Record ID  
    - Fields to update: status = "Sent"  
    - Credentials: Airtable token  
    - Connect from "Get the record id's of PO"

14. **Optional: Add Sticky Note nodes** for documentation inside the workflow to describe the two major process blocks.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| 📦 **Automated Purchase Order Creation – n8n Workflow**: The workflow runs nightly to automatically generate purchase orders for low stock items, calculate reorder quantities using average sales and lead time, avoid duplicate POs, and create records in Airtable.                                                                                                                                                                      | Sticky Note at workflow start block (nodes 7d04d71d, fe401179, etc.)                                                  |
| 📦 **Daily PO Email & Status Sync**: Runs at 1 AM to fetch pending purchase orders, group them by supplier, send email summaries via Brevo (SendInBlue), and update PO statuses to "Sent" in Airtable. Keeps suppliers informed and automates the ordering process.                                                                                                                                                                      | Sticky Note at workflow second block (nodes 790b9e2e, 93b3ea8a, 4ba6d12d, etc.)                                      |
| Airtable API Personal Access Token and Brevo (SendInBlue) API credentials are required and must be configured in n8n credential manager.                                                                                                                                                                                                                                                                                                 | Credential setup                                                                                                      |
| The formula used in "Search records" node filters existing POs to prevent duplicate orders by checking product ID and status. This critical step ensures inventory ordering is efficient and avoids duplicate stock orders.                                                                                                                                                                                                                      | Important formula detail                                                                                              |
| Email content is generated as both HTML and plain text for compatibility, including a styled table listing product names, IDs, and quantities, with total quantities and supplier info for clarity.                                                                                                                                                                                                                                       | Email content details                                                                                                |
| Safety margin and lead time multipliers in reorder quantity calculation can be adjusted in the code node to better fit specific inventory management policies.                                                                                                                                                                                                                                                                           | Customization note                                                                                                   |
| The workflow assumes the Airtable base has tables named "Products Table", "Supplier", and "Purchase Orders" with matching field names as used in the nodes. Ensure your Airtable schema matches these.                                                                                                                                                                                                                                     | Airtable schema requirements                                                                                         |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.