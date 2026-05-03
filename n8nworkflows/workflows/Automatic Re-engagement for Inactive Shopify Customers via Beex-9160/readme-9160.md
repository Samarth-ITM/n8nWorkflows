Automatic Re-engagement for Inactive Shopify Customers via Beex

https://n8nworkflows.xyz/workflows/automatic-re-engagement-for-inactive-shopify-customers-via-beex-9160


# Automatic Re-engagement for Inactive Shopify Customers via Beex

### 1. Workflow Overview

This workflow automates the re-engagement of inactive Shopify customers by creating leads in the Beex platform. It is designed for e-commerce businesses using Shopify who want to identify customers who have not purchased for a specific period (e.g., over 30 days) and automatically send their data to Beex for targeted marketing campaigns.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Customer Data Retrieval**: Periodically fetch the full customer list from Shopify.
- **1.2 Customers Filtering**: Filter out customers without any purchase history.
- **1.3 Order and Product Data Enrichment**: Retrieve the last order details and product information for each customer.
- **1.4 Data Extraction and Merging**: Extract relevant fields from customers, orders, and products; merge them together.
- **1.5 Inactivity Calculation and Filtering**: Calculate how many days have passed since the last order and filter customers inactive for more than 30 days.
- **1.6 Lead Creation in Beex**: Create new lead entries in Beex for the filtered inactive customers with mapped customer and product data.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Customer Data Retrieval

- **Overview:**  
  Triggers the workflow on a monthly schedule and fetches the complete list of customers from Shopify via an HTTP API request. The customer list is then split into individual records for processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - API Request  
  - Split Records

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Purpose: Initiates workflow monthly (interval set to months).  
    - Configuration: Interval field set to months (default 1 month).  
    - Input: None (trigger node).  
    - Output: Triggers API Request node.  
    - Edge Cases: Misconfigured schedule could lead to no or overly frequent executions.

  - **API Request**  
    - Type: HTTP Request  
    - Purpose: Fetches customer data from Shopify API endpoint (placeholder URL in JSON).  
    - Configuration: URL set to Shopify API endpoint for customers (needs actual store URL).  
    - Input: Trigger from Schedule Trigger.  
    - Output: Sends data to Split Records node.  
    - Edge Cases: API authentication failure, rate limits, network errors, invalid URL.

  - **Split Records**  
    - Type: SplitOut  
    - Purpose: Converts the customer list array into individual customer records for processing.  
    - Configuration: Splits on the field "customers".  
    - Input: API Request node output.  
    - Output: Sends individual customer records to Filter Customers node.  
    - Edge Cases: Empty customer list, unexpected data format.

---

#### 1.2 Customers Filtering

- **Overview:**  
  Filters out customers who have no recorded last order (i.e., inactive or never purchased customers are excluded from further processing at this stage).

- **Nodes Involved:**  
  - Filter Customers

- **Node Details:**

  - **Filter Customers**  
    - Type: Filter  
    - Purpose: Passes only customers with a non-null `last_order_id` field.  
    - Configuration: Condition checks existence of `last_order_id`.  
    - Input: Individual customer records from Split Records.  
    - Output: Passes customers with orders to two parallel nodes: "1. Shopify" (to get last order details) and "Extract Customer Data".  
    - Edge Cases: Misformatted fields, customers without orders get discarded here.

---

#### 1.3 Order and Product Data Enrichment

- **Overview:**  
  Retrieves detailed data about each customer's last order and the first product in that order to enrich the customer profile.

- **Nodes Involved:**  
  - 1. Shopify (Get Last Order)  
  - 2. Shopify (Get Product)  
  - Extract Product Data

- **Node Details:**

  - **1. Shopify (Get Last Order)**  
    - Type: Shopify node  
    - Purpose: Retrieves details of the customer's last order by `last_order_id`.  
    - Configuration: Operation "get" on order resource, with authentication via Shopify Access Token.  
    - Input: Filter Customers output.  
    - Output: Passes order data to "2. Shopify".  
    - Edge Cases: Invalid order ID, expired token, API limits.

  - **2. Shopify (Get Product)**  
    - Type: Shopify node  
    - Purpose: Retrieves the product details of the first item in the last order.  
    - Configuration: Operation "get" on product resource, with product ID from order line items.  
    - Input: Output of "1. Shopify".  
    - Output: Passes product data to "Extract Product Data".  
    - Edge Cases: Missing product ID in order, API errors.

  - **Extract Product Data**  
    - Type: Set  
    - Purpose: Extracts relevant product fields such as `product_type` and `closed_at` (order closing date) for downstream use.  
    - Configuration: Assigns `closed_at` from last order data and `product_type` from product info.  
    - Input: "2. Shopify" node output.  
    - Output: Passes data merged later with customer data.  
    - Edge Cases: Missing or null fields in product or order data.

---

#### 1.4 Data Extraction and Merging

- **Overview:**  
  Extracts relevant customer fields and merges customer data with order and product data to prepare a complete data record for inactivity evaluation.

- **Nodes Involved:**  
  - Extract Customer Data  
  - Merge Data

- **Node Details:**

  - **Extract Customer Data**  
    - Type: Set  
    - Purpose: Extracts and maps customer fields: `client_id`, `first_name`, `last_name`, `email`, `wsp` (phone).  
    - Configuration: Assigns fields directly from customer JSON.  
    - Input: Output of Filter Customers.  
    - Output: Passed as second input to Merge Data node.  
    - Edge Cases: Missing customer fields, type mismatches.

  - **Merge Data**  
    - Type: Merge  
    - Purpose: Combines extracted product/order data (first input) with extracted customer data (second input) by position to align corresponding records.  
    - Configuration: Mode set to "combine" by position.  
    - Input: From Extract Product Data (first input), Extract Customer Data (second input).  
    - Output: Passes merged data to Calculate Days node.  
    - Edge Cases: Mismatched record counts causing misaligned merges.

---

#### 1.5 Inactivity Calculation and Filtering

- **Overview:**  
  Calculates the number of days since the customer's last purchase and filters customers inactive for more than 30 days.

- **Nodes Involved:**  
  - Calculate Days  
  - Filter by Days

- **Node Details:**

  - **Calculate Days**  
    - Type: DateTime  
    - Purpose: Calculates the difference in days between the `closed_at` date of the last order and current date/time.  
    - Configuration: Operation "getTimeBetweenDates" with startDate = last order close date, endDate = now.  
    - Input: Merged customer and product/order data.  
    - Output: Passes data with appended `timeDifference.days` field to Filter by Days.  
    - Edge Cases: Invalid or missing date formats, time zone issues.

  - **Filter by Days**  
    - Type: Filter  
    - Purpose: Passes only customers with `timeDifference.days` > 30 (inactive customers).  
    - Configuration: Condition for numeric greater than 30 on `timeDifference.days`.  
    - Input: Calculate Days output.  
    - Output: Passes filtered inactive customers to Create Lead node.  
    - Edge Cases: Edge cases around exactly 30 days, null values.

---

#### 1.6 Lead Creation in Beex

- **Overview:**  
  Creates a lead in Beex for each inactive customer, mapping relevant customer and product data fields to the lead resource parameters.

- **Nodes Involved:**  
  - Create Lead

- **Node Details:**

  - **Create Lead**  
    - Type: Beex (community node)  
    - Purpose: Creates a new lead in Beex for re-engagement campaigns.  
    - Configuration:  
      - Maps email, client ID, phone number (split into country code and number), portfolio ID, sequence ID, priority, and additional parameters including last order date, product type, and customer names.  
      - Uses fields sliced from `wsp` phone string to separate country code and phone number.  
    - Input: Filter by Days output (inactive customers).  
    - Output: None (final node).  
    - Edge Cases: API authentication failure, invalid phone or email formats, Beex lead creation errors, missing required fields.  
    - Requirements: Only available on self-hosted n8n instances with Beex node installed. Requires Beex credentials with lead creation permissions.

---

### 3. Summary Table

| Node Name          | Node Type              | Functional Role                         | Input Node(s)          | Output Node(s)              | Sticky Note                                                                                          |
|--------------------|------------------------|---------------------------------------|------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger   | Schedule Trigger        | Monthly trigger to start workflow     | -                      | API Request                 |                                                                                                |
| API Request        | HTTP Request            | Fetch Shopify customers list          | Schedule Trigger       | Split Records               | ## Get the Customer List from Shopify - Extracts the customer list for a specific Shopify store via API on a monthly basis. - Splits the records to create a tabular format. |
| Split Records      | SplitOut                | Split customer array into single items| API Request            | Filter Customers            |                                                                                                |
| Filter Customers   | Filter                  | Filter customers with last orders only| Split Records          | 1. Shopify, Extract Customer Data | ## Filter Customers - We discard customers with a null **last_order_id** field.                      |
| 1. Shopify         | Shopify                 | Get last order details per customer   | Filter Customers       | 2. Shopify                 | ## Extract Features from the Last Order - We must consider the closing date of the last order. - The type of product ordered in the order is also considered. |
| 2. Shopify         | Shopify                 | Get product details from last order   | 1. Shopify             | Extract Product Data        |                                                                                                |
| Extract Product Data | Set                    | Extract product type and order close date | 2. Shopify           | Merge Data                 | ## Extract Data Product                                                                            |
| Extract Customer Data | Set                   | Extract customer fields for merging   | Filter Customers       | Merge Data                 |                                                                                                |
| Merge Data         | Merge                   | Combine customer and product/order data | Extract Product Data, Extract Customer Data | Calculate Days           | ## Attach Along with the Relevant Customer Data - Relevant customer data is merged with data extracted from the order and product. - Combine By Position |
| Calculate Days     | DateTime                | Calculate days since last order       | Merge Data             | Filter by Days             | ## Calculate Customer Inactivity Days - We obtain the number of days elapsed from the date of the last purchase until today. We filter according to what is obtained |
| Filter by Days     | Filter                  | Filter customers inactive > 30 days   | Calculate Days         | Create Lead                |                                                                                                |
| Create Lead        | Beex                    | Create lead in Beex for inactive customers | Filter by Days      | -                          | ## Capture Lead in Beex - A sequence of steps is prepared for contact.                             |
| Sticky Note        | Sticky Note             | Informational notes                    | -                      | -                          | Covers multiple nodes as per above descriptions                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Node Type: Schedule Trigger  
   - Configuration: Set interval to 1 month to run workflow monthly.  
   - Output connected to API Request node.

2. **Add HTTP Request Node (API Request)**  
   - Node Type: HTTP Request  
   - Configuration:  
     - URL: Shopify API endpoint for customers (replace placeholder "http://shopify.com" with actual Shopify customers API URL, e.g. `https://{shop}.myshopify.com/admin/api/2023-01/customers.json`).  
     - Authentication: Configure with Shopify API token or OAuth as needed (not shown in original node but required).  
   - Connect input from Schedule Trigger.  
   - Output connected to Split Records node.

3. **Add SplitOut Node (Split Records)**  
   - Node Type: SplitOut  
   - Configuration: Set field to split out to "customers" (the array field containing customer objects).  
   - Input from API Request.  
   - Output to Filter Customers.

4. **Add Filter Node (Filter Customers)**  
   - Node Type: Filter  
   - Configuration:  
     - Condition: `last_order_id` exists (non-null) to keep customers with purchase history.  
   - Input from Split Records.  
   - Output to two nodes:  
     - 1. Shopify (Get Last Order)  
     - Extract Customer Data.

5. **Add Shopify Node (1. Shopify)**  
   - Node Type: Shopify  
   - Configuration:  
     - Operation: Get order.  
     - Order ID: Expression `{{$json["last_order_id"]}}`.  
     - Authentication: Use Shopify Access Token credential.  
   - Input from Filter Customers.  
   - Output to 2. Shopify node.

6. **Add Shopify Node (2. Shopify)**  
   - Node Type: Shopify  
   - Configuration:  
     - Resource: Product  
     - Operation: Get product.  
     - Product ID: Expression `{{$json["line_items"][0]["product_id"]}}`.  
     - Authentication: Use same Shopify Access Token credential.  
   - Input from 1. Shopify node.  
   - Output to Extract Product Data.

7. **Add Set Node (Extract Product Data)**  
   - Node Type: Set  
   - Configuration: Assign fields:  
     - `closed_at` = `{{$node["1. Shopify"].json["closed_at"]}}`  
     - `product_type` = `{{$json["product_type"]}}`  
   - Input from 2. Shopify node.  
   - Output to Merge Data node (first input).

8. **Add Set Node (Extract Customer Data)**  
   - Node Type: Set  
   - Configuration: Assign fields from customer JSON:  
     - `client_id` = `{{$json["id"]}}`  
     - `first_name` = `{{$json["first_name"]}}`  
     - `last_name` = `{{$json["last_name"]}}`  
     - `email` = `{{$json["email"]}}`  
     - `wsp` = `{{$json["phone"]}}`  
   - Input from Filter Customers node.  
   - Output to Merge Data node (second input).

9. **Add Merge Node (Merge Data)**  
   - Node Type: Merge  
   - Configuration:  
     - Mode: Combine  
     - Combine By: Position (to align records by order).  
   - Inputs from Extract Product Data (first) and Extract Customer Data (second).  
   - Output to Calculate Days node.

10. **Add DateTime Node (Calculate Days)**  
    - Node Type: DateTime  
    - Configuration:  
      - Operation: Get time between dates.  
      - Start Date: `{{$json["closed_at"]}}` (last order close date).  
      - End Date: `{{$now}}` (current date).  
      - Include input fields: true (to preserve data).  
    - Input from Merge Data.  
    - Output to Filter by Days node.

11. **Add Filter Node (Filter by Days)**  
    - Node Type: Filter  
    - Configuration:  
      - Condition: `{{$json["timeDifference"]["days"]}} > 30` (customers inactive more than 30 days).  
    - Input from Calculate Days.  
    - Output to Create Lead node.

12. **Add Beex Node (Create Lead)**  
    - Node Type: Beex (community node)  
    - Configuration:  
      - Resource: Leads  
      - Parameters mapping:  
        - `email` = `{{$json["email"]}}`  
        - `priority` = 1  
        - `code_client` = `{{$json["client_id"]}}`  
        - `sequence_id` = 22 (predefined sequence in Beex)  
        - `code_country` = `{{$json["wsp"].slice(0,3)}}` (country code from phone)  
        - `phone_number` = `{{$json["wsp"].slice(3)}}` (phone number without country code)  
        - `portfolio_id` = 81  
        - Additional parameters:  
          - `text_01` = last order date sliced to YYYY-MM-DD  
          - `text_02` = product type  
          - `text_03` = "Inactive" (status label)  
          - `first_name` and `paternal_surname` from customer data  
    - Input from Filter by Days.  
    - Credentials: Configure with Beex API credentials including Bearer Token.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses the community Beex node (`n8n-nodes-beex`) which is only available on self-hosted n8n instances. Ensure your environment supports this node before deploying.                                                                                                                                                                                                       | Workflow disclaimer and requirement.                                                                                                  |
| Workflow visual preview and explanation: ![Workflow preview](https://i.imgur.com/grICvWZ.png)                                                                                                                                                                                                                                                                                           | Workflow diagram link.                                                                                                                |
| The inactivity threshold (currently 30 days) can be adjusted in the "Filter by Days" node to match specific business rules.                                                                                                                                                                                                                                                           | Allows customization based on marketing needs.                                                                                        |
| Mapping phone number slices assumes the phone number format starts with a 3-digit country code followed by the local number; adjust slicing if your data format differs.                                                                                                                                                                                                               | Important for accurate phone data parsing for Beex lead creation.                                                                     |
| Requires Shopify API credentials with permission to read customers, orders, and products; and Beex API credentials with permission to create leads.                                                                                                                                                                                                                                    | Credential setup note.                                                                                                                |
| For large customer datasets, consider Shopify API rate limits and pagination handling which are not explicitly implemented here.                                                                                                                                                                                                                                                      | Potential scalability note.                                                                                                          |
| The workflow is designed for monthly execution but can be adapted to other schedules.                                                                                                                                                                                                                                                                                                    | Scheduling flexibility.                                                                                                              |

---

*Disclaimer: The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled are legal and public.*