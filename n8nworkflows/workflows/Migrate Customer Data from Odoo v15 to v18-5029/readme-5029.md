Migrate Customer Data from Odoo v15 to v18

https://n8nworkflows.xyz/workflows/migrate-customer-data-from-odoo-v15-to-v18-5029


# Migrate Customer Data from Odoo v15 to v18

---

### 1. Workflow Overview

This workflow is designed to migrate customer data from Odoo version 15 to Odoo version 18. It primarily targets businesses or IT teams needing to transfer their customer records when upgrading or switching between these Odoo versions. The workflow is structured into three logical blocks:

- **1.1 Input Trigger:** Manual start of the migration process.
- **1.2 Data Extraction:** Fetching customer data from the source Odoo instance (v15).
- **1.3 Data Migration:** Processing the fetched data in batches and creating corresponding customer records in the destination Odoo instance (v18).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block provides a manual trigger to start the migration workflow on demand.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - **Type:** Manual Trigger  
    - **Role:** Initiates the workflow execution manually.  
    - **Configuration:** No special parameters; standard manual trigger node.  
    - **Input/Output:** No input; outputs trigger the next node "Get Customers from v15".  
    - **Version Requirements:** Compatible with all n8n versions supporting manual triggers.  
    - **Edge Cases:** No typical failure modes; user must manually trigger.  
    - **Sub-workflow:** None.

#### 1.2 Data Extraction

- **Overview:**  
  This block connects to the Odoo v15 instance and retrieves the customer data to migrate.

- **Nodes Involved:**  
  - Get Customers from v15

- **Node Details:**

  - **Get Customers from v15**  
    - **Type:** Odoo Node  
    - **Role:** Queries the Odoo v15 database to fetch customer records.  
    - **Configuration:**  
      - Uses Odoo credentials linked to the v15 instance.  
      - Parameters for the node are default or preconfigured to fetch relevant customer data (e.g., model 'res.partner' or a customer-specific model).  
    - **Key Expressions/Variables:** No expressions; fetches all customers or a predefined selection.  
    - **Input/Output:**  
      - Input from manual trigger node.  
      - Outputs full data set to "SplitInBatches".  
    - **Version Requirements:** Requires valid Odoo v15 API credentials and network accessibility.  
    - **Edge Cases:**  
      - Authentication failure if credentials are invalid or expired.  
      - API rate limits or timeouts on large data sets.  
      - Data schema differences if customized Odoo v15 instance.  
    - **Sub-workflow:** None.

#### 1.3 Data Migration

- **Overview:**  
  This block processes the customer data in manageable batches and creates customer records in the Odoo v18 instance.

- **Nodes Involved:**  
  - SplitInBatches  
  - Create Customers in v18

- **Node Details:**

  - **SplitInBatches**  
    - **Type:** Split In Batches  
    - **Role:** Divides the list of customers into smaller batches to prevent overloading the destination system and to comply with API limits.  
    - **Configuration:** Default batch size (usually 1, 5, 10, or configurable) to control processing granularity.  
    - **Input/Output:**  
      - Input from "Get Customers from v15".  
      - Output to "Create Customers in v18".  
    - **Version Requirements:** Compatible with all modern n8n versions.  
    - **Edge Cases:**  
      - Batch size too large causing timeout or API errors.  
      - Batch size too small causing performance inefficiency.  
    - **Sub-workflow:** None.

  - **Create Customers in v18**  
    - **Type:** Odoo Node  
    - **Role:** Creates customer records in Odoo v18 based on the batch data received.  
    - **Configuration:**  
      - Uses Odoo credentials linked to the v18 instance.  
      - Configured to map and insert fields appropriately to the new system’s schema.  
    - **Key Expressions/Variables:** Mapping expressions or transformations may be applied to align v15 data fields to v18 format.  
    - **Input/Output:**  
      - Input from "SplitInBatches".  
      - Outputs data (optional) or completes the workflow.  
    - **Version Requirements:** Requires valid Odoo v18 API credentials.  
    - **Edge Cases:**  
      - Authentication errors if credentials are invalid.  
      - Data validation errors if field mappings are incorrect or required fields are missing.  
      - API rate limits or timeouts.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                 | Node Type        | Functional Role                    | Input Node(s)           | Output Node(s)           | Sticky Note |
|---------------------------|------------------|----------------------------------|-------------------------|--------------------------|-------------|
| When clicking ‘Test workflow’ | Manual Trigger   | Initiates the workflow manually  |                         | Get Customers from v15    |             |
| Get Customers from v15    | Odoo             | Retrieves customer data from v15 | When clicking ‘Test workflow’ | SplitInBatches           |             |
| SplitInBatches            | Split In Batches | Splits data into manageable batches | Get Customers from v15  | Create Customers in v18   |             |
| Create Customers in v18   | Odoo             | Creates customers in Odoo v18     | SplitInBatches          |                          |             |
| Sticky Note               | Sticky Note      |                                  |                         |                          |             |
| Sticky Note1              | Sticky Note      |                                  |                         |                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking ‘Test workflow’" to start the workflow manually.

2. **Add an Odoo node** named "Get Customers from v15":
   - Set credentials for Odoo v15 instance.
   - Configure the node to query the customer model (commonly 'res.partner' with customer filters).
   - Connect the output of the Manual Trigger node to this node.

3. **Add a SplitInBatches node** named "SplitInBatches":
   - Connect the output of "Get Customers from v15" to this node.
   - Configure batch size according to expected load (e.g., 10).

4. **Add an Odoo node** named "Create Customers in v18":
   - Set credentials for Odoo v18 instance.
   - Configure it to create new customer records matching the v18 schema.
   - Map or transform fields from v15 data as needed (e.g., customer name, email, address).
   - Connect the output of "SplitInBatches" to this node.

5. **Arrange nodes visually** to reflect the flow: Manual Trigger → Get Customers from v15 → SplitInBatches → Create Customers in v18.

6. **Test the workflow** by running the manual trigger and observing successful customer data migration.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                      |
|------------------------------------------------------------------------------|------------------------------------|
| This workflow assumes Odoo instances have standard customer schema and API access. Custom fields or modules may require additional mapping. | Internal project assumption        |
| For detailed Odoo API usage and credentials setup, consult official Odoo documentation: https://www.odoo.com/documentation | Odoo official docs                 |
| Use appropriate API credentials for each Odoo version to avoid authentication errors. | Credential management best practice |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.