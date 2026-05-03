Complete Shopify to Odoo Integration: Sync Orders, Products & Customers

https://n8nworkflows.xyz/workflows/complete-shopify-to-odoo-integration--sync-orders--products---customers-4069


# Complete Shopify to Odoo Integration: Sync Orders, Products & Customers

### 1. Workflow Overview

This workflow provides a complete integration between Shopify and Odoo, synchronizing Orders, Products, and Customers from Shopify into Odoo ERP. It is designed to trigger automatically on Shopify events, process and map Shopify data, and update or create corresponding records in Odoo. The workflow is modularized into logical blocks for data acquisition, transformation, filtering, and writing to Odoo entities.

Logical blocks include:

- **1.1 Input Reception:** Trigger on Shopify events to receive new or updated data.
- **1.2 Contact Search & Preparation:** Search existing contacts in Odoo and prepare customer data.
- **1.3 Product & Order Data Handling:** Process Shopify order data, split and transform order lines.
- **1.4 Filtering and Conditional Processing:** Apply filters to data for deciding subsequent actions.
- **1.5 Odoo Write Operations:** Create or update Odoo Sale Orders, Sale Order Lines, and related records.
- **1.6 Helper Code Nodes:** Perform data transformations, mappings, and splitting operations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for Shopify events (likely order creation or update) and initiates the workflow by fetching raw Shopify data.

- **Nodes Involved:**  
  - Shopify Trigger  
  - Odoo  
  - Search Odoo Contact  
  - Code6  

- **Node Details:**

  - **Shopify Trigger**  
    - Type: Shopify Trigger (Webhook Listener)  
    - Role: Entry point that triggers the workflow upon Shopify event (e.g., new order)  
    - Configuration: Uses a webhook ID for Shopify to send event data to n8n  
    - Inputs: External webhook call from Shopify  
    - Outputs: Sends raw Shopify data to three parallel nodes (Odoo, Search Odoo Contact, Code6)  
    - Failure Modes: Webhook timeout, invalid payload, Shopify auth errors

  - **Odoo (connected from Shopify Trigger)**  
    - Type: Odoo node  
    - Role: Likely initial data retrieval or setup related to Odoo (details unspecified, but connected immediately after trigger)  
    - Configuration: Uses Odoo credentials to query or prepare data  
    - Inputs: Raw Shopify event data  
    - Outputs: Connects to Sale Order1 node

  - **Search Odoo Contact**  
    - Type: Odoo node  
    - Role: Searches existing contacts in Odoo to find matching customer records based on Shopify data  
    - Configuration: Query configured to find contact by unique identifiers (e.g., email or Shopify customer ID)  
    - Inputs: Shopify event data  
    - Outputs: Passes found contacts to Code4 node  
    - Failure Modes: Odoo connection failure, query errors, no contact found

  - **Code6**  
    - Type: Code node (JavaScript)  
    - Role: Performs initial custom processing or formatting of Shopify data for later use  
    - Inputs: Shopify data  
    - Outputs: Leads into Split Out4 node  
    - Failure Modes: Code errors, expression failures

---

#### 2.2 Contact Search & Preparation

- **Overview:**  
  Using the search results from Odoo, this block prepares the customer/contact data for order creation or update.

- **Nodes Involved:**  
  - Code4  
  - Filter  
  - Odoo5  

- **Node Details:**

  - **Code4**  
    - Type: Code node  
    - Role: Processes search results from Search Odoo Contact to determine if contact exists and prepares data accordingly  
    - Inputs: Output from Search Odoo Contact  
    - Outputs: Routes data through Filter node  
    - Failure Modes: Logic errors in code, unexpected empty inputs

  - **Filter**  
    - Type: Filter node  
    - Role: Applies conditions to decide if contact exists or if a new contact needs to be created  
    - Inputs: Code4 output  
    - Outputs: Calls Odoo5 node if condition passes (e.g., contact creation or update)  
    - Failure Modes: Misconfigured filters leading to incorrect routing

  - **Odoo5**  
    - Type: Odoo node  
    - Role: Creates or updates contact records in Odoo based on Filter decisions  
    - Inputs: Filter node data  
    - Outputs: Connects to next processing step (unspecified)  
    - Failure Modes: Odoo API issues, data validation errors

---

#### 2.3 Product & Order Data Handling

- **Overview:**  
  This block handles the main order data from Shopify, including splitting complex order data into manageable pieces (e.g., order lines) and preparing for insertion into Odoo.

- **Nodes Involved:**  
  - Sale Order1  
  - Code3  
  - Split Out3  
  - Odoo4  
  - Sale Order Line1  

- **Node Details:**

  - **Sale Order1**  
    - Type: Odoo node  
    - Role: Creates or updates the main Sale Order record in Odoo  
    - Inputs: Data from Odoo node connected after Shopify Trigger  
    - Outputs: Passes data to Code3 for processing  
    - Failure Modes: Odoo API errors, invalid order data

  - **Code3**  
    - Type: Code node  
    - Role: Processes Sale Order data, formats or enriches it for order line handling  
    - Inputs: Sale Order1 output  
    - Outputs: Split Out3 node  
    - Failure Modes: Script errors, invalid data structure

  - **Split Out3**  
    - Type: SplitOut node  
    - Role: Splits the order data into individual order lines for separate processing  
    - Inputs: Code3 output  
    - Outputs: Odoo4 node  
    - Failure Modes: Empty or malformed input arrays

  - **Odoo4**  
    - Type: Odoo node  
    - Role: Processes each order line, likely creating or updating Sale Order Line records in Odoo  
    - Inputs: Split Out3 output  
    - Outputs: Sale Order Line1 node  
    - Failure Modes: Odoo errors, data validation issues

  - **Sale Order Line1**  
    - Type: Odoo node  
    - Role: Finalizes insertion or update of Sale Order Line items in Odoo  
    - Inputs: Odoo4 output  
    - Outputs: End of order line processing chain  
    - Failure Modes: API or data errors

---

#### 2.4 Filtering and Conditional Processing

- **Overview:**  
  Filters are used to control workflow branches based on conditions, such as data existence or validity, directing the flow to appropriate Odoo operations.

- **Nodes Involved:**  
  - Filter2  
  - Code5  
  - Odoo6  
  - Odoo7  
  - Split Out4  

- **Node Details:**

  - **Code5**  
    - Type: Code node  
    - Role: Prepares or modifies data before Filter2 decision  
    - Inputs: Odoo6 output  
    - Outputs: Filter2 node  
    - Failure Modes: Code logic errors

  - **Filter2**  
    - Type: Filter node  
    - Role: Checks specific conditions on data to decide if Odoo7 should be executed  
    - Inputs: Code5 output  
    - Outputs: Odoo7 node if condition met  
    - Failure Modes: Misconfiguration causing wrong routing

  - **Odoo6**  
    - Type: Odoo node  
    - Role: Possibly performs intermediate data operations or queries in Odoo  
    - Inputs: Split Out4 output  
    - Outputs: Code5 node  
    - Failure Modes: API failures

  - **Split Out4**  
    - Type: SplitOut node  
    - Role: Splits data array for parallel processing  
    - Inputs: Code6 output (initial processing from Shopify Trigger)  
    - Outputs: Odoo6 node  
    - Failure Modes: Empty or invalid arrays

  - **Odoo7**  
    - Type: Odoo node  
    - Role: Final Odoo operation in this branch, likely updating or confirming records  
    - Inputs: Filter2 output  
    - Outputs: End of this branch  
    - Failure Modes: API or data issues

---

#### 2.5 Helper Code Nodes and Miscellaneous

- **Overview:**  
  These nodes support the workflow by performing custom data manipulation or conditional logic.

- **Nodes Involved:**  
  - Code3, Code4, Code5, Code6 (described above)

- **Details:**  
  Each code node uses JavaScript to process data between Odoo queries and filters, ensuring data is correctly formatted and decisions are based on precise conditions.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                   | Input Node(s)          | Output Node(s)             | Sticky Note |
|---------------------|--------------------|---------------------------------|-----------------------|----------------------------|-------------|
| Shopify Trigger     | Shopify Trigger    | Entry point: listens for Shopify events | External webhook       | Odoo, Search Odoo Contact, Code6 |             |
| Odoo                | Odoo               | Initial Odoo data query/setup   | Shopify Trigger        | Sale Order1                |             |
| Search Odoo Contact | Odoo               | Search contacts in Odoo         | Shopify Trigger        | Code4                      |             |
| Code6               | Code               | Initial Shopify data processing | Shopify Trigger        | Split Out4                 |             |
| Sale Order1         | Odoo               | Create/update Sale Order        | Odoo                   | Code3                      |             |
| Code3               | Code               | Process Sale Order data         | Sale Order1            | Split Out3                 |             |
| Split Out3          | SplitOut           | Split order into order lines    | Code3                  | Odoo4                      |             |
| Odoo4               | Odoo               | Process Sale Order Lines        | Split Out3             | Sale Order Line1           |             |
| Sale Order Line1    | Odoo               | Finalize Sale Order Lines       | Odoo4                  |                            |             |
| Code4               | Code               | Process contact search results  | Search Odoo Contact    | Filter                     |             |
| Filter              | Filter             | Check contact existence         | Code4                  | Odoo5                      |             |
| Odoo5               | Odoo               | Create/update contacts in Odoo  | Filter                  |                            |             |
| Code5               | Code               | Prepare data before Filter2     | Odoo6                  | Filter2                    |             |
| Filter2             | Filter             | Conditional routing             | Code5                  | Odoo7                      |             |
| Odoo6               | Odoo               | Intermediate Odoo query/update | Split Out4             | Code5                      |             |
| Split Out4          | SplitOut           | Split array for processing      | Code6                  | Odoo6                      |             |
| Odoo7               | Odoo               | Final Odoo update in branch     | Filter2                 |                            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger Node:**  
   - Type: Shopify Trigger  
   - Configure webhook to listen for relevant Shopify events (e.g., orders/create)  
   - Connect output to three nodes: Odoo, Search Odoo Contact, Code6  

2. **Create Odoo Node (Initial):**  
   - Type: Odoo  
   - Configure credentials for your Odoo instance  
   - Set operation to retrieve or prepare base data for sale orders  
   - Connect output to Sale Order1 node  

3. **Create Search Odoo Contact Node:**  
   - Type: Odoo  
   - Configure to query contacts by Shopify customer identifiers (e.g., email or external ID)  
   - Connect output to Code4 node  

4. **Create Code6 Node:**  
   - Type: Code  
   - Implement JavaScript to process Shopify data as needed for splitting  
   - Connect output to Split Out4 node  

5. **Create Sale Order1 Node:**  
   - Type: Odoo  
   - Configure to create or update Sale Orders in Odoo using Shopify order data  
   - Connect output to Code3 node  

6. **Create Code3 Node:**  
   - Type: Code  
   - Write script to transform order data for line splitting  
   - Connect output to Split Out3 node  

7. **Create Split Out3 Node:**  
   - Type: SplitOut  
   - Configure to split array of order lines  
   - Connect output to Odoo4 node  

8. **Create Odoo4 Node:**  
   - Type: Odoo  
   - Configure to create or update Sale Order Line records in Odoo  
   - Connect output to Sale Order Line1 node  

9. **Create Sale Order Line1 Node:**  
   - Type: Odoo  
   - Finalize insertion/updating of order lines  
   - No further connections (end of this chain)  

10. **Create Code4 Node:**  
    - Type: Code  
    - Process contact search results to check existence and prepare data  
    - Connect output to Filter node  

11. **Create Filter Node:**  
    - Type: Filter  
    - Condition: Contact exists or not  
    - If true, connect to Odoo5 node  

12. **Create Odoo5 Node:**  
    - Type: Odoo  
    - Create or update contact in Odoo  
    - No further outputs specified  

13. **Create Split Out4 Node:**  
    - Type: SplitOut  
    - Split processed Shopify data for further Odoo processing  
    - Connect output to Odoo6 node  

14. **Create Odoo6 Node:**  
    - Type: Odoo  
    - Perform intermediate data operations or queries  
    - Connect output to Code5 node  

15. **Create Code5 Node:**  
    - Type: Code  
    - Prepare or transform data for filtering  
    - Connect output to Filter2 node  

16. **Create Filter2 Node:**  
    - Type: Filter  
    - Condition based on data for further processing  
    - If true, connect to Odoo7 node  

17. **Create Odoo7 Node:**  
    - Type: Odoo  
    - Finalize updates or confirmations in Odoo  
    - End of workflow branch  

18. **Credentials Setup:**  
    - Shopify Trigger: Configure Shopify app credentials and webhook URL in Shopify Admin  
    - Odoo Nodes: Configure with Odoo instance URL, user credentials, and database name  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The workflow uses multiple Odoo nodes configured to always output data to ensure consistent flow even on empty results. | General workflow stability |
| No sticky note content was provided in the workflow export. | N/A |
| For Shopify webhook setup, ensure the webhook URL is publicly accessible and Shopify app has appropriate permissions for orders and customers. | Shopify developer documentation |
| Odoo API credentials must have rights to read/write contacts, products, and sales orders. | Odoo API docs: https://www.odoo.com/documentation/15.0/developer/api.html |
| Use the SplitOut node carefully to avoid empty arrays which may cause downstream nodes to fail. Add error handling if necessary. | n8n documentation on SplitOut node |

---

_Disclaimer: The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._